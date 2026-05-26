# AAP HAProxy / Keepalived

### Purpose

This repository configures two RHEL HAProxy/Keepalived nodes in front of **Ansible Automation Platform 2.6** platform gateway. It is designed to run from an **AAP 2.6 Job Template** — not from local CLI inventory workflows.

## Architecture

```
User/DNS -> VIP -> HAProxy pair -> AAP gateway/backend
```

- The **VIP** floats between the two HAProxy nodes via Keepalived.
- **HAProxy** terminates client TLS on port 443.
- **HAProxy** re-encrypts to the AAP gateway/backend over HTTPS (HTTP/1.1).
- The **AAP backend/controller host is not managed** by this playbook.

## AAP Inventory Model

The selected AAP inventory must contain **only** the HAProxy/Keepalived nodes.

Example inventory hosts:

- `aap-lb-01`
- `aap-lb-02`

Do **not** include the AAP backend/controller host in this inventory. The playbook uses `hosts: all`, so every host in the selected inventory receives the role.

## Required AAP Host Vars

Set these per node in AAP inventory host variables (not in the survey):

**aap-lb-01:**

```yaml
ansible_host: 192.168.4.51
keepalived_state: BACKUP
keepalived_priority: 101
keepalived_unicast_src_ip: 192.168.4.51
keepalived_unicast_peers:
  - 192.168.4.52
```

**aap-lb-02:**

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

- [aap/survey_spec.yml](aap/survey_spec.yml) — Job Template survey questions
- [aap/extra_vars.example.yml](aap/extra_vars.example.yml) — working lab example

Key working lab values:

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

Multiple backends are supported via CSV, for example: `"192.168.4.60,192.168.4.61"`.

## Job Template Setup

| Setting | Value |
|---------|-------|
| Project | This repository |
| Inventory | Only the two HAProxy nodes |
| Playbook | `playbooks/site.yml` |
| Execution Environment | Custom EE with required collections |
| Machine credential | SSH user with sudo |
| Survey | Enabled (see `aap/survey_spec.yml`) |

## Custom EE

The recommended path is a **custom execution environment** with collections installed from [collections/requirements.yml](collections/requirements.yml).

1. Use a supported, **pinned** AAP 2.6 EE base image (not floating `latest` in production).
2. Install collections from `collections/requirements.yml`.
3. Push the image to your registry and assign it on the Job Template.

Reference: [execution-environment.yml](execution-environment.yml)

**Lab fallback:** [Containerfile.example](Containerfile.example) shows a manual podman build. Use for lab/reference only.

Project-based collection install may work if enabled in your AAP organization, but the tested approach is a custom EE.

`community.general` is pinned below 10.x because version 10+ does not support Ansible Core 2.16 used by AAP 2.6.

## Validation

```bash
curl -k https://<VIP>/api/gateway/v1/ping/
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
5. With `keepalived_nopreempt: true`, confirm the VIP stays on the peer until you manually fail back.

## Production Notes

- Use an **FQDN**, not a VIP IP, for external access.
- Set AAP `gateway_main_url` to the external FQDN.
- Use certificates whose SAN matches the external name.
- Store `keepalived_auth_pass` securely (max 8 characters; use vault in production).
- Consider enabling `haproxy_verify_backend_cert: true` with a backend CA file when using an internal CA.
- Consider enabling `haproxy_manage_selinux: true` with `cert_t` for certificate labeling on RHEL.
- HAProxy access/error logging is not configured by this role; use `journalctl -u haproxy` or your central logging standard.
- Use **pinned EE image tags**, not floating `latest`.
- Validate interface names on each node (`ens18`, `ens160`, `ens192`, etc.) before setting `aap_gateway_vip_interface`.

Optional production modes (set via survey or extra vars):

- `haproxy_manage_tls_files: true` — deploy PEM from project cert/key files
- `haproxy_verify_backend_cert: true` — requires `haproxy_backend_ca_src` in the synced project
- `haproxy_manage_selinux: true` — label cert directory with `cert_t` on RHEL

## Layout

```
README.md
ansible.cfg
playbooks/site.yml
roles/aap_haproxy/
aap/survey_spec.yml
aap/extra_vars.example.yml
collections/requirements.yml
execution-environment.yml
Containerfile.example
LICENSE
```
