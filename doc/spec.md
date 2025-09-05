# Software Specification: Fuzzlock Secrets Vault

## Overview

Fuzzlock is a secrets vault designed for Linux systems. It utilizes the `pass` command and `fzf` for fuzzy selection of secrets. All secrets are stored in the `.secrets` directory within your home directory. 

## File Structure

Secrets are organized as follows:

```
.secrets/
├── example.com/
│   ├── user1
│   │   ├── account.gpg
│   │   ├── generic.gpg
│   │   └── totp.gpg
│   └── user2
│       ├── account.gpg
│       └── totp.gpg
└── other.com/
```

File names are arbitrary and each file is a simple key-value store.

### Account File Schema

An example schema for an account file, which stores a username and password, is shown below:

```
# account
username:...
password:...
```

Schemas are defined as comma-separated lists of key names and are stored in `.secrets/.schemas`. For example:

```
username,password
```

The `account` schema is built-in, but users may define their own schemas.

---

## Usage

### Fetching Secrets

- Start by running `fuzzlock`.
- Supply a flag to indicate what to fetch:
  - `-a` for accounts

The application will prompt the user to select an account via `fzf`. Only the identifier (such as the domain) and the sub-identifier (such as the account) are displayed. Available entries will look similar to:

```
example.com/user1
example.com/user2
other.com/user3
```

After a selection, the application opens the corresponding file, prompting the user for a password to decrypt it. Decryption uses AES256 GPG symmetric encryption and requires a password only once per session, leveraging built-in GPG caching.

On successful decryption:

- The content is copied to the clipboard in a multi-step process:
  - Each field specified in the schema is copied to the clipboard one at a time, following the exact order defined in the schema—regardless of the order in the actual secrets file. All other fields in the secrets file are ignored.
  - For each field, the application prints a notification in the format "<KeyName> copied...", where the field name is taken directly from the key name in the schema, with the first letter capitalized (for example, "Username copied...").
  - The user must press Enter to proceed to the next field.
  - When all schema-defined fields are copied, it prints: "All fields copied!"

Whitespace and newlines are trimmed from all content before copying.

---

### Creating Secrets

To create a new secret:

1. Run `fuzzlock create`.
2. The script prompts you to select a schema file.
3. You may create a new schema by typing a filename:
   - If the file doesn't exist, it will be created and opened in `$EDITOR`.
   - You must enter a single line of comma-separated values (alphanumeric, spaces, and hyphens only).
   - Upon exit, the input is validated. If invalid, a message appears, and you are prompted to confirm. If you agree, the file reopens for editing. This repeats until a valid schema is defined, or you select "N" to cancel.

4. Next, the script requests identifying information (e.g., domain name). This is mandatory and used for fuzzy selection. If blank or spaces, you will be reprompted.

5. For each key in the schema (comma-separated), the script prompts for a value:
   - Inputs cannot be blank or consist only of spaces.
   - Inputs are trimmed of whitespace and newlines.

6. Once all values are entered, the script creates the necessary files in the `.secrets` directory, naming each file according to the selected schema. For example, if you choose the "account" schema, an `account.gpg` file is created. The script then exits.

---

## Technology

- **Python**
- **GPG** (AES256 symmetric encryption)
- **fzf** (fuzzy selection)

## Testing

- Use **pytest** for testing.

---
