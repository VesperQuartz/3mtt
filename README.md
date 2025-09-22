#!/bin/bash

#==============================================================================
# DataWise Solutions - AWS Infrastructure Deployment Script
# Purpose: Automate EC2 and S3 setup for Data Science environments
# Author: DataWise Solutions DevOps Team
# Version: 1.0
#==============================================================================

# Color codes for output formatting
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Global arrays to track created resources
declare -a CREATED_INSTANCES=()
declare -a CREATED_BUCKETS=()
declare -a CREATED_SECURITY_GROUPS=()
declare -a CREATED_KEY_PAIRS=()

# Environment Variables (can be overridden)
export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION:-"us-east-1"}
export DEFAULT_INSTANCE_TYPE=${DEFAULT_INSTANCE_TYPE:-"t3.medium"}
export DEFAULT_AMI_ID=${DEFAULT_AMI_ID:-"ami-0c02fb55956c7d316"}  # Amazon Linux 2
export PROJECT_NAME=${PROJECT_NAME:-"datawise"}
export ENVIRONMENT=${ENVIRONMENT:-"dev"}

#==============================================================================
# UTILITY FUNCTIONS
#==============================================================================

# Function to print formatted messages
print_message() {
    local level=$1
    local message=$2
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    case $level in
        "INFO")
            echo -e "${GREEN}[INFO]${NC} [$timestamp] $message"
            ;;
        "WARN")
            echo -e "${YELLOW}[WARN]${NC} [$timestamp] $message"
            ;;
        "ERROR")
            echo -e "${RED}[ERROR]${NC} [$timestamp] $message"
            ;;
        "DEBUG")
            echo -e "${BLUE}[DEBUG]${NC} [$timestamp] $message"
            ;;
    esac
}

# Function to check if required tools are installed
check_prerequisites() {
    print_message "INFO" "Checking prerequisites..."

    local tools=("aws" "jq")
    local missing_tools=()

    for tool in "${tools[@]}"; do
        if ! command -v "$tool" &> /dev/null; then
            missing_tools+=("$tool")
        fi
    done

    if [ ${#missing_tools[@]} -ne 0 ]; then
        print_message "ERROR" "Missing required tools: ${missing_tools[*]}"
        print_message "INFO" "Please install missing tools and try again"
        return 1
    fi

    # Check AWS credentials
    if ! aws sts get-caller-identity &> /dev/null; then
        print_message "ERROR" "AWS credentials not configured or invalid"
        print_message "INFO" "Please run 'aws configure' or set AWS environment variables"
        return 1
    fi

    print_message "INFO" "Prerequisites check completed successfully"
    return 0
}

# Function to validate input parameters
validate_parameters() {
    local instance_type=$1
    local bucket_name=$2

    # Validate instance type format
    if [[ ! $instance_type =~ ^[a-z][0-9]+\.[a-z]+$ ]]; then
        print_message "ERROR" "Invalid instance type format: $instance_type"
        return 1
    fi

    # Validate S3 bucket name
    if [[ ! $bucket_name =~ ^[a-z0-9][a-z0-9-]*[a-z0-9]$ ]] || [ ${#bucket_name} -lt 3 ] || [ ${#bucket_name} -gt 63 ]; then
        print_message "ERROR" "Invalid S3 bucket name: $bucket_name"
        print_message "INFO" "Bucket name must be 3-63 characters, lowercase, alphanumeric and hyphens only"
        return 1
    fi

    return 0
}

#==============================================================================
# AWS RESOURCE CREATION FUNCTIONS
#==============================================================================

# Function to create security group
create_security_group() {
    local sg_name="$PROJECT_NAME-$ENVIRONMENT-sg"
    local description="Security group for DataWise $ENVIRONMENT environment"

    print_message "INFO" "Creating security group: $sg_name"

    local sg_id=$(aws ec2 create-security-group \
        --group-name "$sg_name" \
        --description "$description" \
        --query 'GroupId' \
        --output text 2>/dev/null)

    if [ $? -ne 0 ] || [ -z "$sg_id" ]; then
        print_message "ERROR" "Failed to create security group"
        return 1
    fi

    # Add to tracking array
    CREATED_SECURITY_GROUPS+=("$sg_id")

    # Add inbound rules for SSH and HTTP/HTTPS
    aws ec2 authorize-security-group-ingress \
        --group-id "$sg_id" \
        --protocol tcp \
        --port 22 \
        --cidr 0.0.0.0/0 &> /dev/null

    aws ec2 authorize-security-group-ingress \
        --group-id "$sg_id" \
        --protocol tcp \
        --port 80 \
        --cidr 0.0.0.0/0 &> /dev/null

    aws ec2 authorize-security-group-ingress \
        --group-id "$sg_id" \
        --protocol tcp \
        --port 443 \
        --cidr 0.0.0.0/0 &> /dev/null

    # Add Jupyter notebook port
    aws ec2 authorize-security-group-ingress \
        --group-id "$sg_id" \
        --protocol tcp \
        --port 8888 \
        --cidr 0.0.0.0/0 &> /dev/null

    print_message "INFO" "Security group created successfully: $sg_id"
    echo "$sg_id"
}

# Function to create key pair
create_key_pair() {
    local key_name="$PROJECT_NAME-$ENVIRONMENT-key"
    local key_file="$key_name.pem"

    print_message "INFO" "Creating key pair: $key_name"

    # Check if key already exists
    if aws ec2 describe-key-pairs --key-names "$key_name" &> /dev/null; then
        print_message "WARN" "Key pair $key_name already exists, using existing key"
        echo "$key_name"
        return 0
    fi

    # Create new key pair
    aws ec2 create-key-pair \
        --key-name "$key_name" \
        --query 'KeyMaterial' \
        --output text > "$key_file" 2>/dev/null

    if [ $? -ne 0 ]; then
        print_message "ERROR" "Failed to create key pair"
        return 1
    fi

    chmod 400 "$key_file"
    CREATED_KEY_PAIRS+=("$key_name")

    print_message "INFO" "Key pair created successfully: $key_name (saved as $key_file)"
    echo "$key_name"
}

# Function to create S3 bucket
create_s3_bucket() {
    local bucket_name=$1
    local region=$2

    print_message "INFO" "Creating S3 bucket: $bucket_name in region: $region"

    # Create bucket with region-specific configuration
    if [ "$region" = "us-east-1" ]; then
        aws s3api create-bucket --bucket "$bucket_name" &> /dev/null
    else
        aws s3api create-bucket \
            --bucket "$bucket_name" \
            --region "$region" \
            --create-bucket-configuration LocationConstraint="$region" &> /dev/null
    fi

    if [ $? -ne 0 ]; then
        print_message "ERROR" "Failed to create S3 bucket: $bucket_name"
        return 1
    fi

    # Add to tracking array
    CREATED_BUCKETS+=("$bucket_name")

    # Enable versioning
    aws s3api put-bucket-versioning \
        --bucket "$bucket_name" \
        --versioning-configuration Status=Enabled &> /dev/null

    # Block public access for security
    aws s3api put-public-access-block \
        --bucket "$bucket_name" \
        --public-access-block-configuration \
        "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true" &> /dev/null

    # Create folder structure for data science project
    local folders=("raw-data/" "processed-data/" "models/" "reports/" "scripts/")
    for folder in "${folders[@]}"; do
        aws s3api put-object --bucket "$bucket_name" --key "$folder" &> /dev/null
    done

    print_message "INFO" "S3 bucket created successfully: $bucket_name"
    return 0
}

# Function to create EC2 instance
create_ec2_instance() {
    local instance_type=$1
    local security_group_id=$2
    local key_name=$3
    local instance_name="$PROJECT_NAME-$ENVIRONMENT-instance"

    print_message "INFO" "Creating EC2 instance: $instance_name (Type: $instance_type)"

    # User data script for instance initialization
    local user_data=$(cat <<EOF
#!/bin/bash
yum update -y
yum install -y python3 python3-pip git htop
pip3 install --upgrade pip
pip3 install jupyter pandas numpy scikit-learn boto3 awscli
amazon-linux-extras install docker -y
systemctl start docker
systemctl enable docker
usermod -a -G docker ec2-user

# Create Jupyter config
mkdir -p /home/ec2-user/.jupyter
cat > /home/ec2-user/.jupyter/jupyter_notebook_config.py << 'EOL'
c.NotebookApp.ip = '0.0.0.0'
c.NotebookApp.port = 8888
c.NotebookApp.open_browser = False
c.NotebookApp.token = 'datawise123'
EOL
chown -R ec2-user:ec2-user /home/ec2-user/.jupyter

# Create startup script
cat > /home/ec2-user/start_jupyter.sh << 'EOL'
#!/bin/bash
cd /home/ec2-user
nohup jupyter notebook --config=/home/ec2-user/.jupyter/jupyter_notebook_config.py > jupyter.log 2>&1 &
EOL
chmod +x /home/ec2-user/start_jupyter.sh
chown ec2-user:ec2-user /home/ec2-user/start_jupyter.sh
EOF
)

    # Launch instance
    local instance_id=$(aws ec2 run-instances \
        --image-id "$DEFAULT_AMI_ID" \
        --count 1 \
        --instance-type "$instance_type" \
        --key-name "$key_name" \
        --security-group-ids "$security_group_id" \
        --user-data "$user_data" \
        --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$instance_name},{Key=Project,Value=$PROJECT_NAME},{Key=Environment,Value=$ENVIRONMENT}]" \
        --query 'Instances[0].InstanceId' \
        --output text 2>/dev/null)

    if [ $? -ne 0 ] || [ -z "$instance_id" ]; then
        print_message "ERROR" "Failed to create EC2 instance"
        return 1
    fi

    # Add to tracking array
    CREATED_INSTANCES+=("$instance_id")

    print_message "INFO" "EC2 instance created successfully: $instance_id"
    print_message "INFO" "Waiting for instance to be running..."

    # Wait for instance to be running
    aws ec2 wait instance-running --instance-ids "$instance_id"

    if [ $? -eq 0 ]; then
        print_message "INFO" "Instance is now running: $instance_id"
        echo "$instance_id"
        return 0
    else
        print_message "ERROR" "Instance failed to start properly: $instance_id"
        return 1
    fi
}

# Function to get instance details
get_instance_details() {
    local instance_id=$1

    local instance_info=$(aws ec2 describe-instances \
        --instance-ids "$instance_id" \
        --query 'Reservations[0].Instances[0].[PublicIpAddress,PrivateIpAddress,State.Name]' \
        --output text)

    echo "$instance_info"
}

#==============================================================================
# DEPLOYMENT STATUS AND CLEANUP FUNCTIONS
#==============================================================================

# Function to verify deployment status
verify_deployment() {
    print_message "INFO" "Verifying deployment status..."

    local all_good=true

    # Check EC2 instances
    if [ ${#CREATED_INSTANCES[@]} -gt 0 ]; then
        print_message "INFO" "Checking EC2 instances..."
        for instance_id in "${CREATED_INSTANCES[@]}"; do
            local state=$(aws ec2 describe-instances \
                --instance-ids "$instance_id" \
                --query 'Reservations[0].Instances[0].State.Name' \
                --output text 2>/dev/null)

            if [ "$state" = "running" ]; then
                print_message "INFO" "✓ Instance $instance_id is running"
            else
                print_message "ERROR" "✗ Instance $instance_id is not running (state: $state)"
                all_good=false
            fi
        done
    fi

    # Check S3 buckets
    if [ ${#CREATED_BUCKETS[@]} -gt 0 ]; then
        print_message "INFO" "Checking S3 buckets..."
        for bucket_name in "${CREATED_BUCKETS[@]}"; do
            if aws s3api head-bucket --bucket "$bucket_name" &> /dev/null; then
                print_message "INFO" "✓ S3 bucket $bucket_name is accessible"
            else
                print_message "ERROR" "✗ S3 bucket $bucket_name is not accessible"
                all_good=false
            fi
        done
    fi

    if $all_good; then
        print_message "INFO" "✓ All resources deployed successfully!"
        return 0
    else
        print_message "ERROR" "✗ Some resources have issues"
        return 1
    fi
}

# Function to display deployment summary
display_summary() {
    print_message "INFO" "=== DEPLOYMENT SUMMARY ==="

    # EC2 Instances Summary
    if [ ${#CREATED_INSTANCES[@]} -gt 0 ]; then
        echo -e "\n${BLUE}EC2 Instances:${NC}"
        for instance_id in "${CREATED_INSTANCES[@]}"; do
            local details=$(get_instance_details "$instance_id")
            local public_ip=$(echo "$details" | cut -f1)
            local private_ip=$(echo "$details" | cut -f2)
            local state=$(echo "$details" | cut -f3)

            echo "  Instance ID: $instance_id"
            echo "  Public IP:   $public_ip"
            echo "  Private IP:  $private_ip"
            echo "  State:       $state"

            if [ "$public_ip" != "None" ] && [ -n "$public_ip" ]; then
                echo "  SSH Command: ssh -i ${PROJECT_NAME}-${ENVIRONMENT}-key.pem ec2-user@$public_ip"
                echo "  Jupyter URL: http://$public_ip:8888/?token=datawise123"
            fi
            echo ""
        done
    fi

    # S3 Buckets Summary
    if [ ${#CREATED_BUCKETS[@]} -gt 0 ]; then
        echo -e "${BLUE}S3 Buckets:${NC}"
        for bucket_name in "${CREATED_BUCKETS[@]}"; do
            echo "  Bucket: $bucket_name"
            echo "  URL:    https://$bucket_name.s3.$AWS_DEFAULT_REGION.amazonaws.com"
            echo ""
        done
    fi

    # Security Groups Summary
    if [ ${#CREATED_SECURITY_GROUPS[@]} -gt 0 ]; then
        echo -e "${BLUE}Security Groups:${NC}"
        for sg_id in "${CREATED_SECURITY_GROUPS[@]}"; do
            echo "  Security Group ID: $sg_id"
        done
        echo ""
    fi

    # Key Pairs Summary
    if [ ${#CREATED_KEY_PAIRS[@]} -gt 0 ]; then
        echo -e "${BLUE}Key Pairs:${NC}"
        for key_name in "${CREATED_KEY_PAIRS[@]}"; do
            echo "  Key Name: $key_name"
            echo "  Key File: $key_name.pem"
        done
        echo ""
    fi
}

# Function to cleanup resources on failure
cleanup_resources() {
    print_message "WARN" "Cleaning up created resources due to failure..."

    # Terminate EC2 instances
    if [ ${#CREATED_INSTANCES[@]} -gt 0 ]; then
        print_message "INFO" "Terminating EC2 instances..."
        for instance_id in "${CREATED_INSTANCES[@]}"; do
            aws ec2 terminate-instances --instance-ids "$instance_id" &> /dev/null
            print_message "INFO" "Terminated instance: $instance_id"
        done
    fi

    # Delete S3 buckets
    if [ ${#CREATED_BUCKETS[@]} -gt 0 ]; then
        print_message "INFO" "Deleting S3 buckets..."
        for bucket_name in "${CREATED_BUCKETS[@]}"; do
            # Empty bucket first
            aws s3 rm s3://"$bucket_name" --recursive &> /dev/null
            # Delete bucket
            aws s3api delete-bucket --bucket "$bucket_name" &> /dev/null
            print_message "INFO" "Deleted bucket: $bucket_name"
        done
    fi

    # Delete security groups (after instances are terminated)
    if [ ${#CREATED_SECURITY_GROUPS[@]} -gt 0 ]; then
        print_message "INFO" "Waiting for instances to terminate before deleting security groups..."
        sleep 30
        for sg_id in "${CREATED_SECURITY_GROUPS[@]}"; do
            aws ec2 delete-security-group --group-id "$sg_id" &> /dev/null
            print_message "INFO" "Deleted security group: $sg_id"
        done
    fi

    # Delete key pairs
    if [ ${#CREATED_KEY_PAIRS[@]} -gt 0 ]; then
        print_message "INFO" "Deleting key pairs..."
        for key_name in "${CREATED_KEY_PAIRS[@]}"; do
            aws ec2 delete-key-pair --key-name "$key_name" &> /dev/null
            rm -f "$key_name.pem"
            print_message "INFO" "Deleted key pair: $key_name"
        done
    fi

    print_message "INFO" "Cleanup completed"
}

#==============================================================================
# MAIN DEPLOYMENT FUNCTION
#==============================================================================

# Function to deploy infrastructure
deploy_infrastructure() {
    local instance_type=$1
    local bucket_name=$2
    local instance_count=${3:-1}

    print_message "INFO" "Starting DataWise infrastructure deployment..."
    print_message "INFO" "Instance Type: $instance_type"
    print_message "INFO" "S3 Bucket: $bucket_name"
    print_message "INFO" "Instance Count: $instance_count"
    print_message "INFO" "Region: $AWS_DEFAULT_REGION"
    print_message "INFO" "Environment: $ENVIRONMENT"

    # Create security group
    local security_group_id=$(create_security_group)
    if [ $? -ne 0 ]; then
        cleanup_resources
        return 1
    fi

    # Create key pair
    local key_name=$(create_key_pair)
    if [ $? -ne 0 ]; then
        cleanup_resources
        return 1
    fi

    # Create S3 bucket
    if ! create_s3_bucket "$bucket_name" "$AWS_DEFAULT_REGION"; then
        cleanup_resources
        return 1
    fi

    # Create EC2 instances
    print_message "INFO" "Creating $instance_count EC2 instance(s)..."
    for ((i=1; i<=instance_count; i++)); do
        print_message "INFO" "Creating instance $i of $instance_count..."
        if ! create_ec2_instance "$instance_type" "$security_group_id" "$key_name"; then
            cleanup_resources
            return 1
        fi
    done

    # Verify deployment
    if verify_deployment; then
        display_summary
        print_message "INFO" "🎉 DataWise infrastructure deployment completed successfully!"
        return 0
    else
        print_message "ERROR" "Deployment verification failed"
        cleanup_resources
        return 1
    fi
}

#==============================================================================
# ERROR HANDLING AND SIGNAL TRAPS
#==============================================================================

# Error handling function
handle_error() {
    local exit_code=$?
    local line_no=$1
    print_message "ERROR" "Script failed at line $line_no with exit code $exit_code"
    cleanup_resources
    exit $exit_code
}

# Signal handler for cleanup on interrupt
cleanup_on_interrupt() {
    print_message "WARN" "Script interrupted by user"
    cleanup_resources
    exit 130
}

# Set error handling
set -e
trap 'handle_error $LINENO' ERR
trap cleanup_on_interrupt SIGINT SIGTERM

#==============================================================================
# COMMAND LINE ARGUMENT PARSING AND HELP
#==============================================================================

# Function to display usage information
show_usage() {
    cat << EOF
DataWise Solutions - AWS Infrastructure Deployment Script

USAGE:
    $0 [OPTIONS]

OPTIONS:
    -t, --instance-type TYPE    EC2 instance type (default: $DEFAULT_INSTANCE_TYPE)
    -b, --bucket-name NAME      S3 bucket name (required)
    -c, --count NUMBER          Number of instances to create (default: 1)
    -r, --region REGION         AWS region (default: $AWS_DEFAULT_REGION)
    -e, --environment ENV       Environment name (default: $ENVIRONMENT)
    -p, --project NAME          Project name (default: $PROJECT_NAME)
    -h, --help                  Show this help message
    --cleanup                   Cleanup mode: delete all resources
    --dry-run                   Show what would be created without actually creating

EXAMPLES:
    # Basic deployment
    $0 --bucket-name my-datawise-bucket

    # Custom instance type and multiple instances
    $0 --bucket-name analytics-data --instance-type m5.large --count 2

    # Production deployment
    $0 --bucket-name prod-analytics --environment prod --region us-west-2

    # Cleanup all resources
    $0 --cleanup

ENVIRONMENT VARIABLES:
    AWS_DEFAULT_REGION      Default AWS region
    DEFAULT_INSTANCE_TYPE   Default EC2 instance type
    PROJECT_NAME           Project name prefix
    ENVIRONMENT            Environment name (dev, staging, prod)

PREREQUISITES:
    - AWS CLI installed and configured
    - jq command-line JSON processor
    - Valid AWS credentials with EC2 and S3 permissions

For more information, visit: https://github.com/datawise-solutions/aws-deployment
EOF
}

# Function for cleanup mode
cleanup_mode() {
    print_message "INFO" "Running in cleanup mode..."

    # Find and delete resources by tags
    local project_filter="Name=tag:Project,Values=$PROJECT_NAME"
    local env_filter="Name=tag:Environment,Values=$ENVIRONMENT"

    # Find instances to terminate
    local instances=$(aws ec2 describe-instances \
        --filters "$project_filter" "$env_filter" "Name=instance-state-name,Values=running,pending,stopped" \
        --query 'Reservations[].Instances[].InstanceId' \
        --output text)

    if [ -n "$instances" ]; then
        print_message "INFO" "Found instances to terminate: $instances"
        aws ec2 terminate-instances --instance-ids $instances
        print_message "INFO" "Termination initiated for instances"
    fi

    # Find and delete S3 buckets
    local buckets=$(aws s3api list-buckets \
        --query "Buckets[?contains(Name, '$PROJECT_NAME')].Name" \
        --output text)

    for bucket in $buckets; do
        if [ -n "$bucket" ]; then
            print_message "INFO" "Deleting S3 bucket: $bucket"
            aws s3 rm s3://"$bucket" --recursive &> /dev/null
            aws s3api delete-bucket --bucket "$bucket" &> /dev/null
        fi
    done

    print_message "INFO" "Cleanup completed"
}

# Function for dry run mode
dry_run_mode() {
    local instance_type=$1
    local bucket_name=$2
    local instance_count=${3:-1}

    print_message "INFO" "=== DRY RUN MODE ==="
    print_message "INFO" "The following resources would be created:"
    echo ""
    echo "EC2 Instances:"
    echo "  - Count: $instance_count"
    echo "  - Type: $instance_type"
    echo "  - AMI: $DEFAULT_AMI_ID"
    echo "  - Region: $AWS_DEFAULT_REGION"
    echo ""
    echo "S3 Buckets:"
    echo "  - Name: $bucket_name"
    echo "  - Region: $AWS_DEFAULT_REGION"
    echo "  - Versioning: Enabled"
    echo "  - Folders: raw-data/, processed-data/, models/, reports/, scripts/"
    echo ""
    echo "Security Groups:"
    echo "  - Name: $PROJECT_NAME-$ENVIRONMENT-sg"
    echo "  - Ports: 22 (SSH), 80 (HTTP), 443 (HTTPS), 8888 (Jupyter)"
    echo ""
    echo "Key Pairs:"
    echo "  - Name: $PROJECT_NAME-$ENVIRONMENT-key"
    echo "  - File: $PROJECT_NAME-$ENVIRONMENT-key.pem"
    echo ""
    print_message "INFO" "No resources were actually created (dry run mode)"
}

#==============================================================================
# MAIN SCRIPT EXECUTION
#==============================================================================

# Main function
main() {
    local instance_type="$DEFAULT_INSTANCE_TYPE"
    local bucket_name=""
    local instance_count=1
    local cleanup_flag=false
    local dry_run_flag=false

    # Parse command line arguments
    while [[ $# -gt 0 ]]; do
        case $1 in
            -t|--instance-type)
                instance_type="$2"
                shift 2
                ;;
            -b|--bucket-name)
                bucket_name="$2"
                shift 2
                ;;
            -c|--count)
                instance_count="$2"
                shift 2
                ;;
            -r|--region)
                export AWS_DEFAULT_REGION="$2"
                shift 2
                ;;
            -e|--environment)
                export ENVIRONMENT="$2"
                shift 2
                ;;
            -p|--project)
                export PROJECT_NAME="$2"
                shift 2
                ;;
            --cleanup)
                cleanup_flag=true
                shift
                ;;
            --dry-run)
                dry_run_flag=true
                shift
                ;;
            -h|--help)
                show_usage
                exit 0
                ;;
            *)
                print_message "ERROR" "Unknown option: $1"
                show_usage
                exit 1
                ;;
        esac
    done

    # Handle special modes
    if $cleanup_flag; then
        cleanup_mode
        exit 0
    fi

    # Validate required parameters
    if [ -z "$bucket_name" ]; then
        print_message "ERROR" "S3 bucket name is required"
        show_usage
        exit 1
    fi

    # Handle dry run mode
    if $dry_run_flag; then
        dry_run_mode "$instance_type" "$bucket_name" "$instance_count"
        exit 0
    fi

    # Validate instance count
    if ! [[ "$instance_count" =~ ^[1-9][0-9]*$ ]]; then
        print_message "ERROR" "Invalid instance count: $instance_count"
        exit 1
    fi

    # Check prerequisites
    if ! check_prerequisites; then
        exit 1
    fi

    # Validate parameters
    if ! validate_parameters "$instance_type" "$bucket_name"; then
        exit 1
    fi

    # Deploy infrastructure
    if deploy_infrastructure "$instance_type" "$bucket_name" "$instance_count"; then
        print_message "INFO" "Deployment completed successfully!"
        exit 0
    else
        print_message "ERROR" "Deployment failed!"
        exit 1
    fi
}

# Execute main function with all arguments
main "$@"
