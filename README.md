# Nutanix VM Inventory Export Script

A Python script for exporting Nutanix VM inventory from Prism Central to CSV, JSON, or Markdown.

## Features

- Lists VMs from Prism Central
- Exports VM inventory to CSV
- Exports VM inventory to JSON
- Exports VM inventory to Markdown
- Includes VM name and UUID
- Includes cluster information
- Includes power state
- Includes CPU, memory, and disk information
- Includes NIC, MAC address, IP address, and subnet information
- Includes categories and project references
- Uses only the Python standard library

## Requirements

- Python 3.8 or newer
- Network access to Nutanix Prism Central
- A valid Nutanix Prism Central API token
- Permission to read VM inventory

No external Python packages are required.

## Configuration

Before running the script, update these values in `nutanix_vm_inventory_export.py`:

```python
NTNX_PRISMCENTRAL_IP = "YOUR_IP:9440"
PC_TOKEN = "YOUR GENERATED TOKEN FROM nutanix_auth.py"
```

## Usage

Export to CSV:

```bash
python nutanix_vm_inventory_export.py \
  --format csv \
  --output inventory.csv
```

Export to JSON:

```bash
python nutanix_vm_inventory_export.py \
  --format json \
  --output inventory.json
```

Export to Markdown:

```bash
python nutanix_vm_inventory_export.py \
  --format markdown \
  --output inventory.md
```

Print JSON to stdout:

```bash
python nutanix_vm_inventory_export.py --format json
```

## Arguments

| Argument | Required | Description |
|---|---:|---|
| `--format` | Yes | `csv`, `json`, or `markdown` |
| `--output` | No | Output file path. If omitted, output is printed to stdout. |

## Exported Fields

The CSV and JSON output can include:

```text
name
uuid
cluster_name
cluster_uuid
power_state
vcpus
sockets
vcpus_per_socket
memory_gib
disk_count
disk_total_gib
mac_addresses
ip_addresses
subnet_names
subnet_uuids
categories
project_name
project_uuid
description
```

## Security Notes

Do not commit real API tokens, passwords, Prism Central addresses, exported inventory files, or internal infrastructure details to a public GitHub repository.

The script currently disables SSL certificate verification by using:

```python
ssl._create_unverified_context()
```

This may be useful in lab environments, but it is not recommended for production. For production use, configure proper certificate validation.

## Disclaimer

This script is provided as an example. Test it in a safe environment before using it against production Nutanix infrastructure.
