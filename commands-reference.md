# Commands Reference — Ansible Cross-Platform Software Installation

This reference lists the SSH, Linux, Ansible, Vault and connectivity commands recorded in the project documents.

> **Before publishing:** Replace private/public IPs with placeholders if you prefer not to disclose infrastructure details. Never publish `.pem`/`.ppk` private keys, passwords, Vault passwords or the real encrypted `group_vars/windows.yml`.

## 1. Connect to the Ubuntu Master from your workstation

```bash
ssh -i <path-to-private-key.pem> ubuntu@<UBUNTU_MASTER_PUBLIC_IP>
```

Connects from the local workstation to the Ubuntu EC2 Ansible control node.

## 2. Linux navigation and file management

| Command | What it was used for |
|---|---|
| `pwd` | Check the current working directory. |
| `ls -la` | List all files, including hidden files, permissions and ownership. |
| `cd ~/ansible-project` | Move to the Ansible project root. |
| `cd ~/ansible-project/group_vars` | Move to the Windows group variables directory. |
| `cd ..` | Move one directory level up. |
| `mkdir -p group_vars` | Create the `group_vars` directory if it does not exist. |
| `nano <file>` | Create or edit a file using Nano. |
| `export EDITOR=nano` | Select Nano as the editor for Ansible Vault in the current shell session. |
| `ls -l group_vars/` | Check files in the group variables directory. |
| `rm windows.yml` | Remove the incorrect/plaintext file when located in `group_vars`; verify the current directory before running. |

Nano save/exit: `Ctrl+O`, Enter, then `Ctrl+X`.

## 3. Install Ansible and WinRM support on Ubuntu Master

```bash
sudo apt update
sudo apt install -y python3 python3-pip pipx
pipx install --include-deps ansible
pipx inject ansible pywinrm
ansible --version
```

Purpose:
- Update Ubuntu package metadata.
- Install Python, pip and pipx prerequisites.
- Install Ansible in a pipx-managed environment.
- Add `pywinrm` to the same Ansible environment for Windows WinRM connectivity.
- Check Ansible version, Python interpreter and active configuration.

## 4. Install and inspect Ansible collections

```bash
ansible-galaxy collection install ansible.windows chocolatey.chocolatey
ansible-galaxy collection list
ansible-doc -t connection ansible.builtin.winrm
```

The project uses `ansible.windows` for Windows modules and `chocolatey.chocolatey` for Chocolatey package installation.

## 5. Test network connectivity from Ubuntu Master

```bash
nc -zv <UBUNTU_SLAVE_PRIVATE_IP> 22
nc -zv <WINDOWS_SLAVE_PRIVATE_IP> 5986
```

- TCP 22: SSH connectivity from Master to Ubuntu Slave.
- TCP 5986: WinRM HTTPS connectivity from Master to Windows Slave.

A successful TCP test confirms that the port is reachable; it does not by itself prove that authentication works.

## 6. Test Ansible connectivity

```bash
ansible ubuntu -m ping
ansible windows -m ansible.windows.win_ping --ask-pass
```

After configuring the encrypted Vault file, the Windows test was run with:

```bash
ansible windows -m ansible.windows.win_ping --ask-vault-pass
```

Ansible `ping`/`win_ping` tests Ansible connectivity. It is not the same as the operating system's ICMP `ping`.

## 7. Inspect Ansible configuration and inventory

```bash
ansible --version
ansible-inventory --graph
ansible all -m ping
```

The project uses the root-level inventory file named `hosts`; `ansible.cfg` sets `inventory = ./hosts`.

## 8. Validate and execute the complete project

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --ask-vault-pass
```

- `--syntax-check`: validates playbook syntax without executing the tasks.
- `--ask-vault-pass`: prompts for the Vault decryption password and runs the complete playbook.

## 9. Ansible Vault commands used

Run from the project root:

```bash
export EDITOR=nano
ansible-vault create group_vars/windows.yml
ansible-vault view group_vars/windows.yml
```

The encrypted file contains the Windows `ansible_password` variable. The Windows Administrator password is stored inside Vault; the Vault password decrypts that file. They are not the same password.

## 10. Ubuntu software verification commands

```bash
ansible ubuntu -m shell -a 'git --version'
ansible ubuntu -m shell -a 'java -version'
ansible ubuntu -m shell -a 'docker --version && systemctl is-active docker'
ansible ubuntu -m shell -a 'nginx -v && systemctl is-active nginx'
```

## 11. Windows software verification commands

Run in a new PowerShell terminal if an installer updated PATH:

```powershell
git --version
java -version
Get-Service W3SVC
```

The project assignment also used Chocolatey to verify 7-Zip. The exact `choco list` command can vary by Chocolatey version; use the syntax supported by the installed version.

## 12. Network ports recorded for the lab

| Source | Destination | Protocol / port | Purpose |
|---|---|---|---|
| Ubuntu Master | Ubuntu Slave | SSH TCP 22 | Ansible Linux transport |
| Ubuntu Master | Windows Slave | WinRM HTTPS TCP 5986 | Ansible Windows transport |
| Administrator workstation | Windows Slave | RDP TCP 3389 | Optional remote desktop; restrict to trusted source IP |

Do not leave All TCP ports 0–65535 open to `0.0.0.0/0`.
