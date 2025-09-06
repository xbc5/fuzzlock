# Software Specification: Fuzzlock Secrets Vault

## Glossary

**Identifier**: The top-level directory name used to group secrets, typically a domain name. Must be alphanumeric with hyphens only and valid as a POSIX directory name.

**SubIdentifier**: The second-level directory name used to group secrets within an Identifier, typically an account name. Must be alphanumeric with hyphens only and valid as a POSIX directory name.

**FieldName**: The key names used within secret files (e.g., "username", "password"). Must be lowercase, alphanumeric characters only, with hyphens used only to join separate words. No other characters (including underscores) are allowed.

**SpecType**: The name of a spec that defines the structure and behavior of a secret file (e.g., "account", "totp", "backup-keys").

**MasterKeyFile**: A 4096-bit GPG private key exported to ASCII armor format and encrypted with AES256. Stored as `.secrets/.masterkey-[FINGERPRINT].asc` where FINGERPRINT is the GPG key fingerprint. There can only be one MasterKeyFile at any time - operations that change the MasterGpgKey will update both the fingerprint in the filename and the file contents. Committed to git only after successful creation and encryption.

**MasterGpgKey**: The 4096-bit GPG private key stored in the user's GPG keyring, used for encrypting and decrypting all secret files.

**SecretExport**: A tar archive backup of the entire `.secrets` directory, available as either an EncryptedExport (`.tar.gpg`) or PlaintextExport (`.tar`).

**EncryptedExport**: A SecretExport that is encrypted using GPG with AES256 symmetric encryption.

**PlaintextExport**: A SecretExport that is not encrypted.

**CommitMessage**: Automatic commit messages generated from context using the format "[operation]: [SubIdentifier] [SpecType] for [Identifier]" (e.g., "create: user1 account for example.com", "delete: foo account for example.com").

**SpecFile**: A JSON file located at `.secrets/.specs` that defines all available secret types and their schemas. If this file doesn't exist, only the built-in "account" spec is available.

---

## Overview

Fuzzlock is a secrets vault for Linux systems. It uses a GPG 4096-bit private key for encryption and relies on `fzf` for fuzzy selection. All secrets are kept in individual files inside the `.secrets` directory in the user's home directory.

---

## File Structure

Secrets are grouped by Identifier (such as domain) and SubIdentifier (such as account):

```
.secrets/
├── example.com/
│   ├── user1/
│   │   ├── account.gpg
│   │   ├── backup-keys.gpg
│   │   └── totp.gpg
│   └── user2/
│       ├── account.gpg
│       └── totp.gpg
└── other.com/
```

The Identifier and SubIdentifier are directories as defined in the Glossary.

When a secret file is created, it's named after the spec file used to create it. For instance, if the "account" spec is selected, the resulting secret file will be called `account.gpg`.

Secret files contain simple key-value pairs.

There is a built-in default spec named "account" that is always available, even when no SpecFile exists. The built-in "account" spec has the commands "account" and "a", schema `["username", "password"]`, and provides basic login credential storage.

### Example Secret File

A secret file like `account.gpg` has this structure:

```
username:...
password:...
```

---

## SpecFile

The SpecFile (`.secrets/.specs`) is a JSON file that defines all available secret types beyond the built-in "account" spec. If no SpecFile exists, only the built-in "account" spec is available.

## Specs

A **spec** defines the layout and behavior for a specific type of secret file. Custom specs are stored in the SpecFile. Each spec is defined by its name and must include the following fields:

- **schema**: Array of key names (for example, `["username", "password"]`)
- **commands**: Array with exactly two values—one single-character command and one multi-character command (like `["a", "account"]`). These are used as subcommands with `fuzzlock copy`. The order can vary; Fuzzlock will recognize each and use them in the help menu for the script. Multi-character commands use hyphens only to join separate words.
- **help**: A non-empty string describing the spec. This appears as the help text in the CLI.

#### Example Specs File (`.secrets/.specs`):

```json
{
  "totp": {
    "schema": ["secret", "issuer"],
    "commands": ["t", "totp"],
    "help": "Stores TOTP authentication secrets"
  },
  "backup-keys": {
    "schema": ["private-key", "public-key", "passphrase"],
    "commands": ["b", "backup-keys"],
    "help": "Stores backup encryption keys and passphrases"
  }
}
```

The order of commands in the array can be reversed (for example, `["backup-keys", "b"]`), and Fuzzlock will still assign the short and long commands properly.

#### Spec Validation

When creating or editing specs:
- The `schema` field must be present and include at least one key name as an array.
- The `commands` field must be present with exactly one single-character and one multi-character value as an array. Multi-character commands use hyphens only to join separate words.
- The `help` field must be present and not empty.
- All spec names must be valid and unique.
- All commands (both single-character and multi-character) must be unique across all specs.

---

## Usage

### Copying Secrets to the Clipboard

- To copy a secret, run `fuzzlock copy <spec>` using either the single-character or multi-character spec identifier (e.g., `fuzzlock copy account` or `fuzzlock copy a`).
- Optionally, specify a specific field: `fuzzlock copy <spec> <field>` (e.g., `fuzzlock copy account password`).
- The user will be prompted to pick an account through `fzf`. Only the identifier (like domain) and sub-identifier (like account) will be shown in this format:

```
example.com/user1
example.com/user2
other.com/user3
```

- Once the user selects an account, the corresponding secret file is opened and decrypted using the MasterGpgKey. The user's MasterKeyFile password only needs to be entered once per session.

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
2. Pick a spec from available specs (built-in "account" spec plus any defined in the SpecFile). If the SpecFile doesn't exist, only the "account" spec is available.

3. Provide the Identifier and SubIdentifier as defined in the Glossary. All inputs are trimmed of whitespace and newlines before validation. Entries that are blank or only spaces are not accepted.

4. For each key in the chosen spec's schema, the user will be prompted for a value, following the schema's order:
   - All inputs are trimmed of whitespace and newlines before validation.
   - Inputs must not be blank or just spaces after trimming.

5. When all values are entered, Fuzzlock creates the appropriate secret file in `.secrets`, naming it after the chosen spec (like `account.gpg` for the "account" spec). The change is automatically committed to git using the CommitMessage format. The script then ends.

### Handling Existing Secrets

If a secret file already exists for the chosen identifier/sub-identifier/spec combination:

1. Fuzzlock warns the user that the secret already exists and presents three options:
   - **Quit** (default): Cancel the operation
   - **Open in EDITOR**: Open the existing secret file for modification
   - **Proceed and overwrite**: Continue with creating a new secret that will replace the existing one

2. If the user chooses to proceed and overwrite:
   - The user completes entering all new values as normal
   - Before actually overwriting the file, Fuzzlock prompts for final confirmation: `Are you sure you want to overwrite the existing secret? [N/y]`
   - Default response is "N" (no) to prevent accidental data loss
   - Only "y" or "Y" will proceed with the overwrite

---

### Modifying Secrets

- To modify an existing secret, run `fuzzlock modify`.
- The user will be prompted to pick from all existing secret files through `fzf`. The selection will show the full path format:

```
example.com/user1/account.gpg
example.com/user1/totp.gpg
example.com/user2/account.gpg
other.com/user3/backup-keys.gpg
```

- Once the user selects a secret file, it's decrypted using the MasterGpgKey and opened in the user's `$EDITOR`.
- The file will contain the key-value pairs in plain text format (e.g., `username:value`).
- After making changes and saving, the file is automatically re-encrypted using the MasterGpgKey.
- The change is automatically committed to git using the CommitMessage format.
- If the editor is closed without saving or with an empty file, the modification is cancelled and the original secret remains unchanged.

---

### Deleting Secrets

- To delete an existing secret, run `fuzzlock delete`.
- The user will be prompted to pick from all existing secret files through `fzf`. The selection will show the full path format:

```
example.com/user1/account.gpg
example.com/user1/totp.gpg
example.com/user2/account.gpg
other.com/user3/backup-keys.gpg
```

- Once the user selects a secret file, Fuzzlock prompts for confirmation: `Are you sure you want to delete this secret? [N/y]`
- Default response is "N" (no) to prevent accidental deletion.
- Only "y" or "Y" will proceed with the deletion.
- After deletion, the change is automatically committed to git using the CommitMessage format.

---

### Editing Specs

- Use `fuzzlock spec edit` to modify the SpecFile.
- The SpecFile (`.secrets/.specs`) is opened in the user's `$EDITOR`.
- After saving, the SpecFile is validated for correctness as described in the SpecFile Validation section.
- If validation fails, the user is prompted to edit again or cancel the operation.

---

## SpecFile Validation

SpecFile validation occurs in two places to ensure command uniqueness and proper formatting:

1. **After editing the SpecFile** through `fuzzlock spec edit`:
   - The SpecFile is validated immediately after saving
   - If validation fails, the user is prompted to edit again or cancel the operation

2. **Before specs are used in operations**:
   - All commands defined in the SpecFile are validated to be unique
   - Both single-character and multi-character commands must be unique across all specs, including the built-in "account" spec
   - If duplicate commands are found, the script prints all conflicting commands and exits
   - No operations are performed if command conflicts exist

### Validation Rules

When creating or editing specs:
- The `schema` field must be present and include at least one key name as an array
- The `commands` field must be present with exactly one single-character and one multi-character value as an array. Multi-character commands use hyphens only to join separate words
- The `help` field must be present and not empty
- All spec names must be valid and unique
- All commands (both single-character and multi-character) must be unique across all specs

---

## Backup and Version Control

Fuzzlock automatically maintains a git repository in the `.secrets` directory to track all changes to secret files.

### Automatic Git Commits

- After every secret file operation (create, modify, delete), Fuzzlock automatically commits the changes to git using the CommitMessage format defined in the Glossary.
- All secret files are encrypted with the MasterGpgKey and stored in ASCII armor format to ensure git compatibility.

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

- Fuzzlock uses a single MasterGpgKey (as defined in the Glossary) for all encryption operations to ensure password synchronization.
- The MasterGpgKey is a 4096-bit GPG private key stored in the user's GPG keyring.
- The encrypted MasterKeyFile is stored as `.secrets/.masterkey-[FINGERPRINT].asc` and contains an ASCII armor export of the MasterGpgKey encrypted with AES256 using a user-provided password.
- When unlocking or encrypting, the script checks if the FINGERPRINT exists in the GPG keyring. If not, it decrypts the MasterKeyFile and imports the key into the GPG keyring.
- The MasterKeyFile is version controlled along with all secret files.
- All secret files are encrypted using the MasterGpgKey (asymmetric encryption). The MasterKeyFile serves as a backup of the GPG key.

### Encryption Abstraction

All encryption operations are abstracted behind a single encryption class that MUST be general enough to support different encryption methods, including future post-quantum cryptography. The abstraction hides all implementation details while maintaining the only consistent user experience requirement:

- User provides a single password per session to unlock their MasterKeyFile
- That password unlocks the encryption key used to encrypt/decrypt all secret files
- The user experience remains the same regardless of the underlying encryption method

This design allows Fuzzlock to migrate away from GPG to other encryption methods without changing the user experience or core application logic. Future encryption backends may use different underlying mechanisms (derived keys, certificates, quantum-resistant algorithms, etc.) but the user will always provide one password per session that secures access to all their secrets.

### Master Key Initialization

- On first use, if no MasterKeyFile exists, Fuzzlock prompts the user to create a MasterGpgKey.
- The user provides a password that will be used to encrypt the MasterKeyFile.
- A 4096-bit GPG private key is generated (MasterGpgKey), exported to ASCII armor format, encrypted with AES256, and stored as the MasterKeyFile at `.secrets/.masterkey-[FINGERPRINT].asc`.
- The MasterKeyFile is committed to git only after successful creation and encryption.

### Session Management

- At the start of each session, users must provide their MasterKeyFile password to unlock the MasterGpgKey.
- The MasterGpgKey remains unlocked in memory for the duration of the session.
- All secret files are encrypted/decrypted using the MasterGpgKey.

### Password Recovery

- There is no recovery mechanism for forgotten MasterKeyFile passwords.
- Users are responsible for backing up their entire `.secrets` directory.
- Forgetting the MasterKeyFile password means permanent loss of access to all secrets.

---

### Exporting Secrets

- Use `fuzzlock export [path]` to create a backup of the entire `.secrets` directory.
- The export creates a tar archive containing all secrets and configuration files.
- The user is prompted whether to encrypt the export: `Encrypt export? [Y/n]` (default is Yes).
- If encrypted, the export uses GPG with AES256 symmetric encryption with a user-provided password.

#### File Naming and Paths:

- **No path provided**: Creates `secrets.tar` or `secrets.tar.gpg` in the current directory
- **Directory path provided**: Creates the archive in that directory with the default name
- **File path provided**: Uses the provided path as the base name, discarding any extension (everything after the first period)
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
- Any user-provided file extensions are discarded (everything after the first period in the filename is ignored).
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

- Use `fuzzlock change-password` to update the MasterKeyFile encryption password.
- The user is prompted for the current MasterKeyFile password.
- The user is then prompted to enter a new password.
- The MasterKeyFile is re-encrypted with the new password and saved. The fingerprint remains unchanged since only the encryption password is modified, not the underlying MasterGpgKey.
- The change is committed to git only after the MasterKeyFile has been successfully changed and encrypted.

---

### Listing Secrets

- Use `fuzzlock list` to view all existing secrets in a directory tree format.
- The output displays only secret files and the MasterKeyFile, using colors when possible.
- Shows the complete structure including identifiers, sub-identifiers, and secret files.

#### Example Output:
```
.secrets/
├── .masterkey-[FINGERPRINT].asc
├── example.com/
│   ├── user1/
│   │   ├── account.gpg
│   │   ├── backup-keys.gpg
│   │   └── totp.gpg
│   └── user2/
│       ├── account.gpg
│       └── totp.gpg
└── other.com/
    └── user3/
        └── backup-keys.gpg
```

---

## Command Line Interface

### Default Action

- Running `fuzzlock` without any arguments executes `fuzzlock copy account` (copy account secrets using the built-in "account" spec).

### Help System

- The help system is provided by Python's argparse module.
- Use `fuzzlock --help` or `fuzzlock -h` to display available commands and usage information.

### Error Handling

- If `fzf` selection is cancelled by the user, the script exits silently.
- For other errors (GPG failures, file permission issues, invalid input validation, etc.), appropriate error messages are displayed for the user to handle.

---

## Dependency Management

- At startup, Fuzzlock evaluates all required dependencies.
- If any dependencies are missing, it prints all missing dependencies and exits.
- No operations are performed if dependencies are not satisfied.

## Implementation

- The entire script must be contained in a single Python file.
- The script must be highly portable with no external runtime dependencies other than `gpg` and `fzf`.
- External development dependencies (such as pytest, linting tools) are acceptable but not required for script execution.

## Code Quality Requirements

- The code must be written using Python best practices.
- The code must be highly testable with proper separation of concerns.
- All code must be linted and formatted according to Python standards.
- The code must pass all linting checks and formatting requirements.

## Testing Requirements

- There must be an extensive test suite covering all functionality.
- The test suite must include unit tests for all major components.
- All tests must pass before the code can be considered complete.
- The code must comply with user acceptance tests.

## Technology

- **Python** (standard library only, no external runtime dependencies)
- **GPG** (for MasterGpgKey generation and AES256 encryption)
- **fzf** (fuzzy selection with fixed vertical size displaying ~10 results)

## Testing

- Use **pytest** to run tests.
