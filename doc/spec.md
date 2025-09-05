# Software Specification: Fuzzlock Secrets Vault

## Overview

Fuzzlock is a secrets vault designed for Linux systems. It uses GPG for AES 256 encryption and leverages `fzf` for fuzzy selection. All secrets are stored in dedicated files within the `.secrets` directory in your home directory.

---

## File Structure

Secrets are organized by identifier (e.g., domain) and sub-identifier (e.g., account):

```
.secrets/
├── example.com/
│   ├── user1/
│   │   ├── account.gpg
│   │   ├── generic.gpg
│   │   └── totp.gpg
│   └── user2/
│       ├── account.gpg
│       └── totp.gpg
└── other.com/
```

The identifier and sub-identifier are directories. Typically they're domain and account name, but they can be anything, but they can only be alpanumeric with hyphens.

When a secret file is created, it takes the name of the spec file chosen to create it. For example, if the "account" spec is chosen, the secret file will be named `account.gpg`.

Secret files store simple key-value pairs.

There is a default, built-in spec called "account" that looks like:

```
schema: username,password
flags: a,account
help: Stores a username and password for an account
```

### Example Secret File

A secret file, such as `account.gpg`, looks like:

```
username:...
password:...
```


---

## Specs

A **spec** defines the structure and behavior for specific kinds of secrets. Specs are plain text files stored in `.secrets/.specs` and must contain the following fields (order does not matter):

- **schema**: Comma-separated list of key names (e.g., `username,password`)
- **flags**: Two comma-separated values—one single-character flag (e.g., `a`) and one multi-character flag (e.g., `account`). The single-character value is used as a short flag (`-a`), and the multi-character value as a long flag (`--account`). Either order is allowed; Fuzzlock will detect which is which, and use these values to construct the help menu for the script.
- **help**: A non-empty string describing the spec; this text appears as help in the CLI.

#### Example Spec

```
schema: username,password
flags: a,account
help: Stores user login information for accounts
```

If the order in the flags line is reversed (e.g., `flags: account,a`) Fuzzlock will still correctly assign the short and long flags.

#### Spec Validation

When creating a spec:
- The `schema` field must exist with at least one key name, comma-separated. No comma is necessary if there is only one key.
- The `flags` field must exist with exactly one single-character and one multi-character value. No hyphens are necessary unless it's to join two discrete words.
- The `help` field must exist and be non-empty.

---

## Usage

### Copying Secrets to the Clipboard

- Run `fuzzlock copy` and provide a flag corresponding to the spec you wish to use (e.g., `-a` or `--account` as defined in the spec).
- The user is prompted to select an account via `fzf`. Only the identifier (e.g., domain) and sub-identifier (e.g., account) are displayed, like so:

```
example.com/user1
example.com/user2
other.com/user3
```

- After selection, the corresponding secret file is opened and decrypted using AES256 symmetric encryption via GPG. The user will only enter their GPG password once per session, which leverages built-in GPG caching.

- On successful decryption:
  - For each field defined in the spec’s schema (in the exact order given in the schema), the value is copied to the clipboard one at a time. Any fields not listed in the schema are ignored.
  - For each copied field, Fuzzlock prints a message with the field name from the schema, using an initial uppercase letter (e.g., "Username copied...").
  - The user presses Enter to move to the next field.
  - After all fields have been copied, Fuzzlock prints: "All fields copied!"

All content is trimmed of whitespace and newlines before being copied.

---

### Creating Secrets

1. Run `fuzzlock create`.
2. Select a spec. You may create a new spec by providing a file name:
   - If the spec doesn’t exist, it will be created and opened in `$EDITOR`.
   - Enter exactly three required lines (in any order): `schema`, `flags`, and `help`.
   - The spec is validated on exit. If invalid, you are prompted to confirm and edit again, or cancel (with "N").

3. Next, provide identifying information (such as a domain or label). This is required for fuzzy selection. Blank values or those containing only spaces will be rejected. Values must be alphanumeric, with hyphens, and must be valid for use as a POSIX directory name.

4. For each key defined in the chosen spec’s schema, the script prompts for a value, in the order defined in the schema:
   - Inputs must not be blank or only spaces.
   - Inputs are trimmed of whitespace and newlines.

5. When all values are entered, Fuzzlock creates the necessary secret file in `.secrets`, naming it after the selected spec (e.g., `account.gpg` if the "account" spec was chosen). The script then exits.

---

## Technology

- **Python**
- **GPG** (AES256 symmetric encryption)
- **fzf** (fuzzy selection)

## Testing

- Use **pytest** for testing.

