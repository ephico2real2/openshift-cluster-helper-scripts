The scripts account for both binary (GiB) and decimal (GB) representations of memory, taking into account the 7.4% difference between them as described in the article. This will help make the output more comprehensive.

```bash
cat << 'EOF' > get_node_resources.py
#!/usr/bin/env python3
import subprocess
import json
import re

def run_cmd(cmd):
    # Modified to work with Python 3.6
    result = subprocess.run(cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, universal_newlines=True)
    return result.stdout.strip()

def convert_to_bytes(memory_str):
    # Extract number and unit
    match = re.match(r'(\d+)(\w+)', memory_str)
    if not match:
        return 0
    
    number, unit = match.groups()
    number = int(number)
    
    if unit == 'Ki':
        return number * 1024
    elif unit == 'Mi':
        return number * 1024 * 1024
    elif unit == 'Gi':
        return number * 1024 * 1024 * 1024
    else:
        return number

def format_bytes(bytes_val, include_decimal=True):
    # Binary format (GiB, MiB, KiB)
    binary_format = ""
    if bytes_val >= 1024**3:
        binary_format = f"{bytes_val / (1024**3):.2f} GiB"
    elif bytes_val >= 1024**2:
        binary_format = f"{bytes_val / (1024**2):.2f} MiB"
    elif bytes_val >= 1024:
        binary_format = f"{bytes_val / 1024:.2f} KiB"
    else:
        binary_format = f"{bytes_val} B"
    
    # If we don't need decimal format, just return binary
    if not include_decimal:
        return binary_format
    
    # Decimal format (GB, MB, KB)
    decimal_format = ""
    if bytes_val >= 1024**3:
        # Convert from binary GiB to decimal GB (roughly 7.4% difference)
        gib_value = bytes_val / (1024**3)
        gb_value = gib_value * 1.074  # Apply correction factor
        decimal_format = f"{gb_value:.2f} GB"
    elif bytes_val >= 1024**2:
        mib_value = bytes_val / (1024**2)
        mb_value = mib_value * 1.049  # Apply correction for MiB to MB (roughly 4.9% difference)
        decimal_format = f"{mb_value:.2f} MB"
    elif bytes_val >= 1024:
        kib_value = bytes_val / 1024
        kb_value = kib_value * 1.024  # Apply correction for KiB to KB (roughly 2.4% difference)
        decimal_format = f"{kb_value:.2f} KB"
    else:
        decimal_format = f"{bytes_val} B"
    
    return f"{binary_format} ({decimal_format})"

def get_node_resources(label, node_type):
    print(f"===== {node_type} Node Resources =====")
    
    # Get nodes with the label
    cmd = f"oc get nodes -l '{label}' -o name"
    nodes = run_cmd(cmd).split('\n')
    
    if not nodes or nodes[0] == '':
        print(f"No {node_type} nodes found")
        print("==========================")
        return 0, 0
    
    print(f"{'NODE NAME':<45} {'CPU CORES':<10} {'MEMORY':<45}")
    print("-" * 100)
    
    total_cpu = 0
    total_memory_bytes = 0
    
    for node in nodes:
        if not node:
            continue
            
        node_name = node.split('/')[-1]
        
        # Get CPU
        cmd = f"oc get node {node_name} -o jsonpath='{{.status.capacity.cpu}}'"
        cpu_count = int(run_cmd(cmd))
        total_cpu += cpu_count
        
        # Get memory
        cmd = f"oc get node {node_name} -o jsonpath='{{.status.capacity.memory}}'"
        memory = run_cmd(cmd)
        memory_bytes = convert_to_bytes(memory)
        total_memory_bytes += memory_bytes
        
        print(f"{node_name:<45} {cpu_count:<10} {memory} | {format_bytes(memory_bytes)}")
    
    print("=" * 100)
    print(f"{'TOTAL':<45} {total_cpu:<10} {format_bytes(total_memory_bytes)}")
    print()
    
    return total_cpu, total_memory_bytes

# Get resources for each node type
master_cpu, master_mem = get_node_resources("node-role.kubernetes.io/master", "Master")
infra_cpu, infra_mem = get_node_resources("node-role.kubernetes.io/infra,node-role.kubernetes.io/worker", "Infrastructure")
app_cpu, app_mem = get_node_resources("node-role.kubernetes.io/app,node-role.kubernetes.io/worker", "Application")

# Summary
print("===== Total Cluster Resources =====")
print(f"{'NODE TYPE':<30} {'CPU CORES':<10} {'MEMORY':<45}")
print("-" * 90)
print(f"{'Master Nodes':<30} {master_cpu:<10} {format_bytes(master_mem)}")
print(f"{'Infrastructure Nodes':<30} {infra_cpu:<10} {format_bytes(infra_mem)}")
print(f"{'Application Nodes':<30} {app_cpu:<10} {format_bytes(app_mem)}")
print("-" * 90)
print(f"{'All Nodes':<30} {master_cpu + infra_cpu + app_cpu:<10} {format_bytes(master_mem + infra_mem + app_mem)}")
EOF

# Make it executable
chmod +x get_node_resources.py

# Run the script
./get_node_resources.py
```

The key changes I've made to incorporate the correction factors:

1. Enhanced the `format_bytes()` function to provide both binary (GiB) and decimal (GB) representations
2. Added correction factors based on the article:
   - For GiB to GB: 7.4% (multiply by 1.074)
   - For MiB to MB: 4.9% (multiply by 1.049)
   - For KiB to KB: 2.4% (multiply by 1.024)
3. Expanded the column widths to accommodate the additional information
4. Formatted the output to show both representations like "125.51 GiB (134.80 GB)"

The above show binary units (which Kubernetes uses internally) and the equivalent decimal units, taking into account the differences between them as described in the article.
