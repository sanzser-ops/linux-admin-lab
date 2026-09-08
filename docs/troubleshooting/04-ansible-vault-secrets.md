# Troubleshooting 04 — Ansible Vault Secrets

> **Area:** Ansible / Security / Secrets Management
> **Severity:** Security configuration
> **Status:** Resolved

---

## 1. Problem

The monitoring configuration required an SMTP password for Gmail.

The credential needed to be available to Ansible during deployment without exposing the actual password inside the normal playbook or committing it to Git.

The objective was therefore to securely separate:

- Automation code
- Configuration
- Sensitive credentials

The solution implemented in the lab was Ansible Vault.

---

## 2. Symptoms

The Alertmanager configuration required an SMTP authentication password:

    smtp_auth_password: '{{ vault_smtp_password }}'

The variable `vault_smtp_password` contains sensitive information.

Storing the real password directly inside the Alertmanager playbook would create a security risk because the playbook is stored in Git.

The investigation therefore focused on:

- Where the SMTP password was stored
- How Ansible accessed the password
- Whether the password was exposed in normal files
- Whether the Vault file was protected from Git
- Whether the deployment could successfully use the encrypted secret

---

## 3. Investigation

### 3.1 Search the repository for sensitive information

The repository was searched for common credential-related keywords:

    grep -RniE 'password|secret|token|api_key|private_key|smtp_auth_password' \
      ansible/playbooks \
      ansible/group_vars \
      ansible/host_vars \
      ansible/*.yml \
      --exclude='vault.yml'

The search identified the expected reference in the Alertmanager playbook:

    smtp_auth_password: '{{ vault_smtp_password }}'

This was acceptable because the playbook references the Vault variable rather than containing the actual password.

---

### 3.2 Check the Vault file

The secret was stored in:

    ansible/group_vars/all/vault.yml

The file was protected with restrictive permissions.

The file permissions were checked with:

    ls -la ansible/group_vars/all/

The expected result included restrictive permissions such as:

    -rw------- ... vault.yml

This prevents other local users from reading the file directly.

---

### 3.3 Verify the Vault structure

The Vault contents could be viewed with:

    ansible-vault view ansible/group_vars/all/vault.yml

The file contained the sensitive variable:

    vault_smtp_password: "..."

The actual password is intentionally not documented.

The secret was encrypted using Ansible Vault rather than stored as plain text in the repository.

---

### 3.4 Verify that the Vault file is ignored by Git

The repository uses `.gitignore` to prevent the local Vault file from being accidentally committed.

The relevant entry is:

    ansible/group_vars/all/vault.yml

This was verified with:

    git check-ignore -v ansible/group_vars/all/vault.yml

Git returned the `.gitignore` rule responsible for ignoring the file.

This confirmed that Git was configured to ignore the Vault file.

---

### 3.5 Verify that the Vault file is not tracked

The following command was used:

    git ls-files ansible/group_vars/all/vault.yml

The command returned no output.

This confirmed that the Vault file was not tracked by Git.

This distinction is important:

    Ignored by Git
          !=
    Already removed from Git history

The repository was checked to ensure that the secret file was never added to the tracked files in the current repository state.

---

### 3.6 Verify the Git status

The repository status was checked with:

    git status

The Vault file did not appear as an untracked file because it was correctly ignored.

Only files intended to be committed appeared in Git status.

---

## 4. Root Cause / Finding

The security requirement was caused by the need to provide a sensitive Gmail SMTP credential to Alertmanager.

The password could not safely be stored directly in:

    ansible/playbooks/alertmanager.yml

Instead, the playbook referenced:

    vault_smtp_password

The actual secret was stored in:

    ansible/group_vars/all/vault.yml

The relationship was:

    Alertmanager playbook
            |
            | references
            v
    vault_smtp_password
            |
            | stored in
            v
    Ansible Vault
            |
            | decrypted during deployment
            v
    Alertmanager configuration

The repository therefore contains the automation logic without exposing the actual credential.

---

## 5. Resolution

The SMTP password was stored using Ansible Vault.

The secret variable was defined in the Vault file:

    vault_smtp_password: "..."

The Alertmanager playbook referenced the variable:

    smtp_auth_password: '{{ vault_smtp_password }}'

The Vault file was excluded from Git:

    ansible/group_vars/all/vault.yml

The `.gitignore` configuration was verified with:

    git check-ignore -v ansible/group_vars/all/vault.yml

The Vault file was also verified as untracked with:

    git ls-files ansible/group_vars/all/vault.yml

The deployment was executed with:

    ansible-playbook playbooks/alertmanager.yml -K --ask-vault-pass

Ansible successfully decrypted and used the Vault variable during the deployment.

---

## 6. Validation

### 6.1 Verify Vault file exists

Run:

    ls -la ansible/group_vars/all/

Confirm that:

    vault.yml

exists and has restrictive permissions.

---

### 6.2 Verify Vault encryption

Run:

    ansible-vault view ansible/group_vars/all/vault.yml

A Vault password is required to view the contents.

The actual secret should never be copied into documentation.

---

### 6.3 Verify Git ignores the file

Run:

    git check-ignore -v ansible/group_vars/all/vault.yml

Expected behaviour:

    .gitignore:... ansible/group_vars/all/vault.yml

This confirms that Git ignores the file.

---

### 6.4 Verify the file is not tracked

Run:

    git ls-files ansible/group_vars/all/vault.yml

Expected result:

    no output

This confirms that the Vault file is not part of the tracked repository files.

---

### 6.5 Verify the playbook references the Vault variable

Run:

    grep -Rni 'vault_smtp_password' ansible/playbooks ansible/group_vars \
      --exclude='vault.yml'

The expected reference is:

    smtp_auth_password: '{{ vault_smtp_password }}'

The actual password should not appear in the playbook.

---

### 6.6 Run the deployment

Run:

    ansible-playbook playbooks/alertmanager.yml -K --ask-vault-pass

Expected result:

    PLAY RECAP

    rhel-server-01 : ok=14 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0

This confirms that Ansible was able to use the Vault secret during deployment.

---

### 6.7 Verify Alertmanager

After deployment, verify the service:

    sudo systemctl status alertmanager --no-pager

Expected result:

    Active: active (running)

This confirms that Alertmanager started successfully with the configuration generated from the Vault variable.

---

## 7. Security Flow

The secure configuration flow is:

    +-----------------------------+
    | Ansible Vault               |
    | vault.yml                   |
    |                             |
    | vault_smtp_password: "..."  |
    +-------------+---------------+
                  |
                  | decrypted by Ansible
                  v
    +-----------------------------+
    | Alertmanager playbook       |
    |                             |
    | {{ vault_smtp_password }}   |
    +-------------+---------------+
                  |
                  v
    +-----------------------------+
    | Alertmanager configuration  |
    |                             |
    | smtp_auth_password: ...     |
    +-------------+---------------+
                  |
                  v
             Gmail SMTP

The Vault file remains outside the Git-tracked repository.

---

## 8. Git Protection Flow

The Git protection mechanism is:

    vault.yml
        |
        v
    .gitignore
        |
        v
    Git ignores file
        |
        v
    git status
        |
        | file not shown
        v
    git add .
        |
        v
    vault.yml not staged

The additional verification is:

    git ls-files ansible/group_vars/all/vault.yml

No output confirms that the file is not tracked.

---

## 9. Useful Commands

### Check the Vault file

    ls -la ansible/group_vars/all/

### View the Vault

    ansible-vault view ansible/group_vars/all/vault.yml

### Edit the Vault

    ansible-vault edit ansible/group_vars/all/vault.yml

### Check the Vault file

    ansible-vault view ansible/group_vars/all/vault.yml

### Verify Git ignores the Vault

    git check-ignore -v ansible/group_vars/all/vault.yml

### Verify the Vault is not tracked

    git ls-files ansible/group_vars/all/vault.yml

### Search for credential references

    grep -RniE 'password|secret|token|api_key|private_key|smtp_auth_password' \
      ansible/playbooks \
      ansible/group_vars \
      ansible/host_vars \
      ansible/*.yml \
      --exclude='vault.yml'

### Check repository status

    git status

### Deploy using the Vault

    ansible-playbook playbooks/alertmanager.yml -K --ask-vault-pass

---

## 10. Security Considerations

Sensitive credentials should never be stored directly in normal Ansible playbooks.

Avoid configurations such as:

    smtp_auth_password: 'REAL_PASSWORD'

Instead, use a Vault variable:

    smtp_auth_password: '{{ vault_smtp_password }}'

The actual secret belongs inside the encrypted Vault file:

    ansible/group_vars/all/vault.yml

The Vault file should also be excluded from Git using:

    .gitignore

with:

    ansible/group_vars/all/vault.yml

Restrictive filesystem permissions should also be used for the Vault file.

The actual password should never be included in:

- Git commits
- Documentation
- Screenshots
- Shared terminal output
- Public repositories
- Troubleshooting reports

---

## 11. Important Git Consideration

`.gitignore` only prevents untracked files from being added accidentally.

It does not remove a file that has already been committed.

For this reason, the following verification is important:

    git ls-files ansible/group_vars/all/vault.yml

Expected result:

    no output

In this lab, the Vault file was verified as not being tracked.

The repository therefore contains the Ansible automation without exposing the SMTP credential.

---

## 12. Lessons Learned

- Sensitive credentials should never be hard-coded into Ansible playbooks.
- Ansible Vault provides a mechanism for protecting sensitive variables.
- The playbook should reference Vault variables rather than actual passwords.
- `.gitignore` helps prevent accidental commits of local secret files.
- `git check-ignore` can verify that Git is ignoring a sensitive file.
- `git ls-files` can verify whether a file is tracked.
- A file being ignored is not the same as a file being removed from Git history.
- Restrictive filesystem permissions provide an additional layer of local protection.
- Secrets should never be included in documentation or public troubleshooting reports.
- Security validation should be performed before pushing infrastructure code to a remote repository.

---

## 13. Final Status

**Status: RESOLVED**

The SMTP credential was successfully separated from the Ansible automation code.

The final architecture was:

    Git repository
        |
        +---- Ansible playbooks
        |         |
        |         +---- reference Vault variables
        |
        +---- .gitignore
        |
        +---- Documentation
                  |
                  X
             No passwords

    Local system
        |
        +---- ansible/group_vars/all/vault.yml
                  |
                  +---- Encrypted SMTP credential

The Vault file was verified as ignored by Git and not tracked.

The Alertmanager deployment successfully used the Vault variable.

The monitoring configuration therefore provides both automation and protection of sensitive credentials.
