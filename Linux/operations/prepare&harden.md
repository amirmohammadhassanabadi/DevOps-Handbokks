# Rocky Linux Server Preparation & Hardening

## 1. Scope

This document provides a practical procedure for preparing and hardening a Rocky Linux server before it is used to host production workloads.

The goal is to establish a secure, stable, and maintainable baseline while keeping the configuration practical and avoiding unnecessary hardening that could reduce functionality or complicate administration.

### 1.1 In Scope

This document covers the preparation and hardening of the Rocky Linux operating system, including:

- Initial system assessment
- Operating system updates
- Hostname and basic system configuration
- Time synchronization
- Network configuration review
- Package management
- User and privilege management
- `sudo` configuration
- Password policy
- SSH hardening
- `firewalld` configuration
- SELinux configuration and verification
- Service hardening
- Filesystem and storage security
- Kernel and `sysctl` hardening
- Network-level kernel parameters
- System logging
- Audit configuration
- Security-related system settings
- Final security verification

### 1.2 Approach

Hardening will be performed incrementally.

For each configuration change:

1. Inspect the current state.
2. Understand the security purpose of the change.
3. Apply the configuration.
4. Validate the configuration.
5. Verify that the system remains functional.
6. Document the final configuration and verification procedure.

Security settings will not be applied blindly. Each hardening measure should have a clear security benefit and should be appropriate for the server's intended role.

### 1.3 Important Principle

The objective is not to make the server as restrictive as possible.

The objective is to establish a **secure and maintainable baseline** while preserving the functionality required by the server.

Where a security setting has compatibility, availability, or operational consequences, those consequences should be considered before applying the setting.

---

## 2. Prerequisites & Safety

Before making configuration changes, verify that the following prerequisites are satisfied.

The purpose of this section is to prevent accidental loss of access, unintended service interruption, or irreversible configuration changes during the hardening process.

### 2.1 Administrative Access

The initial preparation should be performed using an account with sufficient administrative privileges.

For a newly installed server, the initial configuration may be performed using `root`.

A dedicated administrative account should be created and tested before direct root SSH access is disabled.

### 2.2 Console Access

Whenever possible, maintain access to the server's virtualization or out-of-band console.

Examples include:

- KVM console
- Virtual machine console
- IPMI
- iDRAC
- iLO
- Provider web console

Console access provides a recovery path if SSH, networking, or firewall configuration prevents remote access.

> **Important:** Do not rely exclusively on SSH when performing changes that can affect SSH or networking.


### 2.3 Configuration Backup

Before modifying important configuration files, create a backup when appropriate.

Example:

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```
---

## 3. Initial Assessment

Before making any system changes, inspect the server and record its current state.

The initial assessment provides a baseline for the hardening process and helps prevent configuration changes from being made without understanding the existing environment.

> **Rule:** Do not modify the system during the initial assessment.

---

### 3.1 Operating System

Check the OS version, kernel, architecture, and virtualization environment.

```bash
cat /etc/os-release
uname -a
hostnamectl
```

#### Example:

```bash
Rocky Linux 10.x
x86_64
KVM
```

The OS version should be recorded because package repositories, security features, systemd behavior, SELinux policies, and hardening procedures can vary between major releases.

### 3.2 System Uptime and Load

Check the current system uptime and load:

```bash
uptime
```

A high load average on a supposedly fresh server should be investigated before continuing.

### 3.3 Network Interfaces

Inspect the current network interfaces and addresses:

```bash
ip -br addr
```

#### Review:

- Network interface names
- IPv4 addresses
- IPv6 addresses
- Interface state
- Loopback configuration

#### Example:
```bash
lo       UNKNOWN  127.0.0.1/8
ens18    UP       22.17.145.77/24
```
Do not assume that the primary interface is named eth0. Modern Rocky Linux installations commonly use predictable interface names such as ens18, ens160, or similar names.

### 3.4 Routing

Inspect the routing table:

```bash
ip route
```

#### Review:

- Default gateway
- Connected networks
- Network interfaces
- Routing anomalies

#### Example:

```bash
default via <gateway> dev ens18
22.17.145.0/24 dev ens18
```

The default route must be known before changing network or firewall configuration.

### 3.5 DNS Configuration

Inspect the resolver configuration:

```bash
cat /etc/resolv.conf
```

Also determine whether the file is managed by NetworkManager:

```bash
ls -l /etc/resolv.conf
```

Review:

- Nameservers
- Search domains
- Configuration management
- Whether DNS configuration is static or dynamically generated

Do not manually edit /etc/resolv.conf if it is managed by NetworkManager or another service. DNS configuration should be changed through the appropriate network configuration mechanism.

### 3.6 Storage and Filesystems

Inspect block devices and filesystems:

```bash
lsblk -f
lsblk
```

Then inspect filesystem usage:

```bash
df -hT
df -ih
```

#### Review:

- Disk layout
- Partition sizes
- Filesystem types
- Mount points
- Swap
- Available disk space
- Inode usage

Pay particular attention to:

- `/`
- `/boot`
- `/var`
- `/tmp`
- `/home`

### 3.7 CPU and Memory

Check CPU and memory resources:

```bash
nproc
free -h
```

#### Review:

- Number of CPUs
- Total memory
- Available memory
- Swap size
- Swap usage

### 3.8 Logged-in Users

Check currently logged-in users:

```bash
who
w
```

#### Review:

- Active local sessions
- Active SSH sessions
- Remote source addresses
- Unexpected users or sessions


### 3.9 Local User Accounts

List local accounts:
```bash
getent passwd
```

To identify accounts with interactive login shells:

```bash
getent passwd | awk -F: '$7 !~ /(nologin|false)$/ {print $1 ":" $7}'
```

#### Review:

- Root account
- Administrative accounts
- Service accounts
- Interactive login shells
- Unexpected users

Do not remove system accounts simply because they are unfamiliar. Many are required by installed system components.

### 3.10 Administrative Groups

Check the wheel group:

```bash
getent group wheel
```

On Rocky Linux, `wheel` is commonly used for administrative privileges through `sudo`.


### 3.11 SSH Configuration

Inspect the effective SSH configuration:

```bash
sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication)'
```

Review at minimum:

- SSH port
- Root login
- Password authentication
- Public-key authentication

#### Example baseline:

- port 22
- permitrootlogin yes
- passwordauthentication yes
- pubkeyauthentication yes

The effective configuration should be checked with `sshd -T` rather than relying only on the contents of `/etc/ssh/sshd_config`.

### 3.12 Firewall Status

Check whether firewalld is installed, enabled, and running:

```bash
systemctl status firewalld --no-pager
firewall-cmd --state
```

Then inspect the active firewall configuration:

```bash
firewall-cmd --get-active-zones
firewall-cmd --get-default-zone
```

#### Review:

- Firewall service status
- Active zones
- Default zone
- Network interfaces assigned to zones
- Allowed services
- Open ports

The firewall should be understood before modifying its rules.

### 3.13 SELinux Status

Check SELinux:

```bash
getenforce
sestatus
```

#### Review:

- Whether SELinux is enabled
- Current enforcement mode
- Configured mode
- Active policy
- Policy type

Expected secure baseline:

```bash
SELinux status: enabled
Current mode: enforcing
```

SELinux should normally remain enabled and enforcing. Do not disable SELinux during the initial assessment.

### 3.14 Listening Network Services

Inspect all listening sockets:

```bash
ss -tulnp
```


For every remotely accessible service, determine:

- What process owns the port?
- Why is it listening?
- Does the server require it?
- Should it be exposed to the network?
- Should the firewall allow access to it?

#### Example:

```
TCP 0.0.0.0:22    sshd
TCP [::]:22       sshd
```

A listening port is not automatically a security problem. The important question is whether the service is required and appropriately protected.

### 3.15 Running Services

List currently running services:

```bash 
systemctl --type=service --state=running --no-pager
```

Also inspect enabled services:

```bash
systemctl list-unit-files --type=service --state=enabled --no-pager
```
### 3.16 Failed Systemd Units

Check for failed services:

```bash
systemctl --failed
```

Expected result on a healthy fresh installation:

> 0 loaded units listed.

If failed units are present, investigate them before continuing.

---

```
rocky-linux-server-hardening.md
│
├── 1. Scope
├── 2. Prerequisites & Safety
├── 3. Initial Assessment
│
├── 4. System Preparation
│   ├── OS Updates
│   ├── Hostname
│   ├── Time Synchronization
│   ├── DNS / Network
│   └── Package Management
│
├── 5. User & Privilege Hardening
│   ├── Administrative User
│   ├── Sudo
│   ├── Root Account
│   └── Password Policy
│
├── 6. SSH Hardening
│   ├── SSH Keys
│   ├── sshd Configuration
│   ├── Root Login
│   ├── Password Authentication
│   └── Verification
│
├── 7. Firewall Hardening
│   ├── firewalld
│   ├── Zones
│   ├── Allowed Services
│   └── Verification
│
├── 8. SELinux
│   ├── Status
│   ├── Configuration
│   ├── Contexts
│   └── Troubleshooting
│
├── 9. Service Hardening
│   ├── Running Services
│   ├── Enabled Services
│   └── Unnecessary Services
│
├── 10. Filesystem & Storage
│   ├── Permissions
│   ├── Mounts
│   ├── /tmp
│   ├── /var
│   └── Disk Usage
│
├── 11. Kernel & Network Hardening
│   ├── sysctl
│   ├── IPv4
│   ├── IPv6
│   └── Network Parameters
│
├── 12. Logging & Auditing
│   ├── journald
│   ├── rsyslog
│   ├── auditd
│   └── Log Retention
│
├── 13. Security Controls
│   ├── Automatic Updates
│   ├── File Integrity
│   ├── Login Monitoring
│   └── Vulnerability Checks
│
├── 14. Final Verification
│   ├── SSH
│   ├── Firewall
│   ├── SELinux
│   ├── Services
│   ├── Listening Ports
│   └── Updates
│
└── 15. Final Checklist
```