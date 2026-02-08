
#!/bin/bash

# ===============================
# CONFIG
# ===============================
TENANCY_OCID="Fill the OCID"
REGION="Fill the region code"
MAX_DEPTH=6
TIMESTAMP=$(date +%Y%m%d_%H%M)
OUTPUT_FILE="oci_inventory_${REGION}_${TIMESTAMP}.csv"

export OCI_CLI_REGION=$REGION
export OCI_CLI_REQUEST_TIMEOUT=180

# ===============================
# CSV HEADER
# ===============================
# Added "Block_Volume_Sizes_GB" and "Total_Block_Storage_GB"
echo "Hostname,Compartment,Depth,Status,Shape,OCPU,Memory_GB,Image,Private_IP,Public_IP,Subnet,Boot_Size_GB,Block_Sizes_GB,Total_Block_GB,Instance_OCID" > "$OUTPUT_FILE"

# ===============================
# FUNCTION: RECURSIVE SCAN
# ===============================
scan_compartment() {
  local COMP_ID="$1"
  local COMP_NAME="$2"
  local DEPTH="$3"

  if [ "$DEPTH" -gt "$MAX_DEPTH" ]; then return; fi

  echo "Scanning: $COMP_NAME (Level $DEPTH)..."

  oci compute instance list --compartment-id "$COMP_ID" --all 2>/dev/null | jq -c '.data[]?' | while read -r INSTANCE; do

      INSTANCE_OCID=$(echo "$INSTANCE" | jq -r '.id')
      HOSTNAME=$(echo "$INSTANCE" | jq -r '."display-name"')
      STATUS=$(echo "$INSTANCE" | jq -r '."lifecycle-state"')
      SHAPE=$(echo "$INSTANCE" | jq -r '.shape')
      IMAGE_OCID=$(echo "$INSTANCE" | jq -r '."image-id"')
      AD=$(echo "$INSTANCE" | jq -r '."availability-domain"')
      OCPU=$(echo "$INSTANCE" | jq -r '."shape-config".ocpus // 1')
      MEMORY=$(echo "$INSTANCE" | jq -r '."shape-config"."memory-in-gbs" // "N/A"')

      # ---- Fetch Image Details ----
      IMAGE_NAME=$(oci compute image get --image-id "$IMAGE_OCID" --query 'data."display-name"' --raw-output 2>/dev/null || echo "Deleted/Custom")

      # ---- Network details (Primary VNIC) ----
      VNIC_ID=$(oci compute vnic-attachment list --compartment-id "$COMP_ID" --instance-id "$INSTANCE_OCID" --query 'data[0]."vnic-id"' --raw-output 2>/dev/null)
      if [[ -n "$VNIC_ID" && "$VNIC_ID" != "null" ]]; then
        VNIC=$(oci network vnic get --vnic-id "$VNIC_ID")
        PRIVATE_IP=$(echo "$VNIC" | jq -r '.data."private-ip"')
        PUBLIC_IP=$(echo "$VNIC" | jq -r '.data."public-ip" // "NA"')
        SUBNET_NAME=$(oci network subnet get --subnet-id "$(echo "$VNIC" | jq -r '.data."subnet-id"')" --query 'data."display-name"' --raw-output 2>/dev/null)
      else
        PRIVATE_IP="NA"; PUBLIC_IP="NA"; SUBNET_NAME="NA"
      fi

      # ---- Boot Volume Size ----
      BOOT_VOL_ID=$(oci compute boot-volume-attachment list --availability-domain "$AD" --compartment-id "$COMP_ID" --instance-id "$INSTANCE_OCID" --query 'data[0]."boot-volume-id"' --raw-output 2>/dev/null)
      BOOT_SIZE=$(oci bv boot-volume get --boot-volume-id "$BOOT_VOL_ID" --query 'data."size-in-gbs"' --raw-output 2>/dev/null || echo "0")

      # ---- Block Volume Details (Data Disks) ----
      # This looks for all non-boot volumes attached to the instance
      BLOCK_ATTACHMENTS=$(oci compute volume-attachment list --compartment-id "$COMP_ID" --instance-id "$INSTANCE_OCID" 2>/dev/null | jq -c '.data[]?')
      
      BLOCK_SIZES=""
      TOTAL_BLOCK_SIZE=0
      
      if [[ -n "$BLOCK_ATTACHMENTS" ]]; then
        while read -r ATTACHMENT; do
          VOL_ID=$(echo "$ATTACHMENT" | jq -r '."volume-id"')
          SIZE=$(oci bv volume get --volume-id "$VOL_ID" --query 'data."size-in-gbs"' --raw-output 2>/dev/null)
          
          BLOCK_SIZES="${BLOCK_SIZES}${SIZE};"
          TOTAL_BLOCK_SIZE=$((TOTAL_BLOCK_SIZE + SIZE))
        done <<< "$BLOCK_ATTACHMENTS"
      fi
      
      # Clean up string if empty
      [[ -z "$BLOCK_SIZES" ]] && BLOCK_SIZES="0" || BLOCK_SIZES="${BLOCK_SIZES%;}"

      # Append results
      echo "$HOSTNAME,$COMP_NAME,$DEPTH,$STATUS,$SHAPE,$OCPU,$MEMORY,$IMAGE_NAME,$PRIVATE_IP,$PUBLIC_IP,$SUBNET_NAME,$BOOT_SIZE,$BLOCK_SIZES,$TOTAL_BLOCK_SIZE,$INSTANCE_OCID" >> "$OUTPUT_FILE"
  done

  # ---- Recurse child compartments ----
  oci iam compartment list --compartment-id "$COMP_ID" --all --access-level ANY 2>/dev/null | jq -c '.data[]?' | while read -r CHILD; do
      scan_compartment "$(echo "$CHILD" | jq -r '.id')" "$(echo "$CHILD" | jq -r '.name')" $((DEPTH + 1))
  done
}

# ===============================
# START
# ===============================
ROOT_NAME=$(oci iam tenancy get --tenancy-id "$TENANCY_OCID" --query 'data.name' --raw-output 2>/dev/null || echo "Tenancy_Root")
scan_compartment "$TENANCY_OCID" "$ROOT_NAME" 1

echo "------------------------------------------------"
echo "✅ Inventory completed!"
echo "📄 File: $OUTPUT_FILE"
