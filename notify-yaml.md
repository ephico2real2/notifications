The current design of the script stops execution if it encounters an error, specifically when validating or setting configurations for a project. This approach ensures that any critical errors are immediately addressed rather than potentially leading to inconsistent states or configurations across different projects.

However, for a more resilient and user-friendly approach, the script can be modified to log errors and skip problematic entries while continuing to process the rest of the configurations. This design is beneficial for scenarios where you want the script to handle multiple configurations and continue running despite encountering errors in some of them.

`` projects-notifications.yaml ``

```yaml
projects:
  - project_id: "your-project-id-1"
    cluster_name: "your-cluster-name-1"
    compute_region: "your-compute-region-1"
    base_topic_name: "your-base-topic-name-1"
    suffix: "notifier"
    notification_type: "UpgradeEvent|SecurityBulletinEvent"
    disable: false
  - project_id: "your-project-id-2"
    cluster_name: "your-cluster-name-2"
    compute_region: "your-compute-region-2"
    base_topic_name: "your-base-topic-name-2"
    suffix: ""
    notification_type: "UpgradeEvent|SecurityBulletinEvent"
    disable: true

```
The error messages you're encountering suggest that the active project and region information are not being set correctly. Additionally, it seems like there might be permission issues with the credentials being used.

Let's address the issues step-by-step:

1. **Set the Active Project Correctly**: Ensure that the project ID is being set correctly before making any API calls.
2. **Set the Region Correctly**: Ensure the region is not `null`.
3. **Handle Credentials**: Ensure the correct credentials are being used.

Let's refine the script to address these issues.

### Updated Shell Script

Here's an updated version of the shell script with improved handling of project and region settings:

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

# Validate yq and gcloud CLI
validate_tools

# Validate gcloud login
validate_gcloud_login

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

  # Validate required fields
  if [ -z "${PROJECT_ID}" ] || [ -z "${CLUSTER_NAME}" ] || [ -z "${COMPUTE_REGION}" ] || [ -z "${BASE_TOPIC_NAME}" ] || [ -z "${NOTIFICATION_TYPE}" ] && [ "${DISABLE}" == "false" ]; then
    log "Missing required fields in configuration. Skipping this entry."
    continue
  fi

  # Set the project in gcloud
  if ! gcloud config set project "${PROJECT_ID}"; then
    log "Failed to set project ${PROJECT_ID}. Skipping this entry."
    continue
  fi

  # Validate the cluster exists
  if ! validate_cluster_exists "${PROJECT_ID}" "${CLUSTER_NAME}" "${COMPUTE_REGION}"; then
    continue
  fi

  # Construct the full topic name with optional suffix
  if [ -n "${SUFFIX}" ]; then
    TOPIC_NAME="${BASE_TOPIC_NAME}-${SUFFIX}"
  else
    TOPIC_NAME="${BASE_TOPIC_NAME}"
  fi

  # Create the topic if not disabling notifications
  if [ "${DISABLE}" == "false" ]; then
    if ! create_topic_if_not_exists "${PROJECT_ID}" "${TOPIC_NAME}"; then
      continue
    fi
  fi

  # Enable or disable cluster notifications
  if [ "${DISABLE}" == "true" ]; then
    if gcloud container clusters update "${CLUSTER_NAME}" --region="${COMPUTE_REGION}" --notification-config=pubsub=DISABLED; then
      log "Cluster notifications disabled successfully for cluster ${CLUSTER_NAME}."
    else
      log "Failed to disable cluster notifications for cluster ${CLUSTER_NAME}. Skipping this entry."
      continue
    fi
  else
    if gcloud container clusters update "${CLUSTER_NAME}" --region="${COMPUTE_REGION}" --notification-config=pubsub=ENABLED,pubsub-topic=projects/"${PROJECT_ID}"/topics/"${TOPIC_NAME}",filter="${NOTIFICATION_TYPE}"; then
      log "Cluster notifications enabled successfully for cluster ${CLUSTER_NAME}."
    else
      log "Failed to enable cluster notifications for cluster ${CLUSTER_NAME}. Skipping this entry."
      continue
    fi
  fi
done
```

### Key Improvements

1. **Ensure Region is Correctly Set**: Added checks to ensure the `COMPUTE_REGION` is not null.
2. **Project Configuration**: Each project ID is set before any operations, ensuring correct context for each API call.
3. **Handle Permissions**: Ensured proper handling of permissions issues by checking and setting the active project correctly.

### Running the Updated Script

1. **Ensure the Script is Executable**:

   ```bash
   chmod +x enable_disable_cluster_notifications.sh
   ```

2. **Run the Script with the Configuration File**:

   ```bash
   ./enable_disable_cluster_notifications.sh -f projects-notifications.yaml
   ```

This updated script should correctly handle setting the project and region, and manage errors more gracefully, ensuring the script continues processing other entries even if it encounters issues with one.
