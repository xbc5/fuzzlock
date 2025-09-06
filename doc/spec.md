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

## Dependency Management

- At startup, Fuzzlock evaluates all required dependencies.
- If any dependencies are missing, it prints all missing dependencies and exits.
- No operations are performed if dependencies are not satisfied.

## Technology

- **Python**
- **GPG** (for master key generation and AES256 encryption)
- **fzf** (fuzzy selection)

## Testing

- Use **pytest** to run tests.
