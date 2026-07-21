# Ansible Role: OPSI Client ([Ludus](https://ludus.cloud))

> [!CAUTION]
> **Lab use only.** Companion to the [`ludus_opsi`](../ludus_opsi) server role.
> Credentials are passed on the command line (install task is `no_log`),
> certificate verification is disabled, and the installer runs with admin
> privileges. Fine in a throwaway Ludus range; not suitable for production.

Dual-purpose Windows companion role for `ludus_opsi`. Apply it to every Windows
VM in the range:

- **DC** (`opsi_client_dc: true`) — creates the `opsiadmin` AD group + a DNS A
  record for the OPSI server. No extra Ansible collection needed (plain PowerShell).
- **Member VMs** (`opsi_client_dc: false`, default) — pulls `opsi-client-agent-installer.exe`
  from the configserver and installs it silently, then reboots.

## Requirements

- `ansible.windows` collection on the Ludus server.
- The OPSI server must have the `opsi-client-agent` product in its depot
  (`ludus_opsi` installs it automatically via `ludus_opsi_install_base_products`).
- Client VMs must use `depends_on` in the range config to wait for the server
  role to finish before the download is attempted (see example below).

## DC mode (`opsi_client_dc: true`)

Runs on the primary DC. Does two things:

1. Creates the `opsiadmin` security group in AD (idempotent — skips if exists)
   and adds the specified members.
2. Creates a DNS A record so domain-joined clients can resolve the OPSI server
   by hostname (required for opsiclientd TLS cert validation — the cert has the
   hostname as CN, not the IP).

```yaml
opsi_client_dc: true
opsi_client_ad_group: "opsiadmin"              # must match ludus_opsi_ad_login_admin_group
opsi_client_ad_group_scope: "Global"
opsi_client_ad_group_members:
  - domainadmin                                # accounts that get OPSI admin
opsi_client_dns_zone: "ludus.domain"
opsi_client_dns_record_name: "opsi"            # creates opsi.ludus.domain
opsi_client_dns_record_ip: "10.16.10.20"       # OPSI server IP - set in role_vars
```

Both operations are idempotent. The group is only created if absent; the DNS
record is only added if the name doesn't already exist.

## Client mode (`opsi_client_dc: false`, default)

Runs on Windows member VMs. Downloads and silently installs the agent, then
reboots. After reboot, `win_service` ensures `opsiclientd` is running.

```yaml
opsi_client_dc: false
opsi_client_configserver: "https://opsi.ludus.domain:4447"   # hostname, NOT IP
opsi_client_service_user: "adminuser"
opsi_client_service_password: "LudusOpsiAdmin1"              # must match ludus_opsi_admin_password
opsi_client_id: ""                                           # empty = auto-detect from FQDN
opsi_client_force_reinstall: false
opsi_client_enable_wmi_firewall: false
```

> [!IMPORTANT]
> **Use the hostname, not the IP for `opsi_client_configserver`.**
> opsiclientd verifies TLS by default (`verify_server_cert = true`). The
> self-signed cert has the server hostname as CN but not the IP as a SAN —
> connecting by IP causes `certificate is not valid for '10.x.x.x'` and
> opsiclientd fails to connect to the messagebus. The DC mode creates the DNS
> record automatically so domain-joined clients resolve it.

> [!IMPORTANT]
> **No `!` in `opsi_client_service_password`.** The password must match
> `ludus_opsi_admin_password` exactly. The OPSI server's init silently drops
> `!` during Docker Compose env var processing. Use alphanumeric passwords only.

## Verified installer flags (opsi-client-agent 4.3.x)

The role uses these flags (verified against the installer `--help`):

```
--no-gui
--non-interactive
--service-address <configserver>
--service-username <user>
--service-password <password>
--finalize noreboot
```

`--finalize noreboot` is the correct value — `start_service` is not valid in
4.3.x and causes the installer to fall back to interactive/GUI mode, hanging
the Ansible task indefinitely. The role adds `async: 120 / poll: 10` as a
timeout backstop so a bad invocation fails fast rather than blocking the deploy.

## Why pull and not server-side push

`opsi-deploy-client-agent` (the server-side push tool) uses DCOM/WMI (RPC port
135 + dynamic range) in addition to SMB. In a standard Ludus range the Windows
client firewall blocks DCOM, causing `0x800706ba` (DCOM session error). The
pull install only needs outbound HTTPS (port 4447), which works with no
firewall changes. Set `opsi_client_enable_wmi_firewall: true` only if you
specifically want to test server-side push.

## Example Ludus Range Config

```yaml
ludus:
  # DC - creates the opsiadmin group + DNS A record
  - vm_name: "{{ range_id }}-DC01-2022"
    hostname: "{{ range_id }}-DC01-2022"
    template: win2022-server-x64-template
    vlan: 10
    ip_last_octet: 11
    ram_gb: 6
    cpus: 4
    windows:
      sysprep: true
    domain:
      fqdn: ludus.domain
      role: primary-dc
    dns_rewrites:
      - dc01.lab.test
    roles:
      - mojeda101.ludus_opsi_client
    role_vars:
      opsi_client_dc: true
      opsi_client_ad_group_members: ["domainadmin"]
      opsi_client_dns_record_ip: "10.16.10.20"

  # Windows client - installs the agent after the server is ready
  - vm_name: "{{ range_id }}-win11"
    hostname: "{{ range_id }}-win11"
    template: win11-22h2-x64-enterprise-template
    vlan: 10
    ip_last_octet: 21
    ram_gb: 4
    cpus: 2
    windows:
      sysprep: true
    domain:
      fqdn: ludus.domain
      role: member
    roles:
      - name: mojeda101.ludus_opsi_client
        depends_on:
          - vm_name: "{{ range_id }}-opsi"
            role: mojeda101.ludus_opsi
    role_vars:
      opsi_client_configserver: "https://opsi.ludus.domain:4447"
      opsi_client_service_password: "LudusOpsiAdmin1"
```

The `depends_on` is required — without it the client role may run before the
OPSI stack is up and the installer download fails with `Unable to connect`.

## All Role Variables

```yaml
# Mode
opsi_client_dc: false

# Client mode
opsi_client_configserver: "https://10.16.10.20:4447"
opsi_client_service_user: "adminuser"
opsi_client_service_password: ""           # REQUIRED - alphanumeric only
opsi_client_id: ""                         # empty = auto-detect from FQDN
opsi_client_installer_url: "{{ opsi_client_configserver }}/public/opsi-client-agent/opsi-client-agent-installer.exe"
opsi_client_force_reinstall: false
opsi_client_enable_wmi_firewall: false     # only for server-side push testing

# DC mode
opsi_client_ad_group: "opsiadmin"
opsi_client_ad_group_scope: "Global"
opsi_client_ad_group_members: [domainadmin]
opsi_client_dns_zone: "ludus.domain"
opsi_client_dns_record_name: "opsi"
opsi_client_dns_record_ip: ""              # set to OPSI server IP in role_vars
```

## Post-deploy verification

```bash
CC='docker compose -f /opt/opsi-server/docker-compose.yml exec -T opsi-server'

# clients enrolled and phoning home?
$CC opsi-cli jsonrpc execute host_getObjects '[]' '{"type":"OpsiClient"}' \
  | grep -iE '"id"|lastSeen|ipAddress'

# clients on the messagebus (connected = opsiclientd running)?
# check the opsiconfd admin page: https://<server>:4447/admin
# "Clients connected to messagebus" lists active agents

# push a product:
$CC opsi-cli client-action --clients mom-win11.ludus.domain \
  set-action-request --products hwaudit
$CC opsi-cli client-action --clients mom-win11.ludus.domain process-actions
```

## License

MIT.

## Author Information

Created by [mojeda101](https://github.com/mojeda101) for [Ludus](https://ludus.cloud/).
