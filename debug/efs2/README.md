## Demonstrate ECS-EFS bug

### ecs-cli version

1. Show version.
   Example

    ```console
    $ ecs-cli --version
    ecs-cli version 1.20.0 (7547c45)
    ```

### Configure ECS CLI

1. Configure `ecs-cli`.
   Example:

    ```console
    ecs-cli configure \
       --cluster hello-world-cluster \
       --config-name hello-world-config-name \
       --default-launch-type FARGATE \
       --region us-east-1
    ```

### Create cluster

1. Create cluster.
   Example:

    ```console
    ecs-cli up \
      --cluster-config hello-world-config-name \
      --force
    ```

### Save Cluster metadata

1. The `ecs-cli up` command that just completed prints metadata
   that needs to be captured in environment variables for later use.
   Example:

    ```console
    :
    INFO[0001] Waiting for your cluster resources to be created...
    INFO[0002] Cloudformation stack status         stackStatus=CREATE_IN_PROGRESS
    INFO[0063] Cloudformation stack status         stackStatus=CREATE_IN_PROGRESS
    VPC created: vpc-00000000000000000
    Subnet created: subnet-11111111111111111
    Subnet created: subnet-22222222222222222
    Cluster creation succeeded.
    ```

1. :pencil2: Set environment variable with VPC ID.
   Example:

    ```console
    export AWS_VPC_ID=vpc-00000000000000000
    ```

1. :pencil2: Set environment variable with Subnet #1.
   Example:

    ```console
    export AWS_SUBNET_ID_1=subnet-11111111111111111
    ```

1. :pencil2: Set environment variable with Subnet #2.
   Example:

    ```console
    export AWS_SUBNET_ID_2=subnet-22222222222222222
    ```

### Find security group ID

1. Find AWS security group.
   Example:

    ```console
    export AWS_EC2_SECURITY_GROUP=$( \
      aws ec2 describe-security-groups \
        --filters Name=vpc-id,Values=${AWS_VPC_ID} \
        --region us-east-1 \
      | jq --raw-output ".SecurityGroups[0].GroupId"
    )
    ```

### Provision Elastic File system

1. Create EFS file system.
   Example:

    ```console
    export AWS_EFS_FILESYSTEM_ID=$( \
      aws efs create-file-system \
        --creation-token hello-world-efs \
        --tags Key=Name,Value=hello-world-ecs-cluster-efs \
      | jq --raw-output ".FileSystemId"
    )
    ```

### Create EFS mounts

1. Create mount in first subnet.
   Example:

    ```console
    aws efs create-mount-target \
      --file-system-id ${AWS_EFS_FILESYSTEM_ID} \
      --security-groups ${AWS_EC2_SECURITY_GROUP} \
      --subnet-id ${AWS_SUBNET_ID_1}
    ```

1. Create mount in second subnet.
   Example:

    ```console
    aws efs create-mount-target \
      --file-system-id ${AWS_EFS_FILESYSTEM_ID} \
      --security-groups ${AWS_EC2_SECURITY_GROUP} \
      --subnet-id ${AWS_SUBNET_ID_2}
    ```

### Run hello-world

1. Run task.
   Example:

    ```console
    ecs-cli compose \
      --cluster-config hello-world-config-name \
      --ecs-params ecs-params.yaml \
      --file hello-world.yaml \
      --project-name hello-world-project-name \
      up \
        --create-log-groups
    ```

## Cleanup

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [down](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-down.html).
   Example:

    ```console
    export SENZING_ECS_TASK_DEFINITIONS=( \
      "hello-world" \
    )

    for SENZING_ECS_TASK_DEFINITION in ${SENZING_ECS_TASK_DEFINITIONS[@]};\
    do \
      aws ecs deregister-task-definition \
        --task-definition $( \
          aws ecs list-task-definitions \
            --family-prefix "${SENZING_ECS_TASK_DEFINITION}-project-name" \
          | jq --raw-output .taskDefinitionArns[0] \
        ) > /dev/null; \
    done

    ecs-cli down \
      --force \
      --cluster-config hello-world-config-name

    aws efs delete-file-system \
      --file-system-id ${AWS_EFS_FILESYSTEM_ID}

    aws logs delete-log-group \
      --log-group-name hello-world
    ```