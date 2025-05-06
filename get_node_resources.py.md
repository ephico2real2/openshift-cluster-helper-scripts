You're right! Here's the full script with the proper shell commands to create the file:

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

def format_bytes(bytes_val):
    if bytes_val >= 1024**3:
        return f"{bytes_val / 1024**3:.2f} GiB"
    elif bytes_val >= 1024**2:
        return f"{bytes_val / 1024**2:.2f} MiB"
    elif bytes_val >= 1024:
        return f"{bytes_val / 1024:.2f} KiB"
    else:
        return f"{bytes_val} B"

def get_node_resources(label, node_type):
    print(f"===== {node_type} Node Resources =====")
    
    # Get nodes with the label
    cmd = f"oc get nodes -l '{label}' -o name"
    nodes = run_cmd(cmd).split('\n')
    
    if not nodes or nodes[0] == '':
        print(f"No {node_type} nodes found")
        print("==========================")
        return 0, 0
    
    print(f"{'NODE NAME':<30} {'CPU CORES':<10} {'MEMORY':<20}")
    print("-" * 70)
    
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
        
        print(f"{node_name:<30} {cpu_count:<10} {memory} ({format_bytes(memory_bytes):<20})")
    
    print("=" * 70)
    print(f"{'TOTAL':<30} {total_cpu:<10} {format_bytes(total_memory_bytes):<20}")
    print()
    
    return total_cpu, total_memory_bytes

# Get resources for each node type
master_cpu, master_mem = get_node_resources("node-role.kubernetes.io/master", "Master")
infra_cpu, infra_mem = get_node_resources("node-role.kubernetes.io/infra,node-role.kubernetes.io/worker", "Infrastructure")
app_cpu, app_mem = get_node_resources("node-role.kubernetes.io/app,node-role.kubernetes.io/worker", "Application")

# Summary
print("===== Total Cluster Resources =====")
print(f"{'NODE TYPE':<30} {'CPU CORES':<10} {'MEMORY':<20}")
print("-" * 70)
print(f"{'Master Nodes':<30} {master_cpu:<10} {format_bytes(master_mem):<20}")
print(f"{'Infrastructure Nodes':<30} {infra_cpu:<10} {format_bytes(infra_mem):<20}")
print(f"{'Application Nodes':<30} {app_cpu:<10} {format_bytes(app_mem):<20}")
print("-" * 70)
print(f"{'All Nodes':<30} {master_cpu + infra_cpu + app_cpu:<10} {format_bytes(master_mem + infra_mem + app_mem):<20}")
EOF

# Make it executable
chmod +x get_node_resources.py

# Run the script
./get_node_resources.py
```

Now you have the complete script with the proper shell commands to create the file and make it executable. This should work with Python 3.6 as it uses compatible subprocess parameters.
