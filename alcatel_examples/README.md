# Alcatel OmniSwitch AOS8 - Intent-Based Network Automation Examples

This directory contains Ansible playbooks for managing Alcatel-Lucent Enterprise OmniSwitch devices running AOS8. Each playbook follows an intent-based pattern: gather current state, compare against desired state, apply corrections where needed, and verify.

All playbooks use the [ale.aos8](https://github.com/ale-nsa-team/ale.aos8) Ansible collection for native AOS8 module support.

## Prerequisites

- Ansible >= 2.15
- `ale.aos8` collection (>= 1.0.2)
- `ansible.netcommon` collection (>= 6.0.0)
- Network connectivity to target OmniSwitch devices via SSH

Install required collections:

```bash
ansible-galaxy collection install -r ../collections/requirements.yml
```

Passwords for local accounts and RADIUS keys must be stored in Ansible Vault:

```bash
ansible-vault create group_vars/vault.yml
```

Add vault variables referenced in `group_vars/alcatel`:

```yaml
vault_admin_password: "changeme"
vault_netops_password: "changeme"
vault_monitor_password: "changeme"
vault_radius_key: "changeme"
```

## Inventory Structure

The inventory (`inventory_alcatel`) uses a dual-hierarchy of **function groups** and **site groups** so that playbooks can target devices by role, location, or both:

```
alcatel                     # all Alcatel devices (connection vars)
├── alcatel_edge            # edge switches  (STP 32768)
│   ├── campus_east_edge
│   └── campus_west_edge
├── alcatel_mas             # MAS switches   (STP 28672)
│   ├── campus_east_mas
│   └── campus_west_mas
└── alcatel_mds             # MDS switches   (STP 24576)
    ├── campus_east_mds
    └── campus_west_mds

campus_east                 # all Campus East devices (NPS: CE primary)
campus_west                 # all Campus West devices (NPS: CW primary)
```

A device belongs to both a function group and a site group. For example, `ce-edge-sw01` is in `alcatel_edge` (gets edge STP priority) and `campus_east` (gets Campus East NPS servers).

## Playbooks

| Playbook | Hosts | Description |
|---|---|---|
| `upgrade_os6860e.yml` | `alcatel_edge` | Stage firmware, upgrade, validate, report. Uses `serial: 20%` to limit blast radius. |
| `alcatel_nps_config.yml` | `alcatel` | Configure RADIUS/NPS servers based on site. Uses `aos8_radius_servers` resource module. |
| `alcatel_stp_standards.yml` | `alcatel` | Enforce STP bridge priority by device function (edge/MAS/MDS). |
| `alcatel_vlan1_enforcement.yml` | `alcatel` | Validate VLAN 1 is disabled; auto-disable if found enabled. Uses `aos8_vlans` resource module. |
| `alcatel_dhcp_snooping.yml` | `alcatel_edge` | Enable DHCP snooping. Auto-detects uplinks via LLDP and trusts them. |
| `alcatel_bpdu_guard.yml` | `alcatel_edge` | Enable BPDU guard on edge-facing ports. Auto-excludes network device uplinks and LAG members via LLDP. |
| `alcatel_local_accounts.yml` | `alcatel` | Ensure approved accounts exist, remove unauthorized accounts, rotate passwords. |
| `alcatel_ddm.yml` | `alcatel` | Enable DDM and DDM traps on all ports with transceivers installed. |

## Usage

Every playbook supports `--check` mode for a read-only preview that shows what would change without touching the device.

```bash
# Preview changes (no modifications made):
ansible-playbook alcatel_stp_standards.yml -i inventory_alcatel --check

# Apply changes:
ansible-playbook alcatel_stp_standards.yml -i inventory_alcatel

# Target a specific site:
ansible-playbook alcatel_nps_config.yml -i inventory_alcatel --limit campus_east

# Target a specific device:
ansible-playbook alcatel_ddm.yml -i inventory_alcatel --limit ce-edge-sw01

# Rotate passwords (requires explicit tag):
ansible-playbook alcatel_local_accounts.yml -i inventory_alcatel --tags all,rotate
```

## Playbook Pattern

Each playbook follows a consistent 4-phase pattern:

1. **Gather** -- read current device state using `ale.aos8.aos8_command` or resource module `state: gathered`
2. **Compare** -- determine drift from desired state, display a plan showing what will change
3. **Apply** -- push corrections using `ale.aos8.aos8_config` or resource modules (`state: merged`/`replaced`). Skipped in `--check` mode.
4. **Verify** -- re-read device state and `assert` it matches intent

For playbooks using resource modules (`alcatel_nps_config.yml`, `alcatel_vlan1_enforcement.yml`), the module returns structured `before`, `after`, and `commands` output showing exactly what changed.

## Group Variables

| File | Scope | Contents |
|---|---|---|
| `group_vars/alcatel` | All Alcatel devices | Connection settings, approved local accounts, DHCP snooping VLANs, firmware target |
| `group_vars/alcatel_edge` | Edge switches | `stp_bridge_priority: 32768`, `device_role: edge` |
| `group_vars/alcatel_mas` | MAS switches | `stp_bridge_priority: 28672`, `device_role: mas` |
| `group_vars/alcatel_mds` | MDS switches | `stp_bridge_priority: 24576`, `device_role: mds` |
| `group_vars/campus_east` | Campus East site | NPS primary = CE, secondary = CW |
| `group_vars/campus_west` | Campus West site | NPS primary = CW, secondary = CE |

## Running in AAP / Automation Controller

These playbooks work as AAP job templates:

- **Preview workflow**: Create a job template with **Job Type: Check**. The output shows what would change without modifying devices. Add an **Approval Node** before the apply step so management can review.
- **Apply workflow**: Same template with **Job Type: Run**.
- **Tags**: Use the job template **Job Tags** field to select specific config sections (e.g., `ntp`, `vlans`, `rotate`).
- **Limit**: Use the **Limit** field to target specific sites or devices.
- **Credentials**: Store vault passwords and device credentials as AAP credential objects rather than local files.
