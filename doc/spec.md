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

There is a built-in default spec named "account" that is automatically created if no `.specs` file exists.

### Example Secret File

A secret file like `account.gpg` has this structure:

```
username:...
password:...
```

---

## Specs

A **spec** defines the layout and behavior for a specific type of secret file. All specs are stored in a single JSON file at `.secrets/.specs`. Each spec is defined by its name and must include the following fields:

- **schema**: Array of key names (for example, `["username", "password"]`)
- **commands**: Array with exactly two values—one single-character command and one multi-character command (like `["a", "account"]`). These are used as subcommands with `fuzzlock copy`. The order can vary; Fuzzlock will recognize each and use them in the help menu for the script.
- **help**: A non-empty string describing the spec. This appears as the help text in the CLI.

#### Example Specs File (`.secrets/.specs`):

```json
{
  "account": {
    "schema": ["username", "password"],
    "commands": ["a", "account"],
    "help": "Stores user login information for accounts"
  },
  "totp": {
    "schema": ["secret", "issuer"],
    "commands": ["t", "totp"],
    "help": "Stores TOTP authentication secrets"
  },
  "generic": {
    "schema": ["value"],
    "commands": ["g", "generic"],
    "help": "Stores a single generic secret value"
  }
}
```

The order of commands in the array can be reversed (for example, `["account", "a"]`), and Fuzzlock will still assign the short and long commands properly.

#### Spec Validation

When creating or editing specs:
- The `schema` field must be present and include at least one key name as an array.
- The `commands` field must be present with exactly one single-character and one multi-character value as an array. Hyphens are required only to join separate words in multi-character commands.
- The `help` field must be present and not empty.
- All spec names must be valid and unique.
- All commands (both single-character and multi-character) must be unique across all specs.

---

## Usage

### Copying Secrets to the Clipboard

- To copy a secret, run `fuzzlock copy <spec>` using either the single-character or multi-character spec identifier (e.g., `fuzzlock copy account` or `fuzzlock copy a`).
- Optionally, specify a specific field: `fuzzlock copy <spec> <field>` (e.g., `fuzzlock copy account password`).
- You'll be prompted to pick an account through `fzf`. Only the identifier (like domain) and sub-identifier (like account) will be shown in this format:

```
example.com/user1
example.com/user2
other.com/user3
```

- Once you select an account, the corresponding secret file is opened and decrypted using AES256 symmetric encryption via GPG. Your GPG password only needs to be entered once per session, thanks to GPG's built-in caching.

#### Copying All Fields (no field specified):
- For each field in the spec's schema (in the listed order), the value is copied to the clipboard one at a time. Any fields not found in the schema are skipped.
- For each copied field, Fuzzlock displays a message using the field name from the schema, capitalized (e.g., "Username copied...").
- Press Enter to copy the next field.
- When all fields are copied, Fuzzlock prints: "All fields copied!"

#### Copying Specific Field:
- The script validates that the specified field exists in the chosen spec's schema.
- If valid, only that field's value is copied to the clipboard.
- The script displays a message (e.g., "Password copied...") and exits.
- If the field is not found in the schema, an error message is displayed.

All content is trimmed of whitespace and newlines before copying.

---

### Creating Secrets

1. Run `fuzzlock create`.
2. Pick a spec. You can also create a new spec by entering a spec name:
   - If the spec does not exist, you'll be prompted to create it.
   - The `.specs` JSON file will be opened in `$EDITOR` for you to add the new spec.
   - After saving, the specs file is checked for validity. If invalid, you'll be asked to confirm and edit again, or cancel by pressing "N".

3. Provide identifying information (like a domain or label). This is needed for fuzzy selection. Entries that are blank or only spaces are not accepted. The value must be alphanumeric with hyphens and suitable as a POSIX directory name.

4. For each key in the chosen spec’s schema, you’ll be prompted for a value, following the schema's order:
   - Inputs must not be blank or just spaces.
   - Inputs are trimmed of whitespace and newlines.

5. When all values are entered, Fuzzlock creates the appropriate secret file in `.secrets`, naming it after the chosen spec (like `account.gpg` for the "account" spec). The script then ends.

### Handling Existing Secrets

If a secret file already exists for the chosen identifier/sub-identifier/spec combination:

1. Fuzzlock warns the user that the secret already exists and presents three options:
   - **Quit** (default): Cancel the operation
   - **Open in EDITOR**: Open the existing secret file for modification
   - **Proceed and overwrite**: Continue with creating a new secret that will replace the existing one

2. If the user chooses to proceed and overwrite:
   - They complete entering all new values as normal
   - Before actually overwriting the file, Fuzzlock prompts for final confirmation: `Are you sure you want to overwrite the existing secret? [N/y]`
   - Default response is "N" (no) to prevent accidental data loss
   - Only "y" or "Y" will proceed with the overwrite

---

### Modifying Secrets

- To modify an existing secret, run `fuzzlock modify`.
- You'll be prompted to pick from all existing secret files through `fzf`. The selection will show the full path format:

```
example.com/user1/account.gpg
example.com/user1/totp.gpg
example.com/user2/account.gpg
other.com/user3/generic.gpg
```

- Once you select a secret file, it's decrypted and opened in your `$EDITOR`.
- The file will contain the key-value pairs in plain text format (e.g., `username:value`).
- After making changes and saving, the file is automatically re-encrypted using GPG AES256 encryption.
- If the editor is closed without saving or with an empty file, the modification is cancelled and the original secret remains unchanged.

---

### Deleting Secrets

- To delete an existing secret, run `fuzzlock delete`.
- You'll be prompted to pick from all existing secret files through `fzf`. The selection will show the full path format:

```
example.com/user1/account.gpg
example.com/user1/totp.gpg
example.com/user2/account.gpg
other.com/user3/generic.gpg
```

- Once you select a secret file, Fuzzlock prompts for confirmation: `Are you sure you want to delete this secret? [N/y]`
- Default response is "N" (no) to prevent accidental deletion.
- Only "y" or "Y" will proceed with the deletion.
- After deletion, the change is automatically committed to git with the appropriate commit message format.

---

## Backup and Version Control

Fuzzlock automatically maintains a git repository in the `.secrets` directory to track all changes to secret files.

### Automatic Git Commits

- After every secret file operation (create, modify, delete), Fuzzlock automatically commits the changes to git.
- All secret files use GPG ASCII armor format (base64 encoding) to ensure git compatibility.
- Commit messages follow a contextual format based on the operation:
  - **Create**: `create: <sub-identifier> <schema-type> on <identifier>`
  - **Update**: `update: <sub-identifier> <schema-type> on <identifier>`
  - **Delete**: `delete: <sub-identifier> <schema-type> on <identifier>`

#### Examples:
```
create: user1 account on example.com
update: admin totp on github.com
delete: old-user generic on company.org
```

The schema type is derived from the spec name used (e.g., "account", "totp", "generic").

### Repository Initialization

- If `.secrets` is not already a git repository, Fuzzlock initializes it automatically on first use.
- A `.gitignore` file is created to exclude temporary files and editor backups.

### Repository State Validation

- Before any operation, Fuzzlock checks for staged but uncommitted changes in the git repository.
- If staged changes are detected, this indicates potential silent corruption or out-of-band modifications.
- The user must resolve these changes before proceeding with any Fuzzlock operations.

### Repository Reset

- Use `fuzzlock reset` to clean the repository state when staged changes are detected.
- This command resets all changes to HEAD and removes untracked files.
- The user is prompted with a warning: `This will reset all changes and remove untracked files. Continue? [N/y]`
- Default response is "N" (no) to prevent accidental data loss.

---

## Encryption Architecture

### Master Key Management

- Fuzzlock uses a single master key for all encryption operations to ensure password synchronization.
- The master key is a 4096-bit GPG private key, encrypted with a user-provided password.
- The master key is stored in `.secrets/.masterkey` as ASCII armor format.
- The master key file is version controlled along with all secret files.

### Encryption Abstraction

All encryption operations are abstracted behind a single encryption class that MUST be general enough to support different encryption methods, including future post-quantum cryptography. The abstraction hides all implementation details while maintaining the only consistent user experience requirement:

- User provides a single password per session
- That password applies to encrypt/decrypt all secret files

This design allows Fuzzlock to migrate away from GPG to other encryption methods without changing the user experience or core application logic. Future encryption backends may use different underlying mechanisms (derived keys, certificates, quantum-resistant algorithms, etc.) but the user will always provide one password that secures all their secrets.

### Master Key Initialization

- On first use, if no master key exists, Fuzzlock prompts the user to create one.
- The user provides a password that will be used to encrypt the master key.
- A 4096-bit GPG private key is generated and stored in `.secrets/.masterkey`.
- Upon master key creation, it is automatically committed to git.

### Session Management

- At the start of each session, users must provide their master key password.
- The master key remains unlocked in memory for the duration of the session.
- All secret files are encrypted/decrypted using this master key.

### Password Recovery

- There is no recovery mechanism for forgotten master key passwords.
- Users are responsible for backing up their entire `.secrets` directory.
- Forgetting the master key password means permanent loss of access to all secrets.

---

### Exporting Secrets

- Use `fuzzlock export [path]` to create a backup of the entire `.secrets` directory.
- The export creates a tar archive containing all secrets and configuration files.
- The user is prompted whether to encrypt the export: `Encrypt export? [Y/n]` (default is Yes).
- If encrypted, the export uses GPG AES256 symmetric encryption with a user-provided password.

#### File Naming and Paths:

- **No path provided**: Creates `secrets.tar` or `secrets.tar.gpg` in the current directory
- **Directory path provided**: Creates the archive in that directory with the default name
- **File path provided**: Uses the provided path as the base name, ignoring any extension
- **Non-existent parent directories**: The final component becomes the filename

#### Examples:
```
fuzzlock export                    # Creates ./secrets.tar(.gpg)
fuzzlock export /backup/           # Creates /backup/secrets.tar(.gpg)
fuzzlock export /backup/mydata     # Creates /backup/mydata.tar(.gpg)
fuzzlock export /backup/mydata.zip # Creates /backup/mydata.tar(.gpg) (ignores .zip)
fuzzlock export /exists/noexist    # Creates /exists/noexist.tar(.gpg)
```

- The script automatically appends `.tar` for unencrypted exports or `.tar.gpg` for encrypted exports.
- Any user-provided file extensions are ignored.
- If the target archive already exists, the user is prompted: `Archive already exists. Overwrite? [N/y]` (default is "N" to prevent accidental overwriting).

---

### Importing Secrets

- Use `fuzzlock import <archive_path>` to restore from an exported archive.
- Import only works if `~/.secrets` does not exist.
- If `~/.secrets` already exists, the command fails with an error message.
- The archive is always extracted to `~/.secrets` regardless of where the command is executed.
- The script determines file type using MIME type detection, not file extensions.
- If the archive is encrypted, the user will be prompted for the decryption password.
- The git repository is restored as part of the archive (no initialization needed).

---

### Changing Master Key Password

- Use `fuzzlock change-password` to update the master key encryption password.
- The user is prompted for their current master key password.
- The user is then prompted to enter a new password.
- The master key is re-encrypted with the new password and saved.
- The change is committed to git only after the master key has been successfully changed and encrypted.

---

### Listing Secrets

- Use `fuzzlock list` to view all existing secrets in a directory tree format.
- The output displays only secret files and the master key file, using colors when possible.
- Shows the complete structure including identifiers, sub-identifiers, and secret files.

#### Example Output:
```
.secrets/
├── .masterkey
├── example.com/
│   ├── user1/
│   │   ├── account.gpg
│   │   ├── generic.gpg
│   │   └── totp.gpg
│   └── user2/
│       ├── account.gpg
│       └── totp.gpg
└── other.com/
    └── user3/
        └── generic.gpg
```

---

## Command Line Interface

### Default Action

- Running `fuzzlock` without any arguments executes `fuzzlock copy account` (copy account secrets).

### Help System

- The help system is provided by Python's argparse module.
- Use `fuzzlock --help` or `fuzzlock -h` to display available commands and usage information.

### Error Handling

- If `fzf` selection is cancelled by the user, the script exits silently.

---

## Dependency Management

- At startup, Fuzzlock evaluates all required dependencies.
- If any dependencies are missing, it prints all missing dependencies and exits.
- No operations are performed if dependencies are not satisfied.

## Spec Command Validation

- At startup, Fuzzlock validates that all commands defined in spec files are unique.
- Both single-character and multi-character commands must be unique across all specs.
- If duplicate commands are found, the script prints all conflicting commands and exits.
- No operations are performed if command conflicts exist.

## Implementation

- The entire script must be contained in a single Python file.

## Technology

- **Python**
- **GPG** (for master key generation and AES256 encryption)
- **fzf** (fuzzy selection)

## Testing

- Use **pytest** to run tests.
