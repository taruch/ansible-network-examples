# Network Troubleshooting and Information Gathering Examples

This directory contains playbooks that provide a unified method for querying network devices and collecting operational information across vendors. Engineers run one command and get a consistent report regardless of whether the target is Cisco IOS, Cisco NXOS, or Alcatel AOS8.

## How Multi-Vendor Works

Each playbook uses the same two-phase approach:

1. **Gather** -- platform-specific facts modules (`cisco.ios.ios_facts`, `cisco.nxos.nxos_facts`, `ale.aos8.aos8_facts`) are selected automatically based on `ansible_network_os`. The engineer doesn't need to know which module to use.
2. **Analyze and Report** -- all analysis runs against `ansible_network_resources`, which has the **same structure regardless of vendor**. The reporting tasks are completely platform-independent.

This means one playbook, one command, one report format -- no matter how many vendors are in the environment.

## Playbooks

### `network_stp_blocking.yml` -- STP Blocking Port Report

Queries devices for ports in Spanning Tree blocking state and produces a per-device report plus a cross-device summary showing which switches have blocked ports.

```bash
# All devices in inventory
ansible-playbook troubleshooting_examples/network_stp_blocking.yml

# Scope by site
ansible-playbook troubleshooting_examples/network_stp_blocking.yml --limit campus_east

# Scope by device function
ansible-playbook troubleshooting_examples/network_stp_blocking.yml --limit alcatel_edge

# Scope by vendor
ansible-playbook troubleshooting_examples/network_stp_blocking.yml --limit 'nexus:cisco'

# Single switch
ansible-playbook troubleshooting_examples/network_stp_blocking.yml --limit nxosv1
```

**Output includes:**
- Per-device list of blocked ports with VLAN and interface
- Cross-device summary identifying which switches have blocking ports
- Total device count scanned

### `network_port_investigation.yml` -- LLDP, LACP, and VLAN Investigation

Queries a switch and cross-references LLDP neighbor data, LACP bundle membership, and VLAN tagging into a single unified report. For each LLDP neighbor, the report shows:

- The local port where the neighbor is connected
- Whether that port is a member of an LACP bundle, and if so, which one
- All VLANs (access, native, tagged) on the port or its parent bundle

```bash
# Investigate one switch
ansible-playbook troubleshooting_examples/network_port_investigation.yml --limit nxosv1

# Investigate all switches at a site
ansible-playbook troubleshooting_examples/network_port_investigation.yml --limit campus_east

# Investigate all switches
ansible-playbook troubleshooting_examples/network_port_investigation.yml
```

**Output includes:**
- LLDP neighbor table with LACP and VLAN cross-reference per port
- Complete LAG interface summary with member ports and VLAN assignments
- Port statistics (LLDP neighbor count, LAG count, L2 port count)

## Using with Multiple Inventories

To query across both Cisco and Alcatel devices in a single run, pass multiple inventory sources:

```bash
ansible-playbook troubleshooting_examples/network_stp_blocking.yml \
  -i inventory \
  -i alcatel_examples/inventory_alcatel
```

## Using `--limit` for Scoping

The `--limit` flag is the primary mechanism for scoping queries to a building, site, device type, or any logical grouping defined in inventory. Any inventory group name works:

| Scope | `--limit` value | What it targets |
|---|---|---|
| One device | `nxosv1` | Single switch |
| Site | `campus_east` | All devices at Campus East |
| Device function | `alcatel_edge` | All edge switches |
| Vendor | `nexus` or `cisco` | All Cisco NXOS or IOS devices |
| Multiple groups | `'campus_east:campus_west'` | Both sites |
| Exclude | `'alcatel:!alcatel_mds'` | Alcatel devices except MDS |

## Running in AAP / Automation Controller

Create a single job template per playbook. Engineers select the scope at launch time using the **Limit** field -- no need to create separate templates per site or device type. The same template works for any combination of devices.

For recurring audits, attach a **Schedule** to run the STP blocking report nightly and notify on findings.

## Prerequisites

- Ansible >= 2.15
- `cisco.ios` collection
- `cisco.nxos` collection
- `ale.aos8` collection (>= 1.0.2) -- only needed if Alcatel devices are in scope
- `ansible.netcommon` collection (>= 6.0.0)
- LLDP must be enabled on devices for the port investigation playbook to return neighbor data
