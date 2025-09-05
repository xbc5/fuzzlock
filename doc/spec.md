# Software Specification: Fuzzlock Secrets Vault

## Overview

Fuzzlock is a secrets vault for Linux systems. It uses GPG for AES 256 encryption and relies on `fzf` for fuzzy selection. All secrets are kept in individual files inside the `.secrets` directory in your home directory.

---

## File Structure

Secrets are grouped by identifier (such as domain) and sub-identifier (such as account):

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

The identifier and sub-identifier are directories. Typically, they're domain names and account names, but the value MUST be alphanumeric or include hyphens. Both must also be valid POSIX directory names.

When a secret file is created, it's named after the spec file used to create it. For instance, if the "account" spec is selected, the resulting secret file will be called `account.gpg`.

Secret files contain simple key-value pairs.

There is a built-in default spec named "account" with the following format:

```
schema: username,password
flags: a,account
help: Stores a username and password for an account
```

### Example Secret File

A secret file like `account.gpg` has this structure:

```
username:...
password:...
```

---

## Specs

A **spec** defines the layout and behavior for a specific type of secret file. Specs are plain text files stored in `.secrets/.specs`; each must include the following fields (order does not matter):

- **schema**: Comma-separated list of key names (for example, `username,password`)
- **flags**: Two comma-separated values—one single-character flag (like `a`) and one multi-character flag (like `account`). The single-character flag is used with `-a`, and the multi-character flag with `--account`. The order can vary; Fuzzlock will recognize each and use them in the help menu for the script.
- **help**: A non-empty string describing the spec. This appears as the help text in the CLI.

#### Example Spec

```
schema: username,password
flags: a,account
help: Stores user login information for accounts
```

If the order of the flags is reversed (for example, `flags: account,a`), Fuzzlock still assigns the short and long flags properly.

#### Spec Validation

When creating a spec:
- The `schema` field must be present and include at least one key name, separated by commas. If there is only one key, a comma is not required.
- The `flags` field must be present with exactly one single-character and one multi-character value. Hyphens are required only to join separate words.
- The `help` field must be present and not empty.

---

## Usage

### Copying Secrets to the Clipboard

- To copy a secret, run `fuzzlock copy` and use the desired flag for the spec (like `-a` or `--account`, as listed in the spec).
- You’ll be prompted to pick an account through `fzf`. Only the identifier (like domain) and sub-identifier (like account) will be shown in this format:

```
example.com/user1
example.com/user2
other.com/user3
```

- Once you select an account, the corresponding secret file is opened and decrypted using AES256 symmetric encryption via GPG. Your GPG password only needs to be entered once per session, thanks to GPG’s built-in caching.

- After decryption:
  - For each field in the spec's schema (in the listed order), the value is copied to the clipboard one at a time. Any fields not found in the schema are skipped.
  - For each copied field, Fuzzlock displays a message using the field name from the schema, capitalized (e.g., "Username copied...").
  - Press Enter to copy the next field.
  - When all fields are copied, Fuzzlock prints: "All fields copied!"

All content is trimmed of whitespace and newlines before copying.

---

### Creating Secrets

1. Run `fuzzlock create`.
2. Pick a spec. You can also create a new spec by entering a filename:
   - If the spec does not exist, it will be created and opened in `$EDITOR`.
   - Enter the three required lines (`schema`, `flags`, and `help`), in any order.
   - After saving, the spec is checked for validity. If invalid, you’ll be asked to confirm and edit again, or cancel by pressing "N".

3. Provide identifying information (like a domain or label). This is needed for fuzzy selection. Entries that are blank or only spaces are not accepted. The value must be alphanumeric with hyphens and suitable as a POSIX directory name.

4. For each key in the chosen spec’s schema, you’ll be prompted for a value, following the schema's order:
   - Inputs must not be blank or just spaces.
   - Inputs are trimmed of whitespace and newlines.

5. When all values are entered, Fuzzlock creates the appropriate secret file in `.secrets`, naming it after the chosen spec (like `account.gpg` for the "account" spec). The script then ends.

---

## Technology

- **Python**
- **GPG** (AES256 symmetric encryption)
- **fzf** (fuzzy selection)

## Testing

- Use **pytest** to run tests.
