cat << 'EOF' > get_all_node_resources.sh
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

# Function to format bytes to human-readable format
format_bytes() {
  local bytes=$1
  if [ $bytes -ge 1073741824 ]; then
    echo $(echo "scale=2; $bytes/1073741824" | bc)" GiB"
  elif [ $bytes -ge 1048576 ]; then
    echo $(echo "scale=2; $bytes/1048576" | bc)" MiB"
  elif [ $bytes -ge 1024 ]; then
    echo $(echo "scale=2; $bytes/1024" | bc)" KiB"
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
chmod +x get_all_node_resources.sh

# Run the script
./get_all_node_resources.sh
