# Hardened Server (Ansible)

An Ansible project that turns a fresh Ubuntu Server into a hardened, reachable-only-over-VPN host with a single playbook run.

It started as a hands-on exercise: first I hardened a Linux VM by hand, then I rewrote every step as an idempotent Ansible role so the same result can be reproduced on any clean machine, without manual edits.

<p align="center">
  <img src="docs/img/demo.gif" alt="One playbook run configures a clean server" width="800">
</p>
<p align="center"><em>One command turns a clean Ubuntu Server into a hardened host.</em></p>

## What it does

Running `site.yml` against a clean Ubuntu Server:

- creates a dedicated automation account (key-only login, no password) with its own scoped sudo rule, validated with `visudo` before it is written;
- joins the host to a Tailscale network using a short-lived auth key stored in Ansible Vault;
- serves a minimal site through nginx with a hardened configuration;
- protects SSH with fail2ban;
- locks SSH down (no root login, no password authentication) and refuses to apply it if no working key is in place;
- enables a firewall that only accepts management traffic over the Tailscale interface.

## Roles

| Role | Purpose |
|---|---|
| `base` | Automation user, SSH key, sudoers drop-in, base packages, timezone, safe upgrades |
| `tailscale` | Installs Tailscale and connects the host to the tailnet |
| `nginx` | Installs nginx and applies a hardened configuration |
| `fail2ban` | Brute-force protection for SSH, with the Tailscale range whitelisted |
| `ssh_hardening` | Disables root and password login via an sshd drop-in config |
| `ufw` | Default-deny firewall, management access over `tailscale0` only |

## Design decisions

**Order matters.** Every step closes something, so each step first has to leave a working way back in:

`base` → `tailscale` → `nginx` → `fail2ban` → `ssh_hardening` → `ufw`

- A key and a user come first, otherwise disabling passwords locks you out.
- The VPN tunnel comes before the firewall, because the firewall only trusts that interface. Enabling it earlier would cut off the only path to the host.
- The firewall goes last for the same reason.

**A dedicated automation user instead of a shared password or blanket `NOPASSWD`.** Passwordless sudo exists for one account that can only log in with a key. The private key never leaves the control machine.

**Secrets stay out of the repository.** The Tailscale auth key lives in an Ansible Vault file that is not published. `vault.example.yml` and `hosts.example.yml` show the expected shape with placeholder values.

**Guard rails against self-lockout.** The SSH role contains a pre-check that refuses to disable password authentication when no authorised key is present. The sudoers file is validated before it is installed.

**Idempotent.** A second run against an already configured host reports no changes.

## Results

Hardening index reported by [Lynis](https://cisofy.com/lynis/) on an Ubuntu Server VM:

| | Hardening index |
|---|---|
| Fresh install | ~54 (measured manually, report not kept) |
| After `site.yml` | **73** |

Lynis reports are not included in the repository because they contain host-specific details.

### What is intentionally left open

Some remaining Lynis suggestions do not matter for this threat model:

- **GRUB boot password:** the host is a VM without physical access.
- **Password aging and hashing rounds:** password login is disabled, accounts authenticate with keys.
- **PAM password strength modules:** same reason.

## Repository layout

```
.
├── site.yml                 # main playbook, role order lives here
├── lynis.yml                # runs a Lynis audit and collects the report
├── ansible.cfg
├── requirements.yml
├── inventory/
│   ├── hosts.example.yml    # sample inventory with placeholder values
│   └── group_vars/all/vars.yml
├── vault.example.yml        # shape of the (unpublished) vault file
├── docs/img/demo.gif        # demo of a full run
└── roles/
    ├── base/
    ├── tailscale/
    ├── nginx/
    ├── fail2ban/
    ├── ssh_hardening/
    └── ufw/
```

## Tested on

Ubuntu Server VMs in VirtualBox, controlled from a NixOS workstation.

## Possible next steps

- `auditd` rules and log shipping to Wazuh;
- Vagrant to create and destroy test machines automatically;
- Identity layer (SSO/MFA) on top of the hardened hosts.
