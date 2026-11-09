# Ansible Role: cockpit

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-cockpit)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-cockpit)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-cockpit)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-cockpit/dev.yml?branch=dev&label=dev)](https://github.com/jomrr/ansible-role-cockpit/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-cockpit/main.yml?branch=main&label=main)](https://github.com/jomrr/ansible-role-cockpit/actions/workflows/main.yml?query=branch%3Amain)

Install Cockpit for system administration, journal filtering and service
management.

## Purpose

Install cockpit-ws and cockpit-system for browser-based journal filtering and
systemd service management. Select additional applications with
cockpit_packages. AlmaLinux, Fedora, openSUSE Leap and openSUSE Tumbleweed use
their distribution repositories. Debian and Ubuntu select packages with their
codename-specific backports suite enabled. Access the service at
`https://host.example.com:9090`.

## Scope

### Managed

- Selected packages and their required dependencies, without APT or Zypper
  recommendations or DNF weak dependencies.
- Debian backports repository and APT backports package selection during
  installation and updates.
- Authoritative /etc/cockpit/cockpit.conf and enabled, running cockpit.socket.

### Not Managed

- AD domain membership, SSSD, Kerberos keytabs, browser policies and account
  privileges.
- Firewall rules, TLS certificate provisioning, custom socket listeners and
  their SELinux port labels.
- Removal of previously installed Cockpit applications or the distribution's
  cockpit metapackage.

## Requirements

- Debian requires an available backports suite for its release and the
  distribution's debian-archive-keyring package.
- Ubuntu requires its release's backports suite and universe component enabled
  in the distribution repositories. The role uses those sources without adding
  Debian repositories.
- The Zypper package backend on openSUSE requires the community.general
  collection.
- Journal visibility and service administration depend on the logged-in
  account's journal, sudo or polkit permissions.

## Dependencies

```yaml
collections:
  - name: community.general
    version: '>=12.0.0'
  - name: containers.podman
    version: '>=1.18.0'
```

## Role Variables

### `cockpit_packages`

Type: `list`. Required: `false`.

Complete list of packages to install; cockpit-ws is required for the managed web
service.
Required dependencies are installed, recommendations are disabled and previously
installed applications are not removed.

Default:

```yaml
cockpit_packages:
  - cockpit-ws
  - cockpit-system
```

### `cockpit_package_state`

Type: `str`. Required: `false`.

Package installation policy; latest also upgrades installed APT packages to the
selected backports candidate when available.
Present retains already installed versions, even when a newer backports version
is available.

Default:

```yaml
cockpit_package_state: latest
```

### `cockpit_debian_backports_uri`

Type: `str`. Required: `false`.

Debian mirror supplying the distribution codename's backports suite, verified
with the Debian archive keyring.

Default:

```yaml
cockpit_debian_backports_uri: https://deb.debian.org/debian
```

### `cockpit_config`

Type: `dict`. Required: `false`.

Authoritative cockpit.conf sections mapping option names to string, boolean or
numeric values.
Defaults disable remote-host login, multi-host sessions and unencrypted access,
with at most ten concurrent login attempts.
Password sessions expire after fifteen idle minutes; Kerberos SSO is unaffected.
Remote-connection warnings remain enabled.
Empty uses upstream defaults; booleans render as lowercase true or false, and
multi-value options use space-separated strings.
Options must be supported by the installed Cockpit version; credentials do not
belong in this configuration.

Default:

```yaml
cockpit_config:
  WebService:
    LoginTo: false
    AllowMultiHost: false
    AllowUnencrypted: false
    MaxStartups: 10
  Session:
    IdleTimeout: 15
    WarnBeforeConnecting: true
```

## Managed Files

- `/etc/cockpit/cockpit.conf`
- `/etc/apt/sources.list.d/cockpit-backports.sources`

## Check Mode

Check mode predicts changes on configured hosts without applying them.

- On a fresh host, missing packages, repository metadata or systemd units can
  prevent a complete check-mode run.

## Service Behavior

Enable and start cockpit.socket. Restart cockpit.service after package or
configuration changes and apply handlers before returning. Unchanged runs do not
restart the service; package updates available with cockpit_package_state=latest
are intentional changes.

### Handlers

- Package and cockpit.conf updates restart the web service and can interrupt
  active sessions.

## Security Notes

- HTTPS uses Cockpit's certificate selection in /etc/cockpit/ws-certs.d.
  Provision a matching .cert and .key pair there with restricted key
  permissions; Cockpit creates a self-signed certificate when none is supplied.
  The alphabetically last certificate takes precedence.
- Defaults disable direct login to other hosts (LoginTo), multi-host sessions
  (AllowMultiHost) and unencrypted access (AllowUnencrypted). Cockpit retains
  its HTTP exceptions for localhost and /ping. Only enable multi-host sessions
  for mutually trusted hosts.
- MaxStartups=10 limits concurrent login attempts. Session.IdleTimeout=15
  expires inactive password sessions after fifteen minutes; it does not enforce
  logout for Kerberos SSO. WarnBeforeConnecting=true retains remote-connection
  warnings if remote access is enabled.
- ProtocolHeader and ForwardedForHeader are intended for a trusted reverse proxy
  that replaces incoming forwarding headers. Set Origins to the externally used
  HTTPS origin.

## Operational Notes

- AD SSO uses Cockpit's default Kerberos negotiation without an enable switch.
  The host must already be joined to AD, have working domain DNS and a matching
  FQDN, and resolve domain users to Unix accounts. The keytab must contain
  HOST/server.example.com@EXAMPLE.COM; Cockpit uses /etc/krb5.keytab or a
  dedicated /etc/cockpit/krb5.keytab.
- The browser needs a valid Kerberos ticket and permission to negotiate for the
  server's FQDN. Delegating credentials to further hosts requires additional
  domain and browser settings. SSO grants authentication, not administrative
  privileges. Session.IdleTimeout applies to password logins and does not
  provide a Kerberos logout boundary.
- cockpit_config accepts the installed version's cockpit.conf options.
  WebService includes Origins, ProtocolHeader, ForwardedForHeader, LoginTitle,
  LoginTo, RequireHost, AllowMultiHost, MaxStartups, MaxConcurrentConnections,
  AllowUnencrypted, UrlRoot, ClientCertAuthentication, Shell and
  CustomLoginPage. Session includes Banner, IdleTimeout and
  WarnBeforeConnecting. Log.Fatal and OAuth.URL, ErrorParam and TokenParam are
  also documented upstream. Newer options may be unavailable in distribution
  versions.
- OAuth settings alone do not implement AD SSO or token validation.
  Client-certificate authentication also needs its own identity infrastructure.
  Setting negotiate.action=none disables Kerberos; setting basic.action=none
  disables password login. Leave both unset to retain the standard
  authentication methods.
- Port and bind address belong in a cockpit.socket systemd drop-in, not
  cockpit.conf. Reset ListenStream before adding listeners, then reload systemd
  and restart the socket. Arrange firewall access and SELinux websm_port_t
  labels for nonstandard ports separately.
- The package list replaces the defaults, so retain cockpit-ws and normally
  cockpit-system. Optional applications include cockpit-storaged,
  cockpit-networkmanager, cockpit-packagekit, cockpit-podman and
  cockpit-machines where available in the selected distribution. Package
  availability is distribution-specific; required dependencies such as
  cockpit-bridge are installed automatically. No cockpit metapackage is
  requested by default.
- With Ansible's standard dictionary replacement, overriding cockpit_config
  replaces the complete mapping. Include all desired settings when customizing
  it, as shown below. An empty cockpit_config clears role-managed options and
  uses upstream defaults. Cockpit provides no standalone cockpit.conf validation
  command; confirm version-specific settings against the deployment guide and
  the running service.
- MaxConcurrentConnections remains at the installed version's default. Origins,
  proxy headers, OAuth and client-certificate authentication remain unset and
  require deployment-specific choices.

## Supported Platforms

| OS Family | Distribution | Version | Container Image |
| --------- | ------------ | ------- | --------------- |
| RedHat | AlmaLinux | latest | [jomrr/molecule-almalinux:latest](https://hub.docker.com/r/jomrr/molecule-almalinux) |
| Debian | Debian | latest | [jomrr/molecule-debian:latest](https://hub.docker.com/r/jomrr/molecule-debian) |
| RedHat | Fedora | latest | [jomrr/molecule-fedora:latest](https://hub.docker.com/r/jomrr/molecule-fedora) |
| Suse | OpenSuse Leap | latest | [jomrr/molecule-opensuse-leap:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-leap) |
| Suse | OpenSuse Tumbleweed | latest | [jomrr/molecule-opensuse-tumbleweed:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-tumbleweed) |
| Debian | Ubuntu | latest | [jomrr/molecule-ubuntu:latest](https://hub.docker.com/r/jomrr/molecule-ubuntu) |

## Example Playbook

### Minimal system console with AD SSO

Apply to hosts whose AD membership and account privileges are already configured.

```yaml
- name: Install Cockpit
  hosts: all
  gather_facts: true
  roles:
    - role: jomrr.cockpit
```

### Select applications and login options

Add storage management and a login title while retaining the defaults.

```yaml
- name: Configure Cockpit
  hosts: all
  gather_facts: true
  roles:
    - role: jomrr.cockpit
      cockpit_packages:
        - cockpit-ws
        - cockpit-system
        - cockpit-storaged
      cockpit_config:
        WebService:
          LoginTo: false
          AllowMultiHost: false
          AllowUnencrypted: false
          MaxStartups: 10
          LoginTitle: Server administration
        Session:
          IdleTimeout: 15
          WarnBeforeConnecting: true
```

## References

- [Deployment guide](https://docs.cockpit-project.org/cockpit-guide/latest/guide/index.html)
- [Installation](https://cockpit-project.org/running.html)
- [Configuration options](https://docs.cockpit-project.org/cockpit-guide/latest/man/cockpit.conf.5.html)
- [Kerberos SSO](https://docs.cockpit-project.org/cockpit-guide/latest/guide/sso.html)
- [TLS certificates](https://docs.cockpit-project.org/cockpit-guide/latest/guide/https.html)
- [Port and address](https://docs.cockpit-project.org/cockpit-guide/latest/guide/listen.html)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
