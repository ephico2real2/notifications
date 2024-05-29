Sure, to troubleshoot the parsing of individual fields in the `projects-notifications.yaml` file using `yq`, you can add some debug output to see exactly what values are being extracted.

Here's a simplified script to print out each field for debugging purposes:

`` projects-notifications.yaml ``
```yaml
projects:
  - project_id: "your-project-id-1"
    cluster_name: "your-cluster-name-1"
    compute_region: "us-central1"
    base_topic_name: "your-base-topic-name-1"
    suffix: "notifier"
    notification_type: "UpgradeEvent|SecurityBulletinEvent"
    disable: false

  - project_id: "your-project-id-2"
    cluster_name: "your-cluster-name-2"
    compute_region: "europe-west1"
    base_topic_name: "your-base-topic-name-2"
    suffix: ""
    notification_type: "UpgradeEvent|SecurityBulletinEvent"
    disable: true

  - project_id: "your-project-id-3"
    cluster_name: "your-cluster-name-3"
    compute_region: "asia-east1"
    base_topic_name: "your-base-topic-name-3"
    suffix: "alerts"
    notification_type: "NodePoolUpdateEvent"
    disable: false
```

### Simplified Debugging Script

```bash
#!/bin/bash

# Function to display usage
usage() {
  echo "Usage: $0 -f <config_file>"
  echo "  -f <config_file>      Path to the YAML configuration file"
  exit 1
}

# Function to log messages
log() {
  echo "$(date +"%Y-%m-%d %H:%M:%S") - $1"
}

# Validate yq is installed
if ! command -v yq &> /dev/null; then
  log "yq is not installed. Please install it and try again."
  exit 1
fi

# Parse command-line arguments
while getopts "f:h" opt; do
  case ${opt} in
    f)
      CONFIG_FILE=${OPTARG}
      ;;
    h)
      usage
      ;;
    *)
      usage
      ;;
  esac
done

# Validate required arguments
if [ -z "${CONFIG_FILE}" ]; then
  usage
fi

# Validate the configuration file exists and is readable
if [ ! -r "${CONFIG_FILE}" ]; then
  log "Configuration file ${CONFIG_FILE} does not exist or is not readable."
  exit 1
fi

# Read the YAML configuration file and loop over each project configuration
yq eval '.projects[]' "${CONFIG_FILE}" | while read -r project; do
  PROJECT_ID=$(echo "${project}" | yq eval '.project_id' -)
  CLUSTER_NAME=$(echo "${project}" | yq eval '.cluster_name' -)
  COMPUTE_REGION=$(echo "${project}" | yq eval '.compute_region' -)
  BASE_TOPIC_NAME=$(echo "${project}" | yq eval '.base_topic_name' -)
  SUFFIX=$(echo "${project}" | yq eval '.suffix' -)
  NOTIFICATION_TYPE=$(echo "${project}" | yq eval '.notification_type' -)
  DISABLE=$(echo "${project}" | yq eval '.disable' -)

  # Print out the values for debugging
  log "PROJECT_ID: ${PROJECT_ID}"
  log "CLUSTER_NAME: ${CLUSTER_NAME}"
  log "COMPUTE_REGION: ${COMPUTE_REGION}"
  log "BASE_TOPIC_NAME: ${BASE_TOPIC_NAME}"
  log "SUFFIX: ${SUFFIX}"
  log "NOTIFICATION_TYPE: ${NOTIFICATION_TYPE}"
  log "DISABLE: ${DISABLE}"
done
```

### Running the Debugging Script

1. **Ensure the Script is Executable**:

   ```bash
   chmod +x debug_yaml_parsing.sh
   ```

2. **Run the Script with the Configuration File**:

   ```bash
   ./debug_yaml_parsing.sh -f projects-notifications.yaml
   ```

### Expected Output

Running this script should output the values of each field extracted from the YAML file, like this:

```bash
2024-05-29 12:34:56 - PROJECT_ID: your-project-id-1
2024-05-29 12:34:56 - CLUSTER_NAME: your-cluster-name-1
2024-05-29 12:34:56 - COMPUTE_REGION: us-central1
2024-05-29 12:34:56 - BASE_TOPIC_NAME: your-base-topic-name-1
2024-05-29 12:34:56 - SUFFIX: notifier
2024-05-29 12:34:56 - NOTIFICATION_TYPE: UpgradeEvent|SecurityBulletinEvent
2024-05-29 12:34:56 - DISABLE: false
```

This will help you verify that the fields are being parsed correctly from the YAML file. If any values are `null` or missing, you can then check the structure of your YAML file to ensure it matches the expected format.
