# Troubleshooting Log — Ansible Cross-Platform Software Installation

This document captures the issues and outcomes recorded during the project. It focuses particularly on establishing Ansible connectivity to the Windows Server managed node.

## 1. Windows connection overview

The control node was Ubuntu Master. The managed Windows Server was accessed through **WinRM over HTTPS on TCP 5986**, not through SSH.

Recorded connection path:

```text
Ubuntu Master (Ansible Control Node)
          |
          | WinRM HTTPS / TCP 5986
          v
Windows Slave (Windows Server 2022)
```

The Windows inventory selected the WinRM connection plugin and NTLM authentication. The Windows connection did not use the Ubuntu SSH private key.

## 2. Windows WinRM troubleshooting

### Issue 1 — WinRM listener mismatch: HTTP 5985 vs HTTPS 5986

**Observed:** The initial Windows WinRM listener configuration did not match the project's inventory, which expected HTTPS on port 5986.

**Why it mattered:** Ansible's Windows inventory was configured for HTTPS/5986. A listener on HTTP/5985 would not satisfy that configuration.

**Resolution recorded:** Configure a WinRM HTTPS listener on TCP 5986, configure a certificate, create a matching Windows Firewall inbound rule, and make the Ansible inventory settings agree with the listener.

**Connectivity check from Ubuntu Master:**

```bash
nc -zv <WINDOWS_SLAVE_PRIVATE_IP> 5986
```

The TCP port test eventually succeeded.

**Important:** A successful `nc` test proves TCP reachability only. Authentication and Ansible module execution must be tested separately.

The exact initial PowerShell commands used to create the certificate and listener were not fully retained in the source troubleshooting document, so they are not reproduced as if they were the original commands.

### Issue 2 — Windows Firewall / Public network profile

**Observed:** The Windows network profile was Public.

**Resolution recorded:** Add an explicit inbound firewall rule for WinRM HTTPS and scope it to the Ubuntu Master private IP rather than opening the port to all sources.

The recorded lab rule used:
- Direction: Inbound
- Protocol: TCP
- Local port: 5986
- Action: Allow
- Remote address: Ubuntu Master private IP

For a public portfolio, use placeholders instead of publishing actual infrastructure addresses.

### Issue 3 — TCP connection worked, but authentication still needed testing

**Observed:** `nc -zv <WINDOWS_SLAVE_PRIVATE_IP> 5986` succeeded.

**Meaning:** The Windows WinRM HTTPS port was reachable from the Master. This alone did not confirm valid Windows credentials.

**Next test:**

```bash
ansible windows -m ansible.windows.win_ping --ask-pass
```

The command eventually returned `pong`.

### Issue 4 — Ansible prompt displayed “SSH password” for the Windows host

**Observed:** When `--ask-pass` was used, the terminal displayed the generic prompt `SSH password:`.

**Confusion:** The prompt wording suggested SSH, although the Windows host was configured to use WinRM.

**Explanation from the lab configuration:** The inventory selected `ansible_connection=winrm`; therefore the Windows connection was WinRM. The prompt text did not change the selected transport.

**Successful test after configuration:**

```bash
ansible windows -m ansible.windows.win_ping --ask-pass
```

### Issue 5 — `pywinrm` prerequisite on the Ansible control node

**Requirement:** The Ansible control node needs the Python WinRM support library in the same Python environment used by Ansible.

**Installation recorded:**

```bash
pipx inject ansible pywinrm
ansible --version
```

Use `ansible --version` to inspect which Python runtime is running Ansible. Installing `pywinrm` into an unrelated Python environment may not resolve the connection dependency.

### Issue 6 — Windows password prompt vs Ansible Vault password

**Confusion:** The Windows Administrator password and the Vault password were initially treated as if they were the same.

**Correct distinction:**

| Secret | Purpose |
|---|---|
| Windows Administrator password | Authenticates Ansible to Windows; stored as `ansible_password` inside the encrypted group variables file. |
| Ansible Vault password | Decrypts the Vault file so Ansible can read the Windows credential. |

The project created the encrypted file at `group_vars/windows.yml`.

**Vault commands recorded:**

```bash
export EDITOR=nano
ansible-vault create group_vars/windows.yml
ansible-vault view group_vars/windows.yml
```

After Vault was configured, the Windows connectivity test used:

```bash
ansible windows -m ansible.windows.win_ping --ask-vault-pass
```

### Issue 7 — “Input is not vault encrypted data”

**Observed:** Ansible Vault reported that the input was not Vault-encrypted.

**Reason:** The YAML file had been created as a normal plaintext file in Nano. A YAML file does not become encrypted merely because it contains a password variable.

**Resolution recorded:** Remove the incorrect plaintext file after checking the current directory, then recreate it using `ansible-vault create`.

```bash
# Run from ~/ansible-project/group_vars only after confirming pwd
rm windows.yml
export EDITOR=nano
ansible-vault create windows.yml
```

The encrypted content contains:

```yaml
ansible_password: "YOUR_WINDOWS_ADMIN_PASSWORD"
```

Never publish the actual password or expose it in screenshots.

### Issue 8 — Duplicate `group_vars` path

**Observed:** While already inside `~/ansible-project/group_vars`, running:

```bash
ansible-vault view group_vars/windows.yml
```

caused Ansible to look for `group_vars/group_vars/windows.yml`.

**Reason:** The path was specified relative to the current directory, so the directory name was repeated.

**Correct commands:**

```bash
# If current directory is ~/ansible-project/group_vars
ansible-vault view windows.yml

# If current directory is ~/ansible-project
ansible-vault view group_vars/windows.yml
```

## 3. Other project troubleshooting recorded

### Issue 9 — ICMP ping failed but SSH was reachable

**Observed:** Ordinary ping to Ubuntu Slave showed packet loss, while TCP port 22 was reachable and SSH/Ansible connectivity later worked.

**Reason:** ICMP and SSH use different network paths. ICMP failure does not prove SSH is unavailable.

**Check:**

```bash
nc -zv <UBUNTU_SLAVE_PRIVATE_IP> 22
ansible ubuntu -m ping
```

### Issue 10 — Chocolatey module/collection prerequisite

**Requirement:** The Windows software roles used `chocolatey.chocolatey.win_chocolatey`.

**Resolution:**

```bash
ansible-galaxy collection install chocolatey.chocolatey
ansible-galaxy collection list
```

The lab recorded collection version 1.6.0 on the controller and Chocolatey 2.7.5 on Windows.

### Issue 11 — Java not found immediately after installation

**Reason:** Windows PATH changes may not appear in a terminal that was already open.

**Resolution:** Open a new Windows terminal and run `java -version`.

### Issue 12 — Inventory path / no hosts matched

**Project configuration:** Inventory is the root-level file `hosts`, and `ansible.cfg` specifies `inventory = ./hosts`.

**Checks:**

```bash
pwd
ansible --version
ansible-inventory --graph
ansible all -m ping
```

Run Ansible from the project root and verify the active configuration and inventory.

### Issue 13 — Role file placement

Each role task file belongs at `roles/<role-name>/tasks/main.yml`. The project role names were `git`, `java17`, `docker`, `nginx`, `sevenzip`, and `iis`. Ensure role names in `site.yml` match their directory names.

### Issue 14 — Confirm Windows edition before installing IIS

The Windows OS check returned:

```text
Microsoft Windows Server 2022 Datacenter
```

This confirmed that the `ansible.windows.win_feature` module with feature name `Web-Server` was suitable for the target.

## 4. Final connectivity and deployment evidence

The project completed the Windows WinRM connectivity test and the complete playbook run.

**Final playbook command:**

```bash
ansible-playbook site.yml --ask-vault-pass
```

**First full run:**

| Host | ok | changed | unreachable | failed | skipped |
|---|---:|---:|---:|---:|---:|
| Ubuntu Slave | 10 | 0 | 0 | 0 | 2 |
| Windows Slave | 9 | 2 | 0 | 0 | 2 |

**Second full run:**

| Host | ok | changed | unreachable | failed | skipped |
|---|---:|---:|---:|---:|---:|
| Ubuntu Slave | 10 | 0 | 0 | 0 | 2 |
| Windows Slave | 9 | 0 | 0 | 0 | 2 |

The second run reported `changed=0` on both managed nodes, confirming idempotency for the tested configuration.

## 5. Details not reconstructed

The source troubleshooting document states that:
- Exact EC2 User Data script contents for Python/pip bootstrap were not retained.
- The complete initial PowerShell commands for WinRM certificate/listener creation were not retained.
- Exact output from every intermediate WinRM attempt was not retained.

These items are intentionally marked as unavailable rather than replaced with guessed historical commands.
