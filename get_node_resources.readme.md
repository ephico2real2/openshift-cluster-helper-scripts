I'll update the full script with exact conversion factors instead of percentage approximations:

```bash
cat << 'EOF' > get_node_resources.py
#!/usr/bin/env python3
import subprocess
import re

# Exact conversion factors based on binary and decimal definitions
KIB_TO_KB = 1024 / 1000  # 1.024
MIB_TO_MB = (1024 * 1024) / (1000 * 1000)  # 1.048576
GIB_TO_GB = (1024 * 1024 * 1024) / (1000 * 1000 * 1000)  # 1.073741824

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

def format_bytes(bytes_val):
    # Binary format (GiB)
    gib_value = bytes_val / (1024**3)
    # Decimal format (GB) with exact conversion factor
    gb_value = gib_value * GIB_TO_GB
    
    return f"{gib_value:.2f} GiB ({gb_value:.2f} GB)"

def get_node_resources(label, node_type):
    # First check if any nodes exist with this label to avoid hanging
    check_cmd = f"oc get nodes -l '{label}' --no-headers | wc -l"
    count = int(run_cmd(check_cmd))
    
    if count == 0:
        print(f"===== {node_type} Node Resources =====")
        print(f"No {node_type} nodes found")
        print("==========================")
        return 0, 0
    
    # Get CPU and memory directly with one command per type to reduce API calls
    cpu_cmd = f"oc get nodes -l '{label}' -o jsonpath='{{.items[*].status.capacity.cpu}}'"
    cpu_values = run_cmd(cpu_cmd).split()
    
    mem_cmd = f"oc get nodes -l '{label}' -o jsonpath='{{.items[*].status.capacity.memory}}'"
    mem_values = run_cmd(mem_cmd).split()
    
    names_cmd = f"oc get nodes -l '{label}' -o jsonpath='{{.items[*].metadata.name}}'"
    node_names = run_cmd(names_cmd).split()
    
    # Process results
    total_cpu = 0
    total_memory_bytes = 0
    
    print(f"===== {node_type} Node Resources =====")
    print(f"{'NODE NAME':<45} {'CPU CORES':<10} {'MEMORY':<40}")
    print("-" * 95)
    
    for i in range(len(node_names)):
        if i < len(cpu_values) and i < len(mem_values):
            cpu_count = int(cpu_values[i])
            memory = mem_values[i]
            memory_bytes = convert_to_bytes(memory)
            
            total_cpu += cpu_count
            total_memory_bytes += memory_bytes
            
            print(f"{node_names[i]:<45} {cpu_count:<10} {memory} | {format_bytes(memory_bytes)}")
    
    print("=" * 95)
    print(f"{'TOTAL':<45} {total_cpu:<10} {format_bytes(total_memory_bytes)}")
    print()
    
    return total_cpu, total_memory_bytes

# Get resources for each node type
master_cpu, master_mem = get_node_resources("node-role.kubernetes.io/master", "Master")
infra_cpu, infra_mem = get_node_resources("node-role.kubernetes.io/infra,node-role.kubernetes.io/worker", "Infrastructure")
app_cpu, app_mem = get_node_resources("node-role.kubernetes.io/app,node-role.kubernetes.io/worker", "Application")

# Summary
print("===== Total Cluster Resources =====")
print(f"{'NODE TYPE':<30} {'CPU CORES':<10} {'MEMORY':<40}")
print("-" * 80)
print(f"{'Master Nodes':<30} {master_cpu:<10} {format_bytes(master_mem)}")
print(f"{'Infrastructure Nodes':<30} {infra_cpu:<10} {format_bytes(infra_mem)}")
print(f"{'Application Nodes':<30} {app_cpu:<10} {format_bytes(app_mem)}")
print("-" * 80)
total_cpu = master_cpu + infra_cpu + app_cpu
total_mem = master_mem + infra_mem + app_mem
print(f"{'All Nodes':<30} {total_cpu:<10} {format_bytes(total_mem)}")
EOF

# Make it executable
chmod +x get_node_resources.py

# Run the script
./get_node_resources.py
```

Key changes made:

1. Added exact conversion constants at the top of the script:
   ```python
   KIB_TO_KB = 1024 / 1000  # 1.024
   MIB_TO_MB = (1024 * 1024) / (1000 * 1000)  # 1.048576
   GIB_TO_GB = (1024 * 1024 * 1024) / (1000 * 1000 * 1000)  # 1.073741824
   ```

2. Updated the `format_bytes` function to use the exact GIB_TO_GB conversion factor:
   ```python
   # Binary format (GiB)
   gib_value = bytes_val / (1024**3)
   # Decimal format (GB) with exact conversion factor
   gb_value = gib_value * GIB_TO_GB
   ```

3. Maintained the optimization improvements from before to prevent hanging:
   - Get all node data in batch operations
   - Check node existence before trying to process them
   - Batch JSON queries to reduce API calls

This script should now provide mathematically precise conversions between binary and decimal units while still running efficiently.
