import http.client
import json
import argparse
import ssl
import csv
import sys
from datetime import datetime, timezone

NTNX_PRISMCENTRAL_IP = "YOUR_IP:9440"
PC_TOKEN = "YOUR GENERATED TOKEN FROM nutanix_auth.py"


def get_conn():
    context = ssl._create_unverified_context()
    return http.client.HTTPSConnection(NTNX_PRISMCENTRAL_IP, context=context)


def api_request(method, url, payload=None):
    conn = get_conn()
    headers = {
        "Accept": "application/json",
        "Authorization": PC_TOKEN,
        "Content-Type": "application/json"
    }

    body = None
    if payload is not None:
        body = payload if isinstance(payload, str) else json.dumps(payload)

    conn.request(method, url, body=body, headers=headers)
    res = conn.getresponse()
    raw = res.read().decode("utf-8")

    try:
        data = json.loads(raw) if raw else {}
    except json.JSONDecodeError:
        data = {"raw": raw}

    if res.status >= 400:
        raise RuntimeError(f"API error {res.status} on {url}: {data}")

    return data


def list_vms(page_size=100):
    offset = 0
    results = []

    while True:
        payload = {
            "kind": "vm",
            "length": page_size,
            "offset": offset
        }

        data = api_request("POST", "/api/nutanix/v3/vms/list", payload)
        entities = data.get("entities", [])

        if not entities:
            break

        results.extend(entities)

        total_matches = data.get("metadata", {}).get("total_matches")
        offset += page_size

        if total_matches is not None and offset >= total_matches:
            break

    return results


def mib_to_gib(value):
    if value in (None, ""):
        return None

    try:
        return round(float(value) / 1024, 2)
    except (TypeError, ValueError):
        return None


def safe_json(value):
    if value in (None, ""):
        return ""
    return json.dumps(value, ensure_ascii=False, sort_keys=True)


def get_resource(vm, key, default=None):
    return (
        vm.get("status", {}).get("resources", {}).get(key)
        or vm.get("spec", {}).get("resources", {}).get(key)
        or default
    )


def extract_vm_row(vm):
    metadata = vm.get("metadata", {})
    spec = vm.get("spec", {})
    status = vm.get("status", {})

    resources = status.get("resources") or spec.get("resources") or {}
    nics = resources.get("nic_list", []) or []
    disks = resources.get("disk_list", []) or []

    mac_addresses = []
    ip_addresses = []
    subnet_names = []
    subnet_uuids = []

    for nic in nics:
        mac = nic.get("mac_address")
        if mac:
            mac_addresses.append(mac)

        for ip_endpoint in nic.get("ip_endpoint_list", []) or []:
            ip = ip_endpoint.get("ip")
            if ip:
                ip_addresses.append(ip)

        subnet_ref = nic.get("subnet_reference") or {}
        if subnet_ref.get("name"):
            subnet_names.append(subnet_ref.get("name"))
        if subnet_ref.get("uuid"):
            subnet_uuids.append(subnet_ref.get("uuid"))

    disk_total_mib = 0
    disk_count = 0

    for disk in disks:
        size = disk.get("disk_size_mib")
        if isinstance(size, (int, float)):
            disk_total_mib += size
            disk_count += 1

    cluster_ref = (
        status.get("cluster_reference")
        or spec.get("cluster_reference")
        or {}
    )

    project_ref = metadata.get("project_reference") or {}

    cpu = resources.get("num_vcpus_per_socket")
    sockets = resources.get("num_sockets")
    total_vcpus = None

    if isinstance(cpu, int) and isinstance(sockets, int):
        total_vcpus = cpu * sockets
    else:
        total_vcpus = cpu

    return {
        "name": spec.get("name") or status.get("name"),
        "uuid": metadata.get("uuid"),
        "cluster_name": cluster_ref.get("name"),
        "cluster_uuid": cluster_ref.get("uuid"),
        "power_state": resources.get("power_state"),
        "vcpus": total_vcpus,
        "sockets": sockets,
        "vcpus_per_socket": cpu,
        "memory_gib": mib_to_gib(resources.get("memory_size_mib")),
        "disk_count": disk_count,
        "disk_total_gib": mib_to_gib(disk_total_mib),
        "mac_addresses": ", ".join(mac_addresses),
        "ip_addresses": ", ".join(ip_addresses),
        "subnet_names": ", ".join(subnet_names),
        "subnet_uuids": ", ".join(subnet_uuids),
        "categories": safe_json(metadata.get("categories", {})),
        "project_name": project_ref.get("name"),
        "project_uuid": project_ref.get("uuid"),
        "description": spec.get("description") or "",
    }


def write_csv(rows, output_file):
    fieldnames = [
        "name",
        "uuid",
        "cluster_name",
        "cluster_uuid",
        "power_state",
        "vcpus",
        "sockets",
        "vcpus_per_socket",
        "memory_gib",
        "disk_count",
        "disk_total_gib",
        "mac_addresses",
        "ip_addresses",
        "subnet_names",
        "subnet_uuids",
        "categories",
        "project_name",
        "project_uuid",
        "description",
    ]

    target = open(output_file, "w", newline="", encoding="utf-8") if output_file else sys.stdout

    try:
        writer = csv.DictWriter(target, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(rows)
    finally:
        if output_file:
            target.close()


def write_json(rows, output_file):
    data = {
        "generated_at": datetime.now(timezone.utc).isoformat(),
        "prism_central": NTNX_PRISMCENTRAL_IP,
        "vm_count": len(rows),
        "vms": rows
    }

    text = json.dumps(data, indent=2, ensure_ascii=False)

    if output_file:
        with open(output_file, "w", encoding="utf-8") as file:
            file.write(text)
            file.write("\n")
    else:
        print(text)


def markdown_escape(value):
    if value is None:
        return ""
    return str(value).replace("|", "\\|").replace("\n", " ")


def write_markdown(rows, output_file):
    columns = [
        ("name", "Name"),
        ("uuid", "UUID"),
        ("cluster_name", "Cluster"),
        ("power_state", "Power State"),
        ("vcpus", "vCPUs"),
        ("memory_gib", "RAM GiB"),
        ("disk_total_gib", "Disk GiB"),
        ("ip_addresses", "IP Addresses"),
        ("categories", "Categories"),
    ]

    lines = []
    lines.append("# Nutanix VM Inventory")
    lines.append("")
    lines.append(f"Generated at: `{datetime.now(timezone.utc).isoformat()}`")
    lines.append("")
    lines.append(f"VM count: **{len(rows)}**")
    lines.append("")
    lines.append("| " + " | ".join(label for _, label in columns) + " |")
    lines.append("| " + " | ".join("---" for _ in columns) + " |")

    for row in rows:
        lines.append("| " + " | ".join(markdown_escape(row.get(key)) for key, _ in columns) + " |")

    text = "\n".join(lines) + "\n"

    if output_file:
        with open(output_file, "w", encoding="utf-8") as file:
            file.write(text)
    else:
        print(text)


def main():
    parser = argparse.ArgumentParser(
        description="Export Nutanix VM inventory from Prism Central to CSV, JSON, or Markdown."
    )
    parser.add_argument(
        "--format",
        required=True,
        choices=["csv", "json", "markdown"],
        help="Export format"
    )
    parser.add_argument(
        "--output",
        required=False,
        help="Output file path. If omitted, output is printed to stdout."
    )

    args = parser.parse_args()

    vms = list_vms()
    rows = [extract_vm_row(vm) for vm in vms]
    rows.sort(key=lambda item: str(item.get("name") or "").lower())

    if args.format == "csv":
        write_csv(rows, args.output)
    elif args.format == "json":
        write_json(rows, args.output)
    elif args.format == "markdown":
        write_markdown(rows, args.output)

    if args.output:
        print(f"Exported {len(rows)} VMs to {args.output}")


if __name__ == "__main__":
    main()
