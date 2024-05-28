The `gcloud` CLI does not natively support a `--dry-run` option for simulating actions without executing them. Instead, you can simulate a dry run in the script by logging the commands that would be executed without actually running them. Here's how you can handle it:

### Enhanced Script with Simulated Dry Run

```bash
#!/bin/bash

# Function to display usage
usage() {
  echo "Usage: $0 -p <project_id> -c <cluster_name> -r <compute_region> -t <base_topic_name> [-s <suffix>] -f <notification_type> [-d] [-h] [--dry-run]"
  echo "  -p <project_id>       Google Cloud Project ID"
  echo "  -c <cluster_name>     Name of the GKE cluster"
  echo "  -r <compute_region>   Compute region for the cluster"
  echo "  -t <base_topic_name>  Base name of the Pub/Sub topic"
  echo "  -s <suffix>           Suffix to be added to the topic name (optional)"
  echo "  -f <notification_type> Notification types (pipe-delimited list)"
  echo "  -d                    Disable notifications (if set)"
  echo "  --dry-run             Display the actions without executing"
  echo "  -h                    Display this help message"
  exit 1
}

# Function to log messages
log() {
  echo "$(date +"%Y-%m-%d %H:%M:%S") - $1"
}

# Function to validate gcloud CLI is installed
validate_gcloud_cli() {
  if ! command -v gcloud &> /dev/null; then
    log "gcloud CLI is not installed. Please install it and try again."
    exit 1
  fi
}

# Function to validate gcloud authentication
validate_gcloud_login() {
  gcloud auth list --filter=status:ACTIVE --format="value(account)" > /dev/null 2>&1
  if [ $? -ne 0 ]; then
    log "You are not logged in to gcloud. Please run 'gcloud auth login' to login."
    exit 1
  fi
}

# Function to validate the cluster exists
validate_cluster_exists() {
  local cluster_name=$1
  local compute_region=$2

  gcloud container clusters list --region=${compute_region} --filter="name=${cluster_name}" --format="value(name)" | grep -q "${cluster_name}"
  if [ $? -ne 0 ]; then
    log "Cluster ${cluster_name} does not exist in region ${compute_region}. Please check the cluster name and region."
    exit 1
  fi
}

# Function to create Pub/Sub topic if it does not exist
create_topic_if_not_exists() {
  local project_id=$1
  local topic_name=$2

  # Construct the full topic name
  local full_topic_name="projects/${project_id}/topics/${topic_name}"

  # Check if the topic exists
  local topic_exists=$(gcloud pubsub topics list --filter="name:${full_topic_name}" --format="value(name)")

  if [ -z "$topic_exists" ]; then
    log "Topic does not exist. Creating topic ${topic_name}..."
    if [ $DRY_RUN -eq 0 ]; then
      gcloud pubsub topics create ${topic_name} --project=${project_id}
      if [ $? -eq 0 ]; then
        log "Topic ${topic_name} created successfully."
      else
        log "Failed to create topic ${topic_name}."
        exit 1
      fi
    else
      log "Dry run: gcloud pubsub topics create ${topic_name} --project=${project_id}"
    fi
  else
    log "Topic ${topic_name} already exists."
  fi
}

# Parse command-line arguments
DISABLE_NOTIFICATIONS=0
DRY_RUN=0
SUFFIX=""
while getopts "p:c:r:t:s:f:dh-:" opt; do
  case ${opt} in
    p)
      PROJECT_ID=${OPTARG}
      ;;
    c)
      CLUSTER_NAME=${OPTARG}
      ;;
    r)
      COMPUTE_REGION=${OPTARG}
      ;;
    t)
      BASE_TOPIC_NAME=${OPTARG}
      ;;
    s)
      SUFFIX=${OPTARG}
      ;;
    f)
      NOTIFICATION_TYPE=${OPTARG}
      ;;
    d)
      DISABLE_NOTIFICATIONS=1
      ;;
    h)
      usage
      ;;
    -)
      case "${OPTARG}" in
        dry-run)
          DRY_RUN=1
          ;;
        *)
          usage
          ;;
      esac
      ;;
    *)
      usage
      ;;
  esac
done

# Validate required arguments
if [ -z "${PROJECT_ID}" ] || [ -z "${CLUSTER_NAME}" ] || [ -z "${COMPUTE_REGION}" ] || [ -z "${BASE_TOPIC_NAME}" ] || [ -z "${NOTIFICATION_TYPE}" ] && [ ${DISABLE_NOTIFICATIONS} -eq 0 ]; then
  usage
fi

# Validate gcloud CLI
validate_gcloud_cli

# Validate gcloud login
validate_gcloud_login

# Set the project in gcloud
if [ $DRY_RUN -eq 0 ]; then
  gcloud config set project ${PROJECT_ID}
  if [ $? -ne 0 ]; then
    log "Failed to set project ${PROJECT_ID}. Please ensure the project ID is correct and you have the necessary permissions."
    exit 1
  fi
else
  log "Dry run: gcloud config set project ${PROJECT_ID}"
fi

# Validate the cluster exists
validate_cluster_exists ${CLUSTER_NAME} ${COMPUTE_REGION}

# Construct the full topic name with optional suffix
if [ -n "${SUFFIX}" ]; then
  TOPIC_NAME="${BASE_TOPIC_NAME}-${SUFFIX}"
else
  TOPIC_NAME="${BASE_TOPIC_NAME}"
fi

# Create the topic if not disabling notifications
if [ ${DISABLE_NOTIFICATIONS} -eq 0 ]; then
  create_topic_if_not_exists ${PROJECT_ID} ${TOPIC_NAME}
fi

# Enable or disable cluster notifications
if [ ${DISABLE_NOTIFICATIONS} -eq 1 ]; then
  if [ $DRY_RUN -eq 0 ]; then
    gcloud container clusters update ${CLUSTER_NAME} \
      --region=${COMPUTE_REGION} \
      --notification-config=pubsub=DISABLED
    if [ $? -eq 0 ]; then
      log "Cluster notifications disabled successfully for cluster ${CLUSTER_NAME}."
    else
      log "Failed to disable cluster notifications for cluster ${CLUSTER_NAME}."
      exit 1
    fi
  else
    log "Dry run: gcloud container clusters update ${CLUSTER_NAME} --region=${COMPUTE_REGION} --notification-config=pubsub=DISABLED"
  fi
else
  if [ $DRY_RUN -eq 0 ]; then
    gcloud container clusters update ${CLUSTER_NAME} \
      --region=${COMPUTE_REGION} \
      --notification-config=pubsub=ENABLED,pubsub-topic=projects/${PROJECT_ID}/topics/${TOPIC_NAME},filter=${NOTIFICATION_TYPE}
    if [ $? -eq 0 ]; then
      log "Cluster notifications enabled successfully for cluster ${CLUSTER_NAME}."
    else
      log "Failed to enable cluster notifications for cluster ${CLUSTER_NAME}."
      exit 1
    fi
  else
    log "Dry run: gcloud container clusters update ${CLUSTER_NAME} --region=${COMPUTE_REGION} --notification-config=pubsub=ENABLED,pubsub-topic=projects/${PROJECT_ID}/topics/${TOPIC_NAME},filter=${NOTIFICATION_TYPE}"
  fi
fi
```

### Explanation of Changes

1. **Removed Project ID as Suffix**: The project ID is no longer appended to the topic name.
2. **Optional Suffix**: The suffix is now optional and will only be appended if provided.
3. **Simulated Dry Run**: The script now supports a simulated dry run, logging the commands that would be executed without actually running them.

### Making the Script Executable

Make sure the script is executable:

```bash
chmod +x enable_disable_cluster_notifications.sh
```

### Running the Script

Run the script to enable notifications with an optional suffix:

```bash
./enable_disable_cluster_notifications.sh -p your-project-id -c your-cluster-name -r your-compute-region -t your-base-topic-name -s notifier -f "UpgradeEvent|SecurityBulletinEvent"
```

Run the script to enable notifications without a suffix:

```bash
./enable_disable_cluster_notifications.sh -p your-project-id -c your-cluster-name -r your-compute-region -t your-base-topic-name -f "UpgradeEvent|SecurityBulletinEvent"
```

Run the script to disable notifications:

```bash
./enable_disable_cluster_notifications.sh -p your-project-id -c your-cluster-name -r your-compute-region -d
```

Run the script in dry run mode to see what actions will be taken:

```bash
./enable_disable_cluster_notifications.sh -p your-project-id -c your-cluster-name -r your-compute-region -t your-base-topic-name -s notifier -f "UpgradeEvent|SecurityBulletinEvent" --dry-run
```

This script now supports an optional suffix for the topic name without appending the project ID and correctly handles simulated dry run operations.
