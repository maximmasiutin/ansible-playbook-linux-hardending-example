# Ansible Lab #1: Multi-Distro System Hardening

Vagrant/VirtualBox lab applying [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks/), [CCE](https://nvd.nist.gov/config/cce/index), and [C2S](https://aws.amazon.com/compliance/c2s-resources/) hardening controls across four Linux distributions via a single Ansible playbook.

## Target Hosts

| Inventory Group | Host   | Vagrant Box        | Private IP    | SSH Port | Pkg Manager | RAM  | CPUs |
|-----------------|--------|--------------------|---------------|----------|-------------|------|------|
| `debian`        | ubuntu | `ubuntu/focal64`   | 192.168.0.2   | 2001     | apt         | 2 GB | 2    |
| `redhat`        | centos | `centos/8`         | 192.168.0.3   | 2002     | yum/dnf     | 2 GB | 2    |
| `apk`           | alpine | `generic/alpine38` | 192.168.0.4   | 2003     | apk         | 2 GB | 2    |
| `portage`       | gentoo | `generic/gentoo`   | 192.168.0.5   | 2004     | emerge      | 2 GB | 2    |

All SSH ports forwarded to `127.0.0.1` only. Vagrant synced folders disabled.

## Usage

Prerequisites: Vagrant, VirtualBox, Ansible.

```bash
vagrant up                                                    # provision VMs
bash run_ansible.bash                                         # run hardening
# equivalent: ansible-playbook -i ./ansible-inventory -u vagrant hardening.yaml
bash run_molecule.bash                                        # molecule test (optional)
```

Molecule config: `project-role/molecule/default/molecule.yml` (Vagrant driver, converges `hardening.yaml`). Molecule gives Gentoo 4 GB / 4 CPUs.

## Playbook Design (`hardening.yaml`)

Single flat playbook, `hosts: all`, `become: yes`.

**Bootstrap phase:** `gather_facts: no` initially -- Alpine has no Python by default. Installs Python via `raw` module per distro (`apt`/`yum`/`apk`), then runs `gather_facts` and `package_facts`. Gentoo runs `emerge --changed-use --deep @world --backtrack=50`, installs `gentoolkit`, and runs `emerge --depclean` before fact gathering (this step is slow).

All package operations are conditional on `ansible_facts.available_packages` / `ansible_facts.packages`, making the playbook cross-distro safe.

## Applied Controls

### Timezone and NTP

- Timezone: `Europe/Chisinau` (with file-copy fallback if `community.general.timezone` fails)
- NTP: prefers `systemd-timesyncd`, falls back to `chrony` if unavailable. Server: `md.pool.ntp.org`
- Removes conflicting NTP packages before installing the chosen one
- Handles both chrony config paths (`/etc/chrony/chrony.conf` and `/etc/chrony.conf`)

### SSH Hardening (`/etc/ssh/sshd_config`)

All applied via `ansible.builtin.replace` against individual directives:

| CCE ID         | Directive              | Value                          |
|----------------|------------------------|--------------------------------|
| CCE-80645-5    | LogLevel               | INFO                           |
| CCE-27471-2    | PermitEmptyPasswords   | no                             |
| CCE-27082-7    | ClientAliveCountMax    | 0                              |
| CCE-27433-2    | ClientAliveInterval    | 300                            |
| CCE-27455-5    | MACs                   | hmac-sha2-512,hmac-sha2-256    |
| CCE-27320-1    | Protocol               | 2                              |
| CCE-27377-1    | IgnoreRhosts           | yes                            |
| CCE-80226-4    | X11Forwarding          | no                             |
| --             | GatewayPorts           | no                             |
| --             | PermitTunnel           | no                             |
| --             | PasswordAuthentication | no                             |
| --             | PermitRootLogin        | no                             |

SSHD restarted with `async: 60 / poll: 1` to avoid connection drops.

### Intrusion Prevention

`fail2ban` installed where available.

### GRUB Kernel Parameters

On all hosts except Alpine (`apk` group): `GRUB_CMDLINE_LINUX="ipv6.disable=1 audit=1 audit_backlog_limit=8192"`. Rebuilds GRUB via `update-grub` (Debian) or `grub2-mkconfig` (RedHat).

### RPM GPG Enforcement (RedHat only)

Sets `gpgcheck=1` in both `/etc/yum.conf` and `/etc/dnf/dnf.conf`.

### Automatic Security Updates

- RedHat: `dnf-automatic` with `apply_updates = yes`
- Debian: `unattended-upgrades` with auto-reboot, install-on-shutdown, verbose logging enabled

### Legacy Service and Socket Removal

| CCE ID      | Target                                        |
|-------------|-----------------------------------------------|
| CCE-27337-5 | `rsh.socket` stopped/disabled                 |
| CCE-27336-7 | `rlogin.socket` stopped/disabled              |
| CCE-27408-4 | `rexec.socket` stopped/disabled               |
| CCE-27406-8 | `/etc/hosts.equiv`, `~/.rhosts` deleted       |
| --          | `rsh`, `rsh-client`, `rsh-server` removed     |

### Package Removal

Packages removed if present (conditional on `ansible_facts.available_packages`):

| CCE ID      | Packages                                                         |
|-------------|------------------------------------------------------------------|
| --          | `autofs`                                                         |
| CCE-27399-5 | `ypserv`                                                         |
| CCE-27432-4 | `talk`                                                           |
| CCE-27210-4 | `talk-server`                                                    |
| CCE-27218-7 | `xorg-x11-server-common`                                        |
| CCE-80293-4 | `openldap-servers`, `openldap`                                   |
| CCE-80338-7 | `avahi-daemon`, `avahi`                                          |
| CCE-80274-4 | `snmpd`, `net-snmp`                                              |
| CCE-80325-4 | `named`                                                          |
| CCE-80277-7 | `smb`                                                            |
| CCE-80300-7 | `httpd`, `httpd-devel`, `micro-httpd`, `mini-httpd`              |
| --          | `nginx`, `nginx-common`, `nginx-core`, `nginx-full`, `nginx-light` |
| CCE-80269-4 | `rhnsd`                                                          |
| CCE-80285-0 | `squid`, `squid-common`                                          |
| CCE-80330-4 | `dhcpd`, `udhcpd`                                                |
| CCE-80294-2 | `dovecot`, `dovecot-core`, `dovecot-dev`                         |
| CCE-80230-6 | `rpcbind`                                                        |
| CCE-80237-1 | `nfs`, `nfs-common`, `nfs-utils`                                 |
| CCE-80282-7 | `cups`, `cups-daemon`, `cups-server-common`                      |

### Compliance Banner

Standard legal usage warning written to `/etc/motd`.

### Full System Update

Final step: `yum -y update` (RedHat), `apt-get -y dist-upgrade` (Debian), `apk upgrade` (Alpine).

## File Layout

```
hardening.yaml                          # main playbook (all controls)
Vagrantfile                             # 4 VMs, private network, localhost-only SSH
ansible-inventory/hosts                 # static inventory (4 groups)
project-role/molecule/default/molecule.yml  # Molecule config (Vagrant driver)
run_ansible.bash                        # wrapper: ansible-playbook invocation
run_molecule.bash                       # wrapper: molecule converge
```
