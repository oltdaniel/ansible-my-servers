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
# First execution (we assume you configured your ssh key via the hositng provider already)
# NOTE: Change the remote user in case it isn't root.
ansible-playbook genisis.yml -u root
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
networking_ufw_rules: [] # default
#   - { rule: "allow", port: 19999, proto: "any", from_ip: "{{ wireguard_network }}" } # allow Netdata portal access from wireguard network
#   - { rule: "allow", port: 80, proto: "tcp" } # allow HTTP
#   - { rule: "allow", port: 443, proto: "tcp" } # allow HTTPS
#   - { rule: "allow", port: 443, proto: "udp" } # allow HTTP/2+
```
With this variable custom rules can be applied. Currently only the above properties are support.

### `security`

See [geerlingguy/ansible-role-security](https://github.com/geerlingguy/ansible-role-security) for the base of the role and its options.

### `postgresql`

See [geerlingguy/ansible-role-postgresql](https://github.com/geerlingguy/ansible-role-postgresql) for the base of the role and its options.

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
netdata_enable_ufw_rules: true
```
This option defines whether to apply the required UFW rules or not. In this setup it is recommended as we deploy UFW and this option will automatically apply the necessary rules for the primary and secondary netdata communcation. This will not publish the netdata interface to the public.

### `wireguard`

> These aren't allo variables that can be used, see [the role defaults](./roles/wireguard/defaults/main.yml) for more options.

```yml
# Wireguard settings
wireguard_network: 10.0.0.0/24 # default
wireguard_address: 10.0.0.1 # default
```
By default, WireGuard will be configured in such a way, that it uses the above mentioned network settings. You can easily overwrite `wireguard_address` for each host individually (see `example.hosts.ini`).

```yml
# Custom WireGuard profiles
wireguard_export_custom_profiles: true # default: false
wireguard_custom_profiles_directory: wireguard # default
wireguard_custom_profiles: # default: []
  - name: desktop                 # NOTE: this will also be the filename
    address: 10.0.0.100
    private_key: REDACTED         # optional
    public_key: thisisntavalidkey
```
When `wireguard_export_custom_profiles` is set to true, the playbook will generate a config profile and save it into the directory configured in `wireguard_custom_profiles_directory`. The custom profile option `private_key` is optional (should be left empty when not using ansible vault), and the placeholder value `REDCATED` will be put in the config profile instead and hsould be replaced with the correct private key before importing it into the client.

```yml
# Default WireGuard interface name
wireguard_interface: wg0

# Allows config to disable server
wireguard_disable_server: false

# Default port that the WireGuard server will listen on
wireguard_listen_port: 51820
```
These are some more advanced options which allow you to change the interface name, disable listening and thereby the server function and also change the port used in the WireGuard environment.

> After this role has been executed and you connect to the VPN via your custom profile, you can use the `wireguard_address` of each host as the `ansible_host` address and configure everything else to communicate only via the WireGuard VPN in a secure tunnel.

## Tested on

- Debian 12

## License

[MIT License](./LICENSE)