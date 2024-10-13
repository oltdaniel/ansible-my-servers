# ansible-my-servers

These are the ansible playbooks I use to kind of replicate the setup across multiple servers I use for my stuff. This is mainly used for playing a bit around with ansible and do some more complex stuff.

> [!IMPORTANT]  
> These playbooks are currently only tested on Ubuntu 24.04 LTS. Only some functions work across multiple distros.

Thank you [@geerlingguy](https://github.com/geerlingguy) for providing the base roles I use here for my servers.

## Usage

```bash
# Clone the repo
git clone https://github.com/oltdaniel/ansible-servers
cd ansible-servers
# Install the requirements
ansible-galaxy install -r requirements.yml
# Copy the config files
cp example.hosts.ini hosts.ini
cp example.config.yml config.yml
#
# !! Edit the hosts.ini and config.yml to your needs !!
#
# First execution (we assume the default cloud server user root via your local ssh key)
ansible-playbook genisis.yml
# Deploy the software
ansible-playbook software.yml
# Upgrade the server and allow a reboot if required
ansible-playbook upgrade.yml --extra-vars "upgrade_allow_reboot=true"
```

## Playbooks

### `genisis.yml`

> [!IMPORTANT]  
> This role is intended for single execution with the `root` user. The root signin via SSH will be deactivated as a default setting via `geerlingguy.security`.

This is the main playbook that is also responsible to verify for the initial setup of things. By default it will use the remote user `root` as this is only meant to be executed once. After the first successful run, you should append `-u YOUR_GENISIS_USER` to the command.

### `software.yml`

This is the playbook to install the generic software that runs on the servers. Before installing the software it upgrades everything and applies the network configuration.

It currently deploys the following software:
- Docker
- PostgreSQL
- Netdata

### `upgrade.yml`

This is the playbook I use for upgrades. If a reboot is allowed if necessary, add `--extra-vars "upgrade_allow_reboot=true"` to the command.

## Roles

### `genisis`

```yml
genisis_user: "{{ lookup('env','USER') }}" # required
genisis_user_ssh_pubkey: ~/.ssh/id_ed25519.pub # default: ~/.ssh/id_ed25519.pub
```
This role is targeted for creating a user for yourself in order to interact with the system in other ways than the root user. The variable `genisis_user` is required and has no default value. With `genisis_user_ssh_pubkey` you can point to an ssh key that should be configured as an authorized key for the created user.

```yml
genisis_user_passwordless_sudoer: true # default: true
genisis_group_sudo_passwordless_sudoers: true # default: true
```

### `upgrade`

```yml
# Whether to allow a reboot after the upgrade if required.
upgrade_allow_reboot: no
```
This variable will be used to check if an reboot is allowed after an upgrade when necessary.

### `networking`

This role will install the `ufw` package, create an allow rule for OpenSSH and enable the firewall. Additionally, custom rules will be applied.

```yml
# UFW rules to apply
networking_ufw_rules: []
#   - { rule: "allow", port: 80, proto: "tcp" } # allow HTTP
#   - { rule: "allow", port: 443, proto: "tcp" } # allow HTTPS
#   - { rule: "allow", port: 443, proto: "udp" } # allow HTTP/2+
```
With this variable custom rules can be applied. Currently only the above properties are support.

### `postgresql`

See [geerlingguy/ansible-role-postgresql](https://github.com/geerlingguy/ansible-role-postgresql) for the base of the role and its options.

```yml
# Privileges to ensure exist.
postgresql_privs: # default: []
  - database: postgres
    type: group
    objs: pg_monitor
    roles: netdata
# - database: # required
#   grant_option: # default to make no change
#   objs: # default to make no change
#   privs: # defaults to not set
#   roles: # required
#   schema: # default public
#   state: # default present
#   type: # default table
#   login_host: # defaults to 'localhost'
#   login_password: # defaults to not set
#   login_user: # defaults to '{{ postgresql_user }}'
#   login_unix_socket: # defaults to 1st of postgresql_unix_socket_directories
#   port: # defaults to not set

# Whether to output privilege data when managing privileges.
postgres_privs_no_log: yes
```
With these variables the [postgresql_privs](https://docs.ansible.com/ansible/latest/collections/community/postgresql/postgresql_privs_module.html) module can be used directly from the config. The example shows the granting of privileges for the netdata postgresql user that netdata can use to monitor the server.

### `netdata`

> These aren't allo variables that can be used, see [the role defaults](./roles/netdata/defaults/main.yml) for more options.

```yml
# Role can be undefined, 'primary' or 'secondary'
netdata_role: primary
```
If undefined, it will keep netdata in the default configuration. If primary, it will configure the installation to operate in primary mode. If secondary, it will disable the web interface and so on and connect to the configured primary nodes (one or more primary nodes required).

We suggest to set this variable as a host specific variable. An example for this can be seen in `example.hosts.ini`, or you choose another way of defining host specific values.

```yml
# Use for the API Key in stream.conf
netdata_api_key: "00000000-0000-0000-0000-000000000000" # optional, required for parent config (generate with uuidgen)
```
Optional, unless you configure the system as a primary node. You can generate a value with the command `uuidgen`.

```yml
# Change options in the health_alarm_notify.conf
netdata_health_alarm_notify:
  - option: SEND_NTFY
    value: "YES"
  - option: DEFAULT_RECIPIENT_NTFY
    value: "https://ntfy.sh/REPLACEME"
```
This variable allows you to change different options in the `health_alarm_notify.conf` that is used for sending notifications. This example shows a minimal ntfy.sh configuration.

```yml
# Configure the postgres collector
netdata_go_d_postgres:
  - name: local
    dsn: 'host=/var/run/postgresql dbname=postgres user=netdata'
```
This option allows you to configure the postgresql collector in `go.d/postgres.conf`.

```yml
# Configure the necessary UFW rules for netdata.
netdata_enable_ufw_rules: yes
```
This option defines whether to apply the required UFW rules or not. In this setup it is recommended as we deploy UFW and this option will automatically apply the necessary rules for the primary and secondary netdata communcation. This will not publish the netdata interface to the public.

## License

[MIT License](./LICENSE)