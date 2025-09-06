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
- **Git** (for automatic git commits)
- **GPG** (for MasterGpgKey generation and AES256 encryption)
- **fzf** (fuzzy selection with fixed vertical size displaying ~10 results)

## Testing

- Use **pytest** to run tests.

---

# User Acceptance Tests

## 1. Installation and Dependencies

### 1.1 Dependency Validation
- **Given** the system is missing required dependencies
- **When** fuzzlock is executed
- **Then** all missing dependencies are printed and the script exits
- **And** no operations are performed

### 1.2 Required Dependencies Present
- **Given** all required dependencies (Python, git, GPG, fzf) are installed
- **When** fuzzlock is executed
- **Then** the script runs without dependency errors

### 1.3 Dependency Check Timing
- **Given** fuzzlock is executed with any command
- **When** the script starts
- **Then** dependency evaluation occurs immediately at startup
- **And** this happens BEFORE any argument parsing or operation logic
- **And** no operations are attempted if dependencies are missing

### 1.4 Partial Dependency Availability
- **Given** some but not all dependencies are available
- **When** fuzzlock is executed
- **Then** ALL missing dependencies are identified and printed
- **And** the script exits without attempting any operations
- **And** no partial functionality is provided

## 2. File Structure and Directory Management

### 2.1 Secrets Directory Creation
- **Given** no `.secrets` directory exists in the user's home directory
- **When** any fuzzlock operation is attempted
- **Then** the `.secrets` directory is created automatically
- **And** a git repository is initialized in `.secrets`
- **And** a `.gitignore` file is created

### 2.2 Directory Structure Validation
- **Given** secrets are created for multiple identifiers and sub-identifiers
- **When** the file structure is examined
- **Then** the structure follows the pattern: `.secrets/[identifier]/[sub-identifier]/[spec].gpg`
- **And** all directory names are valid POSIX directory names
- **And** identifiers and sub-identifiers contain only alphanumeric characters and hyphens

## 3. Master Key Management

### 3.1 Initial Master Key Creation
- **Given** no MasterKeyFile exists
- **When** fuzzlock is run for the first time
- **Then** the user is prompted to create a MasterGpgKey
- **And** the user provides a password for encrypting the MasterKeyFile
- **And** a 4096-bit GPG private key is generated
- **And** the key is exported to ASCII armor format
- **And** the MasterKeyFile is encrypted with AES256 and stored as `.secrets/.masterkey-[FINGERPRINT].asc`
- **And** the MasterKeyFile is committed to git

### 3.2 Master Key Loading
- **Given** a MasterKeyFile exists but the key is not in the GPG keyring
- **When** an operation requiring decryption is attempted
- **Then** the MasterKeyFile is decrypted using the user's password
- **And** the key is imported into the GPG keyring
- **And** the operation proceeds with the imported key

### 3.3 Session Management
- **Given** a user provides their MasterKeyFile password
- **When** multiple operations are performed in the same session
- **Then** the password is only requested once per session
- **And** the MasterGpgKey remains unlocked for the duration

### 3.4 Password Change
- **Given** a user wants to change their MasterKeyFile password
- **When** `fuzzlock change-password` is executed
- **Then** the user is prompted for the current password
- **And** the user is prompted for a new password
- **And** the MasterKeyFile is re-encrypted with the new password
- **And** the fingerprint remains unchanged
- **And** the change is committed to git

## 4. Spec Management

### 4.1 Built-in Account Spec
- **Given** no SpecFile exists
- **When** fuzzlock operations are performed
- **Then** the built-in "account" spec is available
- **And** the account spec has commands "account" and "a"
- **And** the account spec has schema ["username", "password"]

### 4.2 Custom Spec Creation
- **Given** a user wants to create custom specs
- **When** `fuzzlock spec edit` is executed
- **Then** the SpecFile `.secrets/.specs` is opened in the user's editor
- **And** custom specs can be defined with schema, commands, and help fields

### 4.3 Spec Validation
- **Given** a SpecFile is edited
- **When** the file is saved
- **Then** validation occurs immediately
- **And** if validation fails, the user is prompted to edit again or cancel
- **And** all commands must be unique across all specs
- **And** each spec must have exactly one single-character and one multi-character command
- **And** the schema field must contain at least one key name
- **And** the help field must be non-empty

### 4.4 Command Uniqueness Validation
- **Given** specs with duplicate commands exist
- **When** any fuzzlock operation is attempted
- **Then** all conflicting commands are printed
- **And** the script exits without performing operations

### 4.5 Spec Name Validation
- **Given** a SpecFile contains invalid spec names
- **When** the SpecFile is validated
- **Then** validation fails for non-unique spec names
- **And** validation fails for invalid spec name formats

### 4.6 Command Order Handling
- **Given** a spec has commands in different orders (e.g., ["backup-keys", "b"] vs ["b", "backup-keys"])
- **When** the spec is processed
- **Then** Fuzzlock correctly assigns short and long commands regardless of order
- **And** the help system displays commands consistently

### 4.7 Multi-character Command Validation
- **Given** a multi-character command contains invalid characters
- **When** spec validation occurs
- **Then** validation fails for commands with characters other than alphanumeric and hyphens
- **And** validation fails for commands with hyphens not joining separate words

### 4.8 Schema Field Name Validation
- **Given** a spec schema contains invalid field names
- **When** spec validation occurs
- **Then** validation fails for field names with uppercase characters
- **And** validation fails for field names with underscores
- **And** validation fails for field names with invalid hyphen usage
- **And** validation fails for field names starting or ending with hyphens
- **And** validation fails for field names with consecutive hyphens
- **And** validation fails for empty field names

### 4.9 Built-in Account Spec Command Conflicts
- **Given** a custom spec attempts to use commands "a" or "account"
- **When** SpecFile validation occurs
- **Then** validation fails due to conflict with built-in account spec
- **And** error message specifically mentions conflict with built-in spec

### 4.10 No SpecFile Available State
- **Given** no SpecFile exists at `.secrets/.specs`
- **When** any operation requiring spec selection occurs
- **Then** only the built-in "account" spec is available for selection
- **And** no custom specs are presented as options
- **And** this state is clearly indicated to the user

## 5. Secret Creation

### 5.1 Basic Secret Creation
- **Given** a user runs `fuzzlock create`
- **When** prompted for spec selection
- **Then** all available specs are displayed (built-in account plus custom specs)
- **And** after selecting a spec, user is prompted for Identifier
- **And** after providing Identifier, user is prompted for SubIdentifier
- **And** for each field in the spec's schema, user is prompted for a value
- **And** all inputs are trimmed of whitespace and newlines
- **And** blank or whitespace-only inputs are rejected

### 5.2 Secret File Creation
- **Given** all required values are provided for a new secret
- **When** the creation process completes
- **Then** a secret file is created at `.secrets/[identifier]/[sub-identifier]/[spec].gpg`
- **And** the file is encrypted using the MasterGpgKey
- **And** the change is committed to git with format "[operation]: [SubIdentifier] [SpecType] for [Identifier]"

### 5.3 Existing Secret Handling
- **Given** a secret file already exists for the chosen identifier/sub-identifier/spec combination
- **When** creation is attempted
- **Then** the user is warned that the secret already exists
- **And** three options are presented: Quit (default), Open in EDITOR, Proceed and overwrite
- **And** if "Proceed and overwrite" is chosen, final confirmation is required with default "N"
- **And** only "y" or "Y" proceeds with the overwrite

### 5.4 Open Existing Secret in Editor
- **Given** a secret already exists and user chooses "Open in EDITOR"
- **When** the option is selected
- **Then** the existing secret file is decrypted and opened in $EDITOR
- **And** this bypasses the normal creation flow
- **And** the file can be modified directly
- **And** after saving, the file is re-encrypted and committed to git
- **And** this functions identically to the `fuzzlock modify` command

### 5.5 Secret Creation Without MasterKeyFile
- **Given** no MasterKeyFile exists in the system
- **When** `fuzzlock create` is attempted
- **Then** the master key creation process is triggered automatically
- **And** the user is prompted to create a MasterGpgKey and password
- **And** only after successful key creation does the secret creation continue

## 6. Secret Copying

### 6.1 Default Copy Operation
- **Given** a user runs `fuzzlock` without arguments
- **When** the command executes
- **Then** `fuzzlock copy account` is executed (built-in account spec)

### 6.2 Spec-based Copying
- **Given** a user runs `fuzzlock copy <spec>` with a valid spec
- **When** the command executes
- **Then** fzf displays all available secrets for that spec in format "identifier/sub-identifier"
- **And** only secrets matching the specified spec are shown

### 6.3 Field-specific Copying
- **Given** a user runs `fuzzlock copy <spec> <field>`
- **When** a secret is selected
- **Then** the specified field is validated against the spec's schema
- **And** if valid, only that field's value is copied to clipboard
- **And** a confirmation message is displayed (e.g., "Password copied...")
- **And** if invalid, an error message is displayed

### 6.4 All Fields Copying
- **Given** a user runs `fuzzlock copy <spec>` without specifying a field
- **When** a secret is selected
- **Then** each field in the spec's schema is copied to clipboard one at a time
- **And** for each field, a message is displayed with the capitalized field name
- **And** user presses Enter to proceed to the next field
- **And** when all fields are copied, "All fields copied!" is displayed

### 6.5 Content Processing
- **Given** secret content is copied to clipboard
- **When** the copy operation occurs
- **Then** all content is trimmed of whitespace and newlines

### 6.6 Invalid Spec Command Handling
- **Given** a user runs `fuzzlock copy <invalid-spec>`
- **When** the command is executed
- **Then** an appropriate error message is displayed
- **And** no fzf selection interface appears

### 6.7 No Secrets Available
- **Given** no secrets exist for a specified spec
- **When** `fuzzlock copy <spec>` is executed
- **Then** an appropriate message is displayed indicating no secrets are available
- **And** the operation exits gracefully

### 6.8 Session Password Handling
- **Given** a user starts a copy operation
- **When** the MasterKeyFile password is required
- **Then** the password is requested only once per session
- **And** subsequent operations in the same session use the cached authentication

### 6.9 Copying Without MasterKeyFile
- **Given** no MasterKeyFile exists in the system
- **When** any copy operation is attempted
- **Then** the master key creation process is triggered automatically
- **And** the user must create a MasterGpgKey before copying can proceed

### 6.10 Missing Fields in Secret File
- **Given** a secret file is missing some fields from the spec schema
- **When** copying all fields is attempted
- **Then** missing fields are skipped gracefully
- **And** only existing fields are copied to clipboard
- **And** no error is displayed for missing fields

### 6.11 Extra Fields in Secret File  
- **Given** a secret file contains fields not in the spec schema
- **When** copying all fields is attempted
- **Then** extra fields are ignored during copying
- **And** only fields listed in the spec schema are processed
- **And** fields are processed in schema order, not file order

### 6.12 Field Capitalization Display
- **Given** field names in the schema (e.g., "username", "backup-key")
- **When** copying fields and displaying confirmation messages
- **Then** field names are capitalized for display (e.g., "Username copied...", "Backup-key copied...")
- **And** capitalization preserves hyphens and word boundaries

## 7. Secret Modification

### 7.1 Secret Selection for Modification
- **Given** a user runs `fuzzlock modify`
- **When** the selection interface appears
- **Then** fzf displays all existing secret files in format "identifier/sub-identifier/spec.gpg"

### 7.2 Secret Editing
- **Given** a secret file is selected for modification
- **When** the file is opened
- **Then** the file is decrypted and opened in the user's $EDITOR
- **And** the content displays key-value pairs in plain text format
- **And** after saving and closing, the file is re-encrypted
- **And** the change is committed to git

### 7.3 Modification Cancellation
- **Given** a secret file is opened for editing
- **When** the editor is closed without saving or with an empty file
- **Then** the modification is cancelled
- **And** the original secret remains unchanged

### 7.4 No Secrets Available for Modification
- **Given** no secret files exist in the vault
- **When** `fuzzlock modify` is executed
- **Then** an appropriate message is displayed indicating no secrets are available
- **And** the operation exits gracefully

### 7.5 Editor Environment Variable
- **Given** the $EDITOR environment variable is not set
- **When** `fuzzlock modify` is attempted
- **Then** an appropriate error message is displayed
- **And** the operation fails gracefully

### 7.6 Malformed Secret File Content
- **Given** a secret file contains malformed key:value pairs
- **When** the file is opened for modification
- **Then** the raw content is displayed in the editor
- **And** the user can correct the format manually
- **And** validation occurs when the file is saved and re-encrypted

### 7.7 Git Commit Failure During Modification
- **Given** a secret file is successfully modified and encrypted
- **When** the automatic git commit fails
- **Then** the modified file remains encrypted in place
- **And** an appropriate error message is displayed about the git failure
- **And** the user can attempt to resolve git issues manually

## 8. Secret Deletion

### 8.1 Secret Selection for Deletion
- **Given** a user runs `fuzzlock delete`
- **When** the selection interface appears
- **Then** fzf displays all existing secret files in format "identifier/sub-identifier/spec.gpg"

### 8.2 Deletion Confirmation
- **Given** a secret file is selected for deletion
- **When** the deletion process begins
- **Then** confirmation is required with prompt "Are you sure you want to delete this secret? [N/y]"
- **And** default response is "N" to prevent accidental deletion
- **And** only "y" or "Y" proceeds with deletion
- **And** after deletion, the change is committed to git

### 8.3 No Secrets Available for Deletion
- **Given** no secret files exist in the vault
- **When** `fuzzlock delete` is executed
- **Then** an appropriate message is displayed indicating no secrets are available
- **And** the operation exits gracefully

### 8.4 Directory Cleanup
- **Given** a secret file is the last file in its subdirectory
- **When** the secret is deleted
- **Then** the empty subdirectory is removed
- **And** if the identifier directory becomes empty, it is also removed

## 9. Secret Listing

### 9.1 Directory Tree Display
- **Given** secrets exist in the vault
- **When** `fuzzlock list` is executed
- **Then** all secrets are displayed in a directory tree format
- **And** only secret files and the MasterKeyFile are shown
- **And** colors are used when possible
- **And** the complete structure shows identifiers, sub-identifiers, and secret files

### 9.2 Empty Vault Listing
- **Given** no secrets exist in the vault (only MasterKeyFile)
- **When** `fuzzlock list` is executed
- **Then** the directory tree shows only the .secrets directory and MasterKeyFile
- **And** appropriate indication is given that no secrets exist

### 9.3 SpecFile in Listing
- **Given** a SpecFile exists in the vault
- **When** `fuzzlock list` is executed
- **Then** the SpecFile (.secrets/.specs) is displayed in the tree
- **And** it is visually distinguished from secret files

## 10. Export and Import

### 10.1 Basic Export
- **Given** a user runs `fuzzlock export`
- **When** no path is provided
- **Then** the user is prompted whether to encrypt the export with default "Y"
- **And** an archive is created in the current directory
- **And** unencrypted exports are named "secrets.tar"
- **And** encrypted exports are named "secrets.tar.gpg"

### 10.2 Export with Path
- **Given** a user runs `fuzzlock export <path>`
- **When** a directory path is provided
- **Then** the archive is created in that directory with default naming
- **When** a file path is provided
- **Then** the path is used as base name, discarding extensions after the first period
- **When** parent directories don't exist
- **Then** the final component becomes the filename

### 10.3 Export Overwrite Protection
- **Given** a target archive already exists
- **When** export is attempted
- **Then** the user is prompted "Archive already exists. Overwrite? [N/y]"
- **And** default response is "N" to prevent accidental overwriting

### 10.4 Import Validation
- **Given** a user runs `fuzzlock import <archive_path>`
- **When** `~/.secrets` already exists
- **Then** the command fails with an error message
- **And** no import is performed

### 10.5 Import Execution
- **Given** `~/.secrets` does not exist
- **When** import is executed with a valid archive
- **Then** the archive type is determined using MIME type detection
- **And** if encrypted, the user is prompted for the decryption password
- **And** the archive is extracted to `~/.secrets`
- **And** the git repository is restored as part of the archive

### 10.6 Export Path Validation
- **Given** a user provides an invalid export path
- **When** export is attempted
- **Then** appropriate error handling occurs for permission issues
- **And** appropriate error handling occurs for non-existent paths
- **And** meaningful error messages are displayed

### 10.7 Import Archive Validation
- **Given** a user provides an invalid archive path
- **When** import is attempted
- **Then** appropriate error messages are displayed for non-existent files
- **And** appropriate error messages are displayed for invalid archive formats
- **And** appropriate error messages are displayed for corrupted archives

### 10.8 Symmetric Encryption Password
- **Given** an encrypted export is created
- **When** the user provides an encryption password
- **Then** the password is used for AES256 symmetric encryption
- **And** the same password is required for import
- **Given** an incorrect password is provided during import
- **When** decryption is attempted  
- **Then** decryption fails with an appropriate error message

### 10.9 Export Contents Validation
- **Given** an export is created
- **When** the archive contents are examined
- **Then** all secret files are included
- **And** the MasterKeyFile is included
- **And** the SpecFile is included (if it exists)
- **And** the git repository data is included
- **And** the .gitignore file is included

### 10.10 Export Path Edge Cases
- **Given** export paths with multiple periods (e.g., "backup.2024.01.15.zip")
- **When** export path processing occurs
- **Then** everything after the FIRST period is discarded
- **And** the result is "backup.tar" or "backup.tar.gpg"
- **And** only the first period is significant for extension removal

### 10.11 Export Directory vs File Path Detection
- **Given** various path formats are provided to export
- **When** path analysis occurs
- **Then** existing directories are detected correctly
- **And** non-existent parent directories result in filename usage
- **And** the final path component is used as basename when parents don't exist

### 10.12 MIME Type Detection Accuracy
- **Given** various archive formats are provided to import
- **When** MIME type detection occurs
- **Then** .tar files are detected correctly regardless of extension
- **And** .tar.gpg files are detected as GPG-encrypted
- **And** corrupted or invalid files are identified appropriately
- **And** unsupported archive formats are rejected with clear error messages

### 10.13 Import Location Independence
- **Given** import command is run from various working directories
- **When** import extraction occurs
- **Then** files are ALWAYS extracted to `~/.secrets`
- **And** the current working directory does not affect extraction location
- **And** path resolution works consistently across different invocation contexts

## 11. Git Repository Management

### 11.1 Automatic Commits
- **Given** any secret operation (create, modify, delete) is performed
- **When** the operation completes successfully
- **Then** changes are automatically committed to git
- **And** commit messages follow the format "[operation]: [SubIdentifier] [SpecType] for [Identifier]"

### 11.2 Repository State Validation
- **Given** staged but uncommitted changes exist in the repository
- **When** any fuzzlock operation is attempted
- **Then** the user must resolve these changes before proceeding
- **And** the operation is blocked until the repository is clean

### 11.3 Repository Reset
- **Given** staged changes are detected
- **When** `fuzzlock reset` is executed
- **Then** the user is warned with "This will reset all changes and remove untracked files. Continue? [N/y]"
- **And** default response is "N" to prevent accidental data loss
- **And** if confirmed, all changes are reset to HEAD and untracked files are removed

### 11.4 Git Repository Initialization Details
- **Given** no git repository exists in .secrets
- **When** fuzzlock initializes the repository
- **Then** git init is performed in the .secrets directory
- **And** a .gitignore file is created with appropriate exclusions
- **And** temporary files and editor backups are ignored

### 11.5 ASCII Armor Format Validation
- **Given** secret files are encrypted
- **When** they are stored in the git repository
- **Then** all files use ASCII armor format for git compatibility
- **And** binary compatibility issues are avoided

### 11.6 Commit Message Format Validation
- **Given** various operations are performed (create, modify, delete)
- **When** automatic commits are made
- **Then** commit messages follow exactly the format "[operation]: [SubIdentifier] [SpecType] for [Identifier]"
- **And** examples include "create: user1 account for example.com", "delete: foo account for example.com"

### 11.7 Repository State Check Timing
- **Given** any fuzzlock operation is about to begin
- **When** the operation starts
- **Then** git repository state is checked BEFORE any other processing
- **And** if staged changes exist, the operation is blocked immediately
- **And** this check occurs for every command: create, modify, delete, copy, list, export, import

### 11.8 Git Repository Corruption Detection
- **Given** the .secrets directory exists but git repository is corrupted
- **When** any operation attempts to access git
- **Then** corruption is detected and reported
- **And** appropriate guidance is provided for manual recovery
- **And** operations are blocked until repository is repaired

### 11.9 Non-Git Directory Handling
- **Given** .secrets directory exists but is not a git repository
- **When** any operation is attempted
- **Then** git initialization is attempted automatically
- **And** if initialization fails, appropriate error handling occurs
- **And** operations only proceed after successful git repository creation

## 12. Input Validation and Error Handling

### 12.1 Identifier and SubIdentifier Validation
- **Given** a user provides an Identifier or SubIdentifier
- **When** validation occurs
- **Then** only alphanumeric characters and hyphens are accepted
- **And** the value must be valid as a POSIX directory name
- **And** whitespace and newlines are trimmed before validation
- **And** blank or whitespace-only entries are rejected

### 12.2 FieldName Validation
- **Given** field names are used in specs
- **When** validation occurs
- **Then** only lowercase, alphanumeric characters are allowed
- **And** hyphens are allowed only to join separate words
- **And** no other characters (including underscores) are permitted

### 12.3 Error Handling
- **Given** fzf selection is cancelled by the user
- **When** the cancellation occurs
- **Then** the script exits silently
- **Given** other errors occur (GPG failures, file permissions, etc.)
- **When** the error is encountered
- **Then** appropriate error messages are displayed

### 12.4 GPG Key Import Validation
- **Given** a MasterKeyFile exists but key import fails
- **When** import is attempted
- **Then** appropriate error messages are displayed
- **And** the user is informed about potential corruption or invalid key format

### 12.5 File Permission Validation
- **Given** file permission issues occur during operations
- **When** files cannot be read, written, or executed
- **Then** appropriate error messages are displayed with specific permission issues
- **And** guidance is provided for resolving permission problems

### 12.6 Disk Space Validation
- **Given** insufficient disk space exists for operations
- **When** file operations are attempted
- **Then** appropriate error messages are displayed
- **And** operations fail gracefully without partial file corruption

## 13. Command Line Interface

### 13.1 Help System
- **Given** a user runs `fuzzlock --help` or `fuzzlock -h`
- **When** the help is displayed
- **Then** all available commands and usage information are shown
- **And** the help is provided by Python's argparse module

### 13.2 Command Parsing
- **Given** various command formats are used
- **When** commands are parsed
- **Then** both single-character and multi-character spec commands are accepted
- **And** the order of commands in spec definitions doesn't matter

### 13.3 Invalid Command Handling  
- **Given** a user provides an invalid command or arguments
- **When** the command is parsed
- **Then** appropriate error messages are displayed
- **And** help information is provided for correct usage
- **And** the script exits with appropriate error codes

### 13.4 FZF Configuration
- **Given** fzf is used for selection interfaces
- **When** selection screens are displayed
- **Then** fzf shows a fixed vertical size displaying ~10 results
- **And** fuzzy selection capabilities are available
- **And** consistent behavior across all selection operations

### 13.5 Help Menu Integration
- **Given** custom specs with commands and help text exist
- **When** help information is displayed
- **Then** all spec commands (single and multi-character) are shown
- **And** help text from spec definitions is displayed
- **And** built-in account spec help is included

### 13.6 FZF Selection Cancellation
- **Given** any fzf selection interface is displayed
- **When** the user cancels the selection (Ctrl+C, ESC, etc.)
- **Then** the script exits silently without error messages
- **And** no partial operations are performed
- **And** the exit is clean and immediate

### 13.7 Empty FZF Selection Results
- **Given** an fzf selection interface has no items to display
- **When** the selection screen would appear
- **Then** appropriate handling occurs for empty selection lists
- **And** the user is informed about the lack of available options
- **And** the operation exits gracefully

### 13.8 Command Line Argument Validation
- **Given** invalid command line arguments are provided
- **When** argument parsing occurs
- **Then** specific error messages identify the invalid arguments
- **And** usage information is provided automatically
- **And** the script exits with appropriate error codes

### 13.9 Session and Command Lifecycle
- **Given** multiple operations are performed in sequence
- **When** each command completes or fails
- **Then** session state is maintained appropriately between commands
- **And** authentication persists correctly across operations
- **And** failures in one operation don't corrupt session state for subsequent operations

## 14. Security and Encryption

### 14.1 Encryption Abstraction
- **Given** the encryption system is implemented
- **When** any encryption operation occurs
- **Then** all implementation details are hidden behind a single encryption class
- **And** the class is general enough to support different encryption methods
- **And** the user experience remains consistent regardless of underlying encryption

### 14.2 Password Recovery
- **Given** a user forgets their MasterKeyFile password
- **When** they attempt to access their secrets
- **Then** no recovery mechanism is available
- **And** permanent loss of access to all secrets occurs
- **And** users must rely on their own backups

### 14.3 Key Fingerprint Validation
- **Given** a MasterKeyFile with a specific fingerprint exists
- **When** operations are performed
- **Then** the fingerprint in the filename matches the actual key fingerprint
- **And** fingerprint mismatches are detected and reported
- **And** only one MasterKeyFile can exist at any time

### 14.4 GPG Key Generation Validation
- **Given** a new MasterGpgKey needs to be generated
- **When** key generation occurs
- **Then** a 4096-bit GPG private key is created
- **And** the key uses appropriate cryptographic standards
- **And** the key is properly formatted for ASCII armor export

### 14.5 Encryption Algorithm Validation
- **Given** files need to be encrypted
- **When** encryption operations occur
- **Then** MasterKeyFile uses AES256 encryption
- **And** secret files use GPG asymmetric encryption with the MasterGpgKey
- **And** export archives use GPG with AES256 symmetric encryption

### 14.6 Session Security
- **Given** a session is active with unlocked keys
- **When** the session ends or times out
- **Then** appropriate cleanup of sensitive data occurs
- **And** keys are properly managed in memory
- **And** no sensitive data persists unnecessarily

### 14.7 Multiple MasterKeyFile Detection
- **Given** multiple .masterkey-*.asc files exist in .secrets
- **When** any operation is attempted
- **Then** this inconsistent state is detected
- **And** operations are blocked until only one MasterKeyFile remains
- **And** clear guidance is provided for resolving the conflict

### 14.8 Fingerprint Mismatch Detection
- **Given** a MasterKeyFile filename contains a fingerprint
- **When** the key is imported and checked
- **Then** the actual key fingerprint is verified against the filename
- **And** mismatches are detected and reported as potential corruption
- **And** operations are blocked until fingerprint consistency is restored

### 14.9 GPG System Integration
- **Given** GPG is not properly configured on the system
- **When** key operations are attempted
- **Then** GPG configuration issues are detected and reported
- **And** specific guidance is provided for GPG setup
- **And** operations fail gracefully without corrupting data

### 14.10 Encryption Operation Failures
- **Given** encryption or decryption operations fail
- **When** secret file operations are attempted
- **Then** failures are detected before file corruption occurs
- **And** original data is preserved when encryption fails
- **And** clear error messages indicate the nature of the failure

## 15. Spec File Editing

### 15.1 Basic Spec Edit Operation
- **Given** a user runs `fuzzlock spec edit`
- **When** the command executes
- **Then** the SpecFile `.secrets/.specs` is opened in the user's $EDITOR
- **And** if the SpecFile doesn't exist, a new empty file is created and opened
- **And** the user can add, modify, or remove spec definitions

### 15.2 Spec Edit Validation Flow
- **Given** a SpecFile is edited and saved
- **When** the editor closes
- **Then** immediate validation occurs on the saved content
- **And** if validation passes, the operation completes successfully
- **And** if validation fails, the user is prompted to edit again or cancel

### 15.3 Spec Edit Cancellation
- **Given** a user is editing the SpecFile
- **When** the editor is closed without saving or with no changes
- **Then** the edit operation is cancelled
- **And** the original SpecFile remains unchanged
- **And** no validation is performed

### 15.4 Spec Edit Error Recovery
- **Given** validation fails after editing
- **When** the user chooses to edit again
- **Then** the editor reopens with the invalid content
- **And** the user can correct the validation errors
- **And** validation is re-attempted after saving
- **Given** the user chooses to cancel after validation failure
- **When** cancellation is confirmed
- **Then** the original SpecFile is restored
- **And** any invalid changes are discarded

### 15.5 Editor Environment for Spec Edit
- **Given** the $EDITOR environment variable is not set
- **When** `fuzzlock spec edit` is attempted
- **Then** an appropriate error message is displayed
- **And** the operation fails gracefully

### 15.6 Invalid JSON Handling
- **Given** the SpecFile contains invalid JSON after editing
- **When** validation occurs
- **Then** JSON parsing errors are detected and reported clearly
- **And** the user is prompted to correct the JSON syntax
- **And** specific error location information is provided when possible

### 15.7 Empty SpecFile Handling
- **Given** a user saves an empty SpecFile during editing
- **When** validation occurs
- **Then** the empty file is treated as valid (no custom specs)
- **And** only built-in specs remain available
- **And** no error is displayed for empty SpecFiles

## 16. Code Quality and Testing

### 16.1 Single File Implementation
- **Given** the implementation requirements
- **When** the script is built
- **Then** the entire functionality is contained in a single Python file
- **And** only standard library dependencies are used
- **And** external runtime dependencies are limited to gpg and fzf

### 16.2 Test Coverage
- **Given** the testing requirements
- **When** tests are run
- **Then** extensive test coverage exists for all functionality
- **And** unit tests cover all major components
- **And** all tests pass
- **And** pytest is used as the test runner

### 16.3 Code Standards
- **Given** code quality requirements
- **When** code is evaluated
- **Then** Python best practices are followed
- **And** code is highly testable with proper separation of concerns
- **And** all linting checks pass
- **And** code formatting meets Python standards

## 17. Edge Cases and Boundary Conditions

### 17.1 Empty Input Handling
- **Given** a user provides empty strings for any input field
- **When** validation occurs
- **Then** the input is rejected with appropriate error messages
- **And** the user is re-prompted for valid input

### 17.2 Whitespace-only Input Handling
- **Given** a user provides strings containing only whitespace
- **When** input trimming and validation occurs
- **Then** the input is treated as empty and rejected
- **And** appropriate error messages are displayed

### 17.3 Maximum Input Length Handling
- **Given** extremely long input strings are provided
- **When** validation and processing occurs
- **Then** appropriate limits are enforced for practical usage
- **And** system resources are protected from excessive usage

### 17.4 Special Character Handling
- **Given** inputs contain special characters, unicode, or non-ASCII content
- **When** validation occurs for identifiers and field names
- **Then** only allowed characters pass validation
- **And** clear error messages explain character restrictions

### 17.5 Concurrent Operation Handling
- **Given** multiple fuzzlock instances attempt to run simultaneously
- **When** file locking or repository access conflicts occur
- **Then** appropriate error handling prevents data corruption
- **And** meaningful error messages inform users of conflicts

### 17.6 Corrupted File Recovery
- **Given** secret files or the MasterKeyFile become corrupted
- **When** operations attempt to access these files
- **Then** corruption is detected and reported
- **And** guidance is provided for recovery options
- **And** operations fail safely without further corruption

### 17.7 Large Repository Handling
- **Given** a secrets repository contains thousands of secrets
- **When** listing or selection operations occur
- **Then** performance remains acceptable
- **And** fzf selection remains usable
- **And** git operations complete successfully

### 17.8 Network Interruption Handling
- **Given** network-dependent operations are interrupted
- **When** git operations or external dependencies are affected
- **Then** operations fail gracefully with appropriate error messages
- **And** repository state remains consistent

### 17.9 Filesystem Full Conditions
- **Given** the filesystem becomes full during operations
- **When** file writes are attempted
- **Then** operations fail gracefully without partial writes
- **And** existing files are not corrupted
- **And** appropriate error messages are displayed

### 17.10 Clock/Time Issues
- **Given** system clock issues affect git operations
- **When** commits are attempted
- **Then** operations complete successfully regardless of timestamp issues
- **And** commit history remains valid

## 18. Specific Prompt and Message Validation

### 18.1 Exact Prompt Text Validation
- **Given** various confirmation prompts are displayed
- **When** user interaction is required
- **Then** prompts match exactly: "Are you sure you want to overwrite the existing secret? [N/y]"
- **And** prompts match exactly: "Are you sure you want to delete this secret? [N/y]"
- **And** prompts match exactly: "Archive already exists. Overwrite? [N/y]"
- **And** prompts match exactly: "Encrypt export? [Y/n]"
- **And** prompts match exactly: "This will reset all changes and remove untracked files. Continue? [N/y]"

### 18.2 Default Response Behavior
- **Given** confirmation prompts with default values are displayed
- **When** the user presses Enter without typing
- **Then** default "N" responses are processed for deletion and overwrite prompts
- **And** default "Y" response is processed for export encryption prompt
- **And** defaults are clearly indicated in the prompt format

### 18.3 Response Case Sensitivity
- **Given** confirmation prompts require y/Y responses
- **When** users provide responses
- **Then** both "y" and "Y" are accepted for positive confirmation
- **And** all other responses (including "yes", "YES") are treated as negative
- **And** case sensitivity is consistent across all prompts

## 19. File Format and Content Validation

### 19.1 Secret File Format Validation
- **Given** secret files are created or modified
- **When** file content is processed
- **Then** content follows exact key:value format with colons
- **And** one key:value pair per line
- **And** no leading or trailing whitespace on keys or values
- **And** empty lines are handled appropriately

### 19.2 ASCII Armor Format Enforcement
- **Given** secret files are encrypted for git storage
- **When** encryption occurs
- **Then** files are stored in GPG ASCII armor format
- **And** files contain proper ASCII armor headers and footers
- **And** binary data is properly encoded for text storage
- **And** git can handle the files as text without corruption

### 19.3 Home Directory Path Resolution
- **Given** various home directory configurations exist
- **When** ~/.secrets path is resolved
- **Then** path resolution works with different $HOME values
- **And** path resolution works with symbolic links in home path
- **And** path resolution handles non-standard home directory layouts

### 19.4 POSIX Directory Name Validation
- **Given** identifiers and sub-identifiers are validated as POSIX directory names
- **When** validation occurs
- **Then** names starting with dots are rejected (hidden files)
- **And** names containing forward slashes are rejected
- **And** names containing null bytes are rejected
- **And** names exceeding filesystem limits are rejected
- **And** reserved names like "." and ".." are rejected

## 20. System Integration Edge Cases

### 20.1 Clipboard Operation Failures
- **Given** clipboard operations are attempted
- **When** system clipboard is unavailable or fails
- **Then** appropriate error messages are displayed
- **And** secret values are not left in temporary files or memory
- **And** operations fail gracefully without data exposure

### 20.2 Editor Process Failures
- **Given** $EDITOR process is launched for file editing
- **When** the editor process fails to start or crashes
- **Then** appropriate error handling prevents file corruption
- **And** temporary files are cleaned up properly
- **And** original encrypted files remain intact

### 20.3 Git Configuration Dependencies
- **Given** git operations are required
- **When** git user.name or user.email are not configured
- **Then** appropriate error messages guide configuration
- **And** operations are blocked until git is properly configured
- **And** no commits are attempted with incomplete configuration

### 20.4 Filesystem Case Sensitivity
- **Given** different filesystem case sensitivity behaviors
- **When** file operations are performed
- **Then** identifier/sub-identifier case is preserved correctly
- **And** file path resolution is consistent across filesystems
- **And** no conflicts arise from case-insensitive filesystem behavior
