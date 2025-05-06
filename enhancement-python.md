This is an alternative to using the `bc` command for floating-point calculations in bash scripts. Below is a modified version of the script that uses `awk` instead, which is more commonly available on Unix/Linux systems:

```bash
cat << 'EOF' > get_all_node_resources_no_bc.sh
#!/bin/bash

# Function to convert memory units to bytes
convert_to_bytes() {
  local value=$1
  
  # Extract number and unit
  local number=$(echo $value | sed 's/[^0-9]*//g')
  local unit=$(echo $value | sed 's/[0-9]*//g')
  
  case $unit in
    Ki) echo $((number * 1024)) ;;
    Mi) echo $((number * 1024 * 1024)) ;;
    Gi) echo $((number * 1024 * 1024 * 1024)) ;;
    *) echo $number ;;
  esac
}

# Function to format bytes to human-readable format using awk instead of bc
format_bytes() {
  local bytes=$1
  
  if [ $bytes -ge 1073741824 ]; then
    echo $(awk -v bytes=$bytes 'BEGIN {printf "%.2f GiB", bytes/1073741824}')
  elif [ $bytes -ge 1048576 ]; then
    echo $(awk -v bytes=$bytes 'BEGIN {printf "%.2f MiB", bytes/1048576}')
  elif [ $bytes -ge 1024 ]; then
    echo $(awk -v bytes=$bytes 'BEGIN {printf "%.2f KiB", bytes/1024}')
  else
    echo "$bytes B"
  fi
}

# Function to get resources for a specific node type
get_node_resources() {
  local label=$1
  local node_type=$2
  
  echo "===== $node_type Node Resources ====="
  total_cpu=0
  total_memory_bytes=0
  
  # Get nodes with the specific label
  nodes=$(oc get nodes -l "$label" -o name)
  
  if [ -z "$nodes" ]; then
    echo "No $node_type nodes found"
    echo "=========================="
    return
  fi
  
  printf "%-30s %-10s %-20s\n" "NODE NAME" "CPU CORES" "MEMORY"
  echo "----------------------------------------------------------------------"
  
  # Iterate through each node
  for node in $nodes; do
    node_name=$(echo $node | cut -d '/' -f 2)
    
    # Get CPU count
    cpu_count=$(oc get node $node_name -o jsonpath='{.status.capacity.cpu}')
    total_cpu=$((total_cpu + cpu_count))
    
    # Get memory
    memory=$(oc get node $node_name -o jsonpath='{.status.capacity.memory}')
    memory_bytes=$(convert_to_bytes $memory)
    total_memory_bytes=$((total_memory_bytes + memory_bytes))
    
    # Display resources
    printf "%-30s %-10s %-20s\n" "$node_name" "$cpu_count" "$memory ($(format_bytes $memory_bytes))"
  done
  
  echo "======================================================================"
  printf "%-30s %-10s %-20s\n" "TOTAL" "$total_cpu" "$(format_bytes $total_memory_bytes)"
  echo ""
}

# Get resources for each node type
get_node_resources "node-role.kubernetes.io/master" "Master"
get_node_resources "node-role.kubernetes.io/infra,node-role.kubernetes.io/worker" "Infrastructure"
get_node_resources "node-role.kubernetes.io/app,node-role.kubernetes.io/worker" "Application"

# Summary of all nodes
echo "===== Total Cluster Resources ====="
all_cpu=$(oc get nodes -o jsonpath='{range .items[*]}{.status.capacity.cpu}{"\n"}{end}' | awk '{sum+=$1} END {print sum}')

all_memory_bytes=0
memory_values=$(oc get nodes -o jsonpath='{range .items[*]}{.status.capacity.memory}{"\n"}{end}')
for memory in $memory_values; do
  memory_bytes=$(convert_to_bytes $memory)
  all_memory_bytes=$((all_memory_bytes + memory_bytes))
done

printf "%-30s %-10s %-20s\n" "NODE TYPE" "CPU CORES" "MEMORY"
echo "----------------------------------------------------------------------"
printf "%-30s %-10s %-20s\n" "All Nodes" "$all_cpu" "$(format_bytes $all_memory_bytes)"
EOF

# Make it executable
chmod +x get_all_node_resources_no_bc.sh

# Run the script
./get_all_node_resources_no_bc.sh
```

If you prefer an even simpler approach without any external commands for formatting, here's a more basic version:

```bash
cat << 'EOF' > get_node_resources_simple.sh
#!/bin/bash

echo "===== Node Resources by Type ====="

# Function to get summary for node type
get_node_summary() {
  local label=$1
  local type_name=$2
  
  # Count nodes
  local node_count=$(oc get nodes -l "$label" --no-headers | wc -l)
  
  if [ "$node_count" -eq 0 ]; then
    echo "$type_name Nodes: None found"
    return
  fi
  
  # Get total CPU
  local total_cpu=$(oc get nodes -l "$label" -o jsonpath='{range .items[*]}{.status.capacity.cpu}{"\n"}{end}' | awk '{sum+=$1} END {print sum}')
  
  # Get total memory (just show in original units)
  local memory_list=$(oc get nodes -l "$label" -o custom-columns=MEMORY:.status.capacity.memory --no-headers)
  
  echo "$type_name Nodes ($node_count):"
  echo "  - Total CPU: $total_cpu cores"
  echo "  - Memory per node: $memory_list"
  echo ""
}

# Get summaries for each node type
get_node_summary "node-role.kubernetes.io/master" "Master"
get_node_summary "node-role.kubernetes.io/infra,node-role.kubernetes.io/worker" "Infrastructure"
get_node_summary "node-role.kubernetes.io/app,node-role.kubernetes.io/worker" "Application"

# Get total cluster resources
total_nodes=$(oc get nodes --no-headers | wc -l)
total_cpu=$(oc get nodes -o jsonpath='{range .items[*]}{.status.capacity.cpu}{"\n"}{end}' | awk '{sum+=$1} END {print sum}')

echo "===== Total Cluster Resources ====="
echo "Total Nodes: $total_nodes"
echo "Total CPU Cores: $total_cpu"
echo "Memory values are shown per node due to different units"
EOF

# Make it executable
chmod +x get_node_resources_simple.sh

# Run the script
./get_node_resources_simple.sh
```

Another approach is to use Python, which is available on most systems and handles floating-point calculations well:

```bash
cat << 'EOF' > get_node_resources.py
#!/usr/bin/env python3
import subprocess
import json
import re

def run_cmd(cmd):
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
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

The Python script is the most robust but requires Python 3 to be installed. The second bash script is the simplest but provides less formatting. The first bash script with awk is a good middle ground.
