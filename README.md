# Cross-Platform Software Installation with Ansible

Automated installation of software on Ubuntu Linux and Windows Server 2022 EC2 instances using one Ansible playbook and six reusable roles.

## Project overview

A single `site.yml` playbook calls six roles. Ansible gathers OS facts and uses `when` conditions to run software installation tasks only on the intended operating system.

### AWS EC2 lab topology

| Instance | Purpose | OS | Private IP |
|---|---|---|---|
| Ubuntu Master | Ansible control node | Ubuntu | `172.31.7.141` |
| Ubuntu Slave | Managed node | Ubuntu | `172.31.8.15` |
| Windows Slave | Managed node | Windows Server 2022 Datacenter | `172.31.9.52` |

### Software matrix

| Software | Ubuntu | Windows |
|---|:---:|:---:|
| Git | Yes | Yes |
| Java 17 | Yes | Yes |
| Docker | Yes | — |
| Nginx | Yes | — |
| 7-Zip | — | Yes |
| IIS | — | Yes |

## Architecture

- Ubuntu Master to Ubuntu Slave: SSH over TCP 22.
- Ubuntu Master to Windows Slave: WinRM over HTTPS, TCP 5986.
- Windows package manager: Chocolatey.
- Ansible Windows support: `ansible.windows` and `pywinrm`.
- Windows Administrator credential: stored as an Ansible Vault variable.

## Repository structure

```text
ansible-cross-platform-installation/
├── ansible.cfg
├── hosts
├── site.yml
├── group_vars/
│   └── windows.yml.example
├── roles/
│   ├── git/tasks/main.yml
│   ├── java17/tasks/main.yml
│   ├── docker/tasks/main.yml
│   ├── nginx/tasks/main.yml
│   ├── sevenzip/tasks/main.yml
│   └── iis/tasks/main.yml
└── docs/
    ├── 01_Ansible_A_to_Z_Execution_Guide.docx
    ├── 02_SSH_and_Linux_Command_Reference.docx
    └── 03_Ansible_Troubleshooting_Log.docx
```

## Prerequisites

On the Ubuntu control node, install Ansible, `pywinrm`, and the required collections:

```bash
pipx inject ansible pywinrm
ansible-galaxy collection install ansible.windows chocolatey.chocolatey
ansible --version
```

`pywinrm` must be installed in the same Python environment used by Ansible.

## Configure Windows credentials securely

Do not store a real password in plaintext or commit it to GitHub.

```bash
mkdir -p group_vars
export EDITOR=nano
ansible-vault create group_vars/windows.yml
```

Add this variable inside the editor, replacing the placeholder with the Windows Administrator password:

```yaml
ansible_password: "REPLACE_WITH_WINDOWS_ADMIN_PASSWORD"
```

Save and exit. `ansible-vault create` encrypts the file. The repository contains only `group_vars/windows.yml.example`; the real `group_vars/windows.yml` is excluded by `.gitignore`. Keep the Vault password separate and secure.

## Run all roles using one playbook

From the project root:

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --ask-vault-pass
```

The playbook runs Git, Java 17, Docker, Nginx, 7-Zip and IIS roles. The Vault prompt decrypts the Windows credential file.

## Verify installations

### Ubuntu

```bash
ansible ubuntu -m shell -a "git --version"
ansible ubuntu -m shell -a "java -version"
ansible ubuntu -m shell -a "docker --version && systemctl is-active docker"
ansible ubuntu -m shell -a "nginx -v && systemctl is-active nginx"
```

### Windows

```bash
ansible windows -m ansible.windows.win_command -a "git --version" --ask-vault-pass
ansible windows -m ansible.windows.win_command -a "java -version" --ask-vault-pass
ansible windows -m ansible.windows.win_shell -a "choco list 7zip" --ask-vault-pass
ansible windows -m ansible.windows.win_shell -a "Get-Service W3SVC" --ask-vault-pass
```

Open a new Windows terminal before checking commands if installation updated PATH.

## Idempotency test

The same playbook was executed twice.

First run:
```text
ubuntu-slave  : ok=10  changed=0  unreachable=0  failed=0  skipped=2
windows-slave : ok=9   changed=2  unreachable=0  failed=0  skipped=2
```

Second run:
```text
ubuntu-slave  : ok=10  changed=0  unreachable=0  failed=0  skipped=2
windows-slave : ok=9   changed=0  unreachable=0  failed=0  skipped=2
```

The second run reported `changed=0` on both managed nodes.

## Documentation

- `docs/01_Ansible_A_to_Z_Execution_Guide.docx` — infrastructure, configuration, roles, playbook and verification.
- `docs/02_SSH_and_Linux_Command_Reference.docx` — commands and their purposes.
- `docs/03_Ansible_Troubleshooting_Log.docx` — observed issues and resolutions.

## Security notes

- Never commit plaintext passwords, AWS credentials, PEM/PPK private keys, or unencrypted secret files.
- Keep the Vault password separate from the encrypted Vault file.
- Restrict security-group rules to required sources and ports. Do not leave `All TCP` open to `0.0.0.0/0`.
- Self-signed WinRM certificates and disabled certificate validation are lab-only; use trusted certificates and validation in production.
- Replace lab IP addresses and review inventory before reuse.

## Learning outcomes

Ansible inventory and configuration, Linux SSH, Windows WinRM HTTPS, reusable roles, OS-specific `when` conditions, Chocolatey, Ansible Vault, single-playbook deployment and idempotency.
