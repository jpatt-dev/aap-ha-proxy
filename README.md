# AAP HAProxy / Keepalived

## Purpose

This repository configures two RHEL HAProxy/Keepalived nodes in front of Ansible Automation Platform 2.6.

## Architecture

```
User/DNS -> VIP -> HAProxy pair -> AAP gateway/backend
```

- Keepalived owns the floating VIP between the two HAProxy nodes.
- HAProxy terminates client TLS and forwards HTTPS to the AAP gateway/backend.
- The AAP backend/controller host is not managed by this playbook.

## AAP Inventory Model

The selected AAP inventory must contain only the HAProxy/Keepalived nodes because the playbook targets `hosts: all`.

Example inventory hosts:

- `192.168.4.51`
- `192.168.4.52`

Do not include the AAP backend/controller host in this inventory.

## Required AAP Host Variables

Set these per node in AAP inventory host variables. These are host-specific values and should not be survey variables.

Primary node:

```yaml
ansible_host: 192.168.4.51
keepalived_state: MASTER
keepalived_priority: 101
keepalived_unicast_src_ip: 192.168.4.51
keepalived_unicast_peers:
  - 192.168.4.52
```

Secondary node:

```yaml
ansible_host: 192.168.4.52
keepalived_state: BACKUP
keepalived_priority: 100
keepalived_unicast_src_ip: 192.168.4.52
keepalived_unicast_peers:
  - 192.168.4.51
```

## Required Survey / Extra Vars

Reference:

- [aap/survey_spec.yml](aap/survey_spec.yml)
- [aap/extra_vars.example.yml](aap/extra_vars.example.yml)

```yaml
keepalived_auth_pass: labpass1

haproxy_manage_tls_files: false
haproxy_manage_selinux: false
haproxy_ssl_pem_path: /etc/haproxy/certs/aap-gateway.pem
haproxy_verify_backend_cert: false

aap_gateway_external_host: 192.168.4.50
aap_gateway_fqdn: 192.168.4.50
aap_gateway_vip: 192.168.4.50/24
aap_gateway_vip_interface: ens18
aap_gateway_backend_hosts_csv: "192.168.4.60"

keepalived_peer_ips_csv: "192.168.4.51,192.168.4.52"
keepalived_unicast: true
keepalived_nopreempt: true
configure_firewalld: true
```

Multiple backends: `aap_gateway_backend_hosts_csv: "192.168.4.60,192.168.4.61"`

## Certificate Requirement

By default this repo does not copy private keys from Git. The HAProxy frontend PEM must already exist on both HAProxy nodes:

```text
/etc/haproxy/certs/aap-gateway.pem
```

Expected permissions:

```text
owner: root
group: haproxy
mode: 0640
```

## Job Template Setup

1. Create an AAP inventory with only the HAProxy nodes.
2. Add required host vars to each HAProxy node.
3. Create/sync an AAP project from this repo.
4. Use a custom EE with required collections.
5. Create a Job Template using `playbooks/site.yml`.
6. Enable the survey.
7. Launch the job.

Checklist:

- Project: this repo
- Inventory: only HAProxy nodes
- Playbook: `playbooks/site.yml`
- Execution Environment: custom EE with required collections
- Credential: SSH user with sudo
- Survey: enabled

## Custom Execution Environment

AAP should use a custom EE that includes collections from [collections/requirements.yml](collections/requirements.yml).

Project-based collection syncing may work if enabled in AAP, but the recommended path is a custom EE. Use a pinned supported AAP 2.6 EE base image in production.

See [execution-environment.yml](execution-environment.yml). [Containerfile.example](Containerfile.example) is an optional manual build fallback.

`community.general` is pinned below 10.x for Ansible Core 2.16 (AAP 2.6).

## Validation

```bash
curl -k https://<VIP>
ip -br addr show
systemctl status haproxy --no-pager
systemctl status keepalived --no-pager
journalctl -u haproxy -n 50 --no-pager
journalctl -u keepalived -n 50 --no-pager
```

## Failover Test

1. Find which node owns the VIP (`ip -br addr show` on both nodes).
2. Stop HAProxy on the active node: `sudo systemctl stop haproxy`
3. Confirm the VIP moves to the peer.
4. Restart HAProxy on the original node.
5. Confirm services are healthy.

## Production Notes

- Use an FQDN instead of a VIP IP for external access.
- Set AAP external URL to the VIP/FQDN.
- Use certificate SANs that match the external FQDN.
- Store `keepalived_auth_pass` securely (max 8 characters).
- Consider enabling `haproxy_verify_backend_cert: true` with a backend CA file.
- Consider enabling `haproxy_manage_selinux: true` with `cert_t` on RHEL.
- Set `haproxy_manage_tls_files: true` to deploy PEM from project cert/key files.
- Use pinned EE image tags.
- Confirm interface names (`ens18`, `ens160`, `ens192`, etc.) before setting `aap_gateway_vip_interface`.

Local syntax check only: `ansible-playbook playbooks/site.yml --syntax-check`
