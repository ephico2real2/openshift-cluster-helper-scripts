Here's how to check memory for worker nodes and calculate the total sum:

```bash
# Get memory resources for all worker nodes
oc get nodes -l node-role.kubernetes.io/worker -o custom-columns=NAME:.metadata.name,MEMORY:.status.capacity.memory

# To get the sum of all memory across worker nodes (note: values will be in different units like Ki, Mi, Gi)
oc get nodes -l node-role.kubernetes.io/worker -o jsonpath='{range .items[*]}{.status.capacity.memory}{"\n"}{end}'
```

Since memory values often come with units (Ki, Mi, Gi), we need to normalize them for proper addition. Here's a script that handles this and provides both individual node memory and the total:

```bash
cat << 'EOF' > get_memory_resources.sh
#!/bin/bash
echo "===== Worker Node Memory Resources ====="
total_memory_bytes=0

# Get worker nodes
worker_nodes=$(oc get nodes -l node-role.kubernetes.io/worker -o name)

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
    *) echo $number ;;  # Default case if no unit
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

# Iterate through each worker node
for node in $worker_nodes; do
  node_name=$(echo $node | cut -d '/' -f 2)
  memory=$(oc get node $node_name -o jsonpath='{.status.capacity.memory}')
  memory_bytes=$(convert_to_bytes $memory)
  total_memory_bytes=$((total_memory_bytes + memory_bytes))
  
  # Display node memory in original format
  echo "Node: $node_name - Memory: $memory ($(format_bytes $memory_bytes))"
done

echo "============================="
echo "Total memory across all worker nodes: $(format_bytes $total_memory_bytes)"
EOF

# Make it executable
chmod +x get_memory_resources.sh

# Run the script
./get_memory_resources.sh
```

If you want to create a combined script that shows both CPU and memory in one go:

```bash
cat << 'EOF' > get_cluster_resources.sh
#!/bin/bash
echo "===== Worker Node Resources ====="
total_cpu=0
total_memory_bytes=0

# Get worker nodes
worker_nodes=$(oc get nodes -l node-role.kubernetes.io/worker -o name)

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

printf "%-30s %-10s %-20s\n" "NODE NAME" "CPU CORES" "MEMORY"
echo "----------------------------------------------------------------------"

# Iterate through each worker node
for node in $worker_nodes; do
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
EOF

# Make it executable
chmod +x get_cluster_resources.sh

# Run the script
./get_cluster_resources.sh
```
