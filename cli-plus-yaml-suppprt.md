Absolutely, it's possible to support both YAML file and CLI arguments. We can extend the script by adding a function to parse the CLI arguments and then use a condition to determine whether to use the YAML file or CLI arguments.

Here's how you can achieve this:

### Updated Script with CLI and YAML Support

```bash
#!/bin/bash

# Function to display usage
usage() {
  echo "Usage: $0 -f <config_file> | -p <project_id> -c <cluster_name> -r <compute_region> -t <base_topic_name> -s <suffix> -n <notification_type> -d <disable>"
  echo "  -f <config_file>       Path to the YAML configuration file"
  echo "  -p <project_id>        GCP project ID"
  echo "  -c <cluster_name>      Cluster name"
  echo "  -r <compute_region>    Compute region"
  echo "  -t <base_topic_name>   Base topic name"
  echo "  -s <suffix>            Suffix for the topic"
  echo "  -n <notification_type> Notification type"
  echo "  -d <disable>           Disable notifications (true/false)"
  exit 1
}

# Function to log messages
log() {
  echo "$(date +"%Y-%m-%d %H:%M:%S") - $1"
}

# Function to validate yq and gcloud CLI are installed
validate_tools() {
  if ! command -v gcloud &> /dev/null; then
    log "gcloud CLI is not installed. Please install it and try again."
    exit 1
  fi
  if ! command -v yq &> /dev/null; then
    log "yq is not installed. Please install it and try again."
    exit 1
  fi
}

# Function to validate gcloud authentication
validate_gcloud_login() {
  if ! gcloud auth list --filter=status:ACTIVE --format="value(account)" > /dev/null 2>&1; then
    log "You are not logged in to gcloud. Please run 'gcloud auth login' to login."
    exit 1
  fi
}

# Function to validate the cluster exists
validate_cluster_exists() {
  local project_id=$1
  local cluster_name=$2
  local region=$3

  if ! gcloud container clusters list --project="${project_id}" --region="${region}" --filter="name=${cluster_name}" --format="value(name)" | grep -q "${cluster_name}"; then
    log "Cluster ${cluster_name} does not exist in region ${region}. Skipping this entry."
    return 1
  fi
}

# Function to create Pub/Sub topic if it does not exist
create_topic_if_not_exists() {
  local project_id=$1
  local topic_name=$2

  # Construct the full topic name
  local full_topic_name="projects/${project_id}/topics/${topic_name}"

  # Check if the topic exists
  local topic_exists
  topic_exists=$(gcloud pubsub topics list --project="${project_id}" --filter="name:${full_topic_name}" --format="value(name)")

  if [ -z "${topic_exists}" ]; then
    log "Topic does not exist. Creating topic ${topic_name}..."
    if gcloud pubsub topics create "${topic_name}" --project="${project_id}"; then
      log "Topic ${topic_name} created successfully."
    else
      log "Failed to create topic ${topic_name}. Skipping this entry."
      return 1
    fi
  else
    log "Topic ${topic_name} already exists."
  fi
}

# Function to process a single project configuration
process_project() {
  local project_id=$1
  local cluster_name=$2
  local compute_region=$3
  local base_topic_name=$4
  local suffix=$5
  local notification_type=$6
  local disable=$7

  # Validate required fields
  if [ -z "${project_id}" ] || [ -z "${cluster_name}" ] || [ -z "${compute_region}" ] || [ -z "${base_topic_name}" ] || [ -z "${notification_type}" ] && [ "${disable}" == "false" ]; then
    log "Missing required fields in configuration. Skipping this entry."
    return 1
  fi

  # Log the values being used
  log "Processing project: ${project_id}, cluster: ${cluster_name}, region: ${compute_region}"

  # Set the project in gcloud
  if ! gcloud config set project "${project_id}"; then
    log "Failed to set project ${project_id}. Skipping this entry."
    return 1
  fi

  # Validate the cluster exists
  if ! validate_cluster_exists "${project_id}" "${cluster_name}" "${compute_region}"; then
    return 1
  fi

  # Construct the full topic name with optional suffix
  if [ -n "${suffix}" ]; then
    local topic_name="${base_topic_name}-${suffix}"
  else
    local topic_name="${base_topic_name}"
  fi

  # Create the topic if not disabling notifications
  if [ "${disable}" == "false" ]; then
    if ! create_topic_if_not_exists "${project_id}" "${topic_name}"; then
      return 1
    fi
  fi

  # Enable or disable cluster notifications
  if [ "${disable}" == "true" ]; then
    if gcloud container clusters update "${cluster_name}" --region="${compute_region}" --notification-config=pubsub=DISABLED; then
      log "Cluster notifications disabled successfully for cluster ${cluster_name}."
    else
      log "Failed to disable cluster notifications for cluster ${cluster_name}. Skipping this entry."
      return 1
    fi
  else
    if gcloud container clusters update "${cluster_name}" --region="${compute_region}" --notification-config=pubsub=ENABLED,pubsub-topic=projects/"${project_id}"/topics/"${topic_name}",filter="${notification_type}"; then
      log "Cluster notifications enabled successfully for cluster ${cluster_name}."
    else
      log "Failed to enable cluster notifications for cluster ${cluster_name}. Skipping this entry."
      return 1
    fi
  fi
}

# Validate yq and gcloud CLI
validate_tools

# Validate gcloud login
validate_gcloud_login

# Parse command-line arguments
while getopts "f:p:c:r:t:s:n:d:h" opt; do
  case ${opt} in
    f)
      CONFIG_FILE=${OPTARG}
      ;;
    p)
      CLI_PROJECT_ID=${OPTARG}
      ;;
    c)
      CLI_CLUSTER_NAME=${OPTARG}
      ;;
    r)
      CLI_COMPUTE_REGION=${OPTARG}
      ;;
    t)
      CLI_BASE_TOPIC_NAME=${OPTARG}
      ;;
    s)
      CLI_SUFFIX=${OPTARG}
      ;;
    n)
      CLI_NOTIFICATION_TYPE=${OPTARG}
      ;;
    d)
      CLI_DISABLE=${OPTARG}
      ;;
    h)
      usage
      ;;
    *)
      usage
      ;;
  esac
done

# If CLI arguments are provided, use them
if [ -n "${CLI_PROJECT_ID}" ] && [ -n "${CLI_CLUSTER_NAME}" ] && [ -n "${CLI_COMPUTE_REGION}" ] && [ -n "${CLI_BASE_TOPIC_NAME}" ] && [ -n "${CLI_NOTIFICATION_TYPE}" ]; then
  process_project "${CLI_PROJECT_ID}" "${CLI_CLUSTER_NAME}" "${CLI_COMPUTE_REGION}" "${CLI_BASE_TOPIC_NAME}" "${CLI_SUFFIX}" "${CLI_NOTIFICATION_TYPE}" "${CLI_DISABLE}"
  exit 0
fi

# Validate the configuration file exists and is readable
if [ -n "${CONFIG_FILE}" ] && [ -r "${CONFIG_FILE}" ]; then
  # Read the YAML configuration file and loop over each project configuration
  project_count=$(yq eval '.projects | length' "${CONFIG_FILE}")
  for (( i=0; i<project_count; i++ )); do
    PROJECT_ID=$(yq eval ".projects[$i].project_id" "${CONFIG_FILE}")
    CLUSTER_NAME=$(yq eval ".projects[$i].cluster_name" "${CONFIG_FILE}")
    COMPUTE_REGION=$(yq eval ".projects[$i].compute_region" "${CONFIG_FILE}")
    BASE_TOPIC_NAME=$(yq eval ".projects[$i].base_topic_name" "${CONFIG_FILE}")
    SUFFIX=$(yq eval ".projects[$i].suffix" "${CONFIG_FILE}")
    NOTIFICATION_TYPE=$(yq eval ".projects[$i].notification_type" "${CONFIG_FILE}")
    DISABLE=$(yq eval ".projects[$i].disable" "${CONFIG_FILE}")

    process_project "${PROJECT_ID}" "${CLUSTER_NAME}" "${COMPUTE_REGION}" "${BASE_TOPIC_NAME}" "${SUFFIX}" "${NOTIFICATION_TYPE}" "${DISABLE}"
  done
else
  log "Configuration file not provided or not readable, and no CLI arguments provided."
  usage
fi
```

### Explanation

1. **Argument Parsing**: Added support for CLI arguments `-p`, `-c`, `-r`, `-t`, `-s`, `-n`, `-d` to allow passing project configuration directly.
2. **Function `process_project`**: Handles the processing of each project, whether from CLI or YAML.
3. **Conditional Execution**: Checks if CLI arguments are provided; if so, processes them. Otherwise, it falls back to using the YAML configuration file.

### Example `projects-notifications.yaml`

```yaml
projects:
  - project_id: "your-project-id-1"
    cluster_name: "your-cl

uster-name-1"
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

### Running the Script

1. **Using CLI Arguments**:

   ```bash
   ./enable_disable_cluster_notifications.sh -p your-project-id -c your-cluster-name -r your-compute-region -t your-base-topic-name -s notifier -n "UpgradeEvent|SecurityBulletinEvent" -d false
   ```

2. **Using YAML Configuration File**:

   ```bash
   ./enable_disable_cluster_notifications.sh -f projects-notifications.yaml
   ```

This script should now correctly handle both CLI arguments and YAML file input, allowing flexible use according to your needs.
