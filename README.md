# [Ansible role vault](#vault)

description

|GitHub|Downloads|Version|
|------|---------|-------|
|[![github](https://github.com/mullholland/ansible-role-vault/actions/workflows/molecule.yml/badge.svg)](https://github.com/mullholland/ansible-role-vault/actions/workflows/molecule.yml)|[![downloads](https://img.shields.io/ansible/role/d/mullholland/vault)](https://galaxy.ansible.com/mullholland/vault)|[![Version](https://img.shields.io/github/release/mullholland/ansible-role-vault.svg)](https://github.com/mullholland/ansible-role-vault/releases/)|
## [Example Playbook](#example-playbook)

This example is taken from [`molecule/default/converge.yml`](https://github.com/mullholland/ansible-role-vault/blob/master/molecule/default/converge.yml) and is tested on each push, pull request and release.

```yaml
---
- name: Converge
  hosts: all
  become: true
  gather_facts: true
  vars:
    # no ssl certificates for testing
    vault_listeners:
      - name: tcp
        address: "127.0.0.1:8200"
        cluster_address: "127.0.0.1:8201"
        tls_disable: "true"
        unauthenticated_metrics_access: "true"
    # show more info for testing
    vault_show_unseal_information: true
    vault_store_root_token: true
  roles:
    - role: "mullholland.vault"
```

The machine needs to be prepared. In CI this is done using [`molecule/default/prepare.yml`](https://github.com/mullholland/ansible-role-vault/blob/master/molecule/default/prepare.yml):

```yaml
---
- name: Prepare
  hosts: all
  become: true
  gather_facts: true

  tasks:
    - name: Debian/Ubuntu | Install support packages for testing
      package:
        name:
          - "iproute2"  # for ansible_default_ipv4.address
          - "cron"  # for backup cron
          - "findutils"  # for backup cron
        state: latest
      when: ansible_os_family == "Debian"

    - name: RedHat/CentOS | Install support packages for testing
      package:
        name:
          - "iproute"  # for ansible_default_ipv4.address
          - "cronie"  # for backup cron
          - "findutils"  # for backup cron
        state: latest
      when: ansible_os_family == "RedHat" or ansible_os_family == "Rocky"

  roles:
    - role: mullholland.repository_hashicorp
```



## [Role Variables](#role-variables)

The default values for the variables are set in [`defaults/main.yml`](https://github.com/mullholland/ansible-role-vault/blob/master/defaults/main.yml):

```yaml
---
# ---------------------------------------------------------------------------
# Core variables
# ---------------------------------------------------------------------------

# Packages
vault_packages:
  - vault
  # - vault-enterprise

# System user and group (from package)
vault_user: "vault"
vault_group: "vault"

# Paths
vault_bin_path: "/usr/bin"
vault_config_path: "/etc/vault.d"
vault_plugin_path: "/usr/local/lib/vault/plugins"
vault_data_path: "/opt/vault/data"
vault_backup_path: "/opt/vault/backup"
vault_log_path: "/var/log/vault"
vault_run_path: "/var/run/vault"

# Telemetry
vault_telemetry_enabled: false
# metrics
vault_unauthenticated_metrics_access: false

# ---------------------------------------------------------------------------
# config general
# ---------------------------------------------------------------------------
# Enable the default webUI
vault_ui: "true"
# To reduce disk io, mlock can be disabled.
# https://www.vaultproject.io/docs/configuration/storage/raft
vault_disable_mlock: "true"
# Normally a bad idea
vault_disable_cache: "false"

# Configure some general parameters
# lease duration for tokens and secrets
vault_max_lease_ttl: "10h"
vault_default_lease_ttl: "10h"

# ---------------------------------------------------------------------------
# config cluster
# ---------------------------------------------------------------------------

# Configure clustering.
vault_disable_clustering: "false"

# The leader to use, please use a fqdn, i.e. `vault.example.com`
# This variable is not required for single-node installations, where the
# variable `vault_disable_clustering` is set to `"True"`.
# vault_leader: centos-7

# The URL where cluster members can find the leader.
vault_cluster_addr: "http://{{ vault_leader | default('localhost') }}:8201"

# The URL where the API will be served. This is the API of a local instance.
vault_api_addr: "http://127.0.0.1:8200"

# Specifies the identifier for the Vault cluster.
vault_cluster_name: "Vault Cluster"

# ---------------------------------------------------------------------------
# Seal transit
# ---------------------------------------------------------------------------
vault_transit: false
vault_transit_protocol: "http"
vault_transit_server: "{{ vault_cluster_addr }}"
vault_transit_port: "8200"

# ---------------------------------------------------------------------------
# config listeners
# ---------------------------------------------------------------------------
vault_listeners:
  - name: tcp
    address: "127.0.0.1:8200"
    cluster_address: "127.0.0.1:8201"
    tls_disable: "true"  # must be true if key/cert are missing
    tls_cert_file: "fullchain.pem"  # optional
    tls_key_file: "privkey.pem"  # optional
    unauthenticated_metrics_access: "true"  # optional

# ---------------------------------------------------------------------------
# config storage
# ---------------------------------------------------------------------------
# The storage backend(s) to configure.
# https://www.vaultproject.io/docs/configuration/storage/raft
vault_storages:
  - name: raft
    path: "{{ vault_data_path }}"
    node_id: "{{ inventory_hostname }}"

# ---------------------------------------------------------------------------
# Enterprise Features
# ---------------------------------------------------------------------------
# The license is required for Vault enterprise. You can use a trail license:
# https://www.hashicorp.com/products/vault/trial
# vault_license: ""

# ---------------------------------------------------------------------------
# Seals
# ---------------------------------------------------------------------------
# Amount of unseal keys to create
vault_key_shares: 5
# Amount of unseal keys to unseal
vault_key_threshold: 3
# If you want to see the (sensitive) output of `vault operator init`, set
# this parameter to `true`
vault_show_unseal_information: false
# You can store the root token in a file to make using Vault easier.
vault_store_root_token: false

# You can unseal vault using unseal keys that are known.
# For new installations, they will be created.
# vault_unseal_keys:
#   - KeY-oNe
#   - KeY-tWo
#   - KeY-tHrEe

# ---------------------------------------------------------------------------
# Telemetry
# ---------------------------------------------------------------------------
# https://www.vaultproject.io/docs/configuration/telemetry
# Enable Telemetry
vault_telemetry_disable: false
vault_telemetry:
  # https://www.vaultproject.io/docs/configuration/telemetry#prometheus
  prometheus_retention_time: '"30s"'
  disable_hostname: "true"
  # statsite_address: "statsite.company.local:8125"
  # statsd_address: "statsd.company.local:8125"

# ---------------------------------------------------------------------------
# Backup
# ---------------------------------------------------------------------------

# For this to work you need a vaitl token in /root/.vault-token
# you can use the option `vault_store_root_token` from this playbook
# which should be OK for testing or at home.

vault_backup_disable: false
vault_backup:
  cron:
    min: "0,30"
    hour: "*"
    day: "*"
  config:
    directory: "{{ vault_backup_path }}"
    logfile: "{{ vault_log_path }}/backup.log"
    retention: "3"  # in days
    script:
      # pre: 'curl -fsS -m 10 --retry 5 -o /dev/null https://hc-ping.com/XXXX-XXXX-XXXX-XXXX/start'
      pre: ""
      # post: 'scp {{ vault_backup_path }}/ /NFS/PATH'
      post: ""
      # error: 'echo "Vault Backup error" | mail -s subject user@gmail.com'
      error: ""
      # success: 'curl -fsS -m 10 --retry 5 -o /dev/null https://hc-ping.com/XXXX-XXXX-XXXX-XXXX'
      success: ""
```

## [Requirements](#requirements)

- pip packages listed in [requirements.txt](https://github.com/mullholland/ansible-role-vault/blob/master/requirements.txt).

## [State of used roles](#state-of-used-roles)

The following roles are used to prepare a system. You can prepare your system in another way.

| Requirement | GitHub | GitLab |
|-------------|--------|--------|
|[mullholland.repository_hashicorp](https://galaxy.ansible.com/mullholland/repository_hashicorp)|[![Build Status GitHub](https://github.com/mullholland/ansible-role-repository_hashicorp/workflows/Ansible%20Molecule/badge.svg)](https://github.com/mullholland/ansible-role-repository_hashicorp/actions)|[![Build Status GitLab](https://gitlab.com/opensourceunicorn/ansible-role-repository_hashicorp/badges/master/pipeline.svg)](https://gitlab.com/opensourceunicorn/ansible-role-repository_hashicorp)|

## [Context](#context)

This role is a part of many compatible roles. Have a look at [the documentation of these roles](https://mullholland.net) for further information.

Here is an overview of related roles:
![dependencies](https://raw.githubusercontent.com/mullholland/ansible-role-vault/png/requirements.png "Dependencies")

## [Compatibility](#compatibility)

This role has been tested on these [container images](https://hub.docker.com/u/mullholland):

|container|tags|
|---------|----|
|[EL](https://hub.docker.com/r/mullholland/enterpriselinux)|all|
|[Amazon](https://hub.docker.com/r/mullholland/amazonlinux)|Candidate|
|[Fedora](https://hub.docker.com/r/mullholland/fedora/)|all|
|[Ubuntu](https://hub.docker.com/r/mullholland/ubuntu)|all|
|[Debian](https://hub.docker.com/r/mullholland/debian)|all|

The minimum version of Ansible required is 2.10, tests have been done to:

- The previous version.
- The current version.
- The development version.

If you find issues, please register them in [GitHub](https://github.com/mullholland/ansible-role-vault/issues).

## [License](#license)

[MIT](https://github.com/mullholland/ansible-role-vault/blob/master/LICENSE).

## [Author Information](#author-information)

[Mullholland](https://mullholland.net)
