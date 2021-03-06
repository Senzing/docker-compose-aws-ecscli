# docker-compose-aws-ecscli-demo-beginner-kafka

## Overview

This illustrates a reference implementation of Senzing using
Kafka as the queue and
PostgreSQL as the underlying database
on the Amazon Elastic Container Service (ECS) in EC2 mode.

The instructions show how to set up a system that:

1. Reads JSON lines from a file on the internet.
1. Sends each JSON line to a message queue.
    1. In this implementation, the queue is Kafka.
1. Reads messages from the queue and inserts into Senzing.
    1. In this implementation, Senzing keeps its data in a PostgreSQL database.
1. Reads information from Senzing via [Senzing REST API](https://github.com/Senzing/senzing-rest-api-specification) server.
1. Views resolved entities in a [web app](https://github.com/Senzing/entity-search-web-app).

The following diagram shows the relationship of the docker containers in this docker composition.
Arrows represent data flow.

![Image of architecture](architecture.png)

This docker formation brings up the following docker containers:

1. *[bitnami/kafka](https://github.com/bitnami/bitnami-docker-kafka)*
1. *[bitnami/zookeeper](https://github.com/bitnami/bitnami-docker-zookeeper)*
1. *[dockage/phppgadmin](https://hub.docker.com/r/dockage/phppgadmin)*
1. *[obsidiandynamics/kafdrop](https://hub.docker.com/r/obsidiandynamics/kafdrop)*
1. *[postgres](https://hub.docker.com/_/postgres)*
1. *[senzing/debug](https://github.com/Senzing/docker-senzing-debug)*
1. *[senzing/entity-web-search-app](https://github.com/Senzing/entity-search-web-app)*
1. *[senzing/init-container](https://github.com/Senzing/docker-init-container)*
1. *[senzing/jupyter](https://github.com/Senzing/docker-jupyter)*
1. *[senzing/senzing-api-server](https://github.com/Senzing/senzing-api-server)*
1. *[senzing/stream-loader](https://github.com/Senzing/stream-loader)*
1. *[senzing/stream-producer](https://github.com/Senzing/stream-producer)*
1. *[senzing/xterm](https://github.com/Senzing/docker-xterm)*

## Contents

1. [Prerequisites](#prerequisites)
    1. [Install AWS CLI](#install-aws-cli)
    1. [Install ECS CLI](#install-ecs-cli)
    1. [Clone repository](#clone-repository)
1. [Tutorial](#tutorial)
    1. [Authentication](#authentication)
    1. [Identify metadata](#identify-metadata)
        1. [AWS metadata](#aws-metadata)
        1. [Identify project](#identify-project)
        1. [EULA](#eula)
        1. [Synthesize variables](#synthesize-variables)
    1. [Make AWS project directory](#make-aws-project-directory)
    1. [Configure ECS CLI](#configure-ecs-cli)
    1. [Create cluster](#create-cluster)
    1. [Find EC2 host address](#find-ec2-host-address)
    1. [Find security group ID](#find-security-group-id)
    1. [Open inbound ports](#open-inbound-ports)
    1. [Create tasks and services](#create-tasks-and-services)
        1. [Run install Senzing task](#run-install-senzing-task)
        1. [Create Postgres service](#create-postgres-service)
        1. [Run create Senzing database schema task](#run-create-senzing-database-schema-task)
        1. [Create phpPgAdmin service](#create-phppgadmin-service)
        1. [Run init-container task](#run-init-container-task)
        1. [Create Kafka service](#create-kafka-service)
        1. [Create Kafdrop service](#create-kafdrop-service)
        1. [Run Stream producer task](#run-stream-producer-task)
        1. [Create Stream loader service](#create-stream-loader-service)
        1. [Create Senzing API server service](#create-senzing-api-server-service)
        1. [Create Senzing Web App service](#create-senzing-web-app-service)
        1. [Create Senzing X-Term service](#create-senzing-x-term-service)
        1. [Create Jupyter notebook service](#create-jupyter-notebook-service)
    1. [Service recap](#service-recap)
1. [Cleanup](#cleanup)
    1. [Delete services](#delete-services)
    1. [Delete tasks definitions](#delete-tasks-definitions)
    1. [Bring down cluster](#bring-down-cluster)
    1. [Clean logs](#clean-logs)
    1. [Review cleanup in AWS console](#review-cleanup-in-aws-console)
1. [References](#references)

## Prerequisites

### Install AWS CLI

To install `aws`, see
[How to install Amazon Web Service Command Line Interface](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/install-aws-cli.md).

### Install ECS CLI

To install `ecs-cli`, see
[How to install Amazon Web Service Elastic Compute Service Command Line Interface](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/install-aws-ecs-cli.md).

### Clone repository

For more information on environment variables,
see [Environment Variables](https://github.com/Senzing/knowledge-base/blob/master/lists/environment-variables.md).

1. Set these environment variable values:

    ```console
    export GIT_ACCOUNT=senzing
    export GIT_REPOSITORY=docker-compose-aws-ecscli-demo
    export GIT_ACCOUNT_DIR=~/${GIT_ACCOUNT}.git
    export GIT_REPOSITORY_DIR="${GIT_ACCOUNT_DIR}/${GIT_REPOSITORY}"
    ```

1. Using the environment variables values just set, follow steps in [clone-repository](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/clone-repository.md) to install the Git repository.

## Tutorial

### Authentication

1. Create an
   [AWS session](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/set-aws-environment-variables.md#aws-session).
1. Set the following AWS environment variables:
    1. [AWS_ACCESS_KEY_ID](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/set-aws-environment-variables.md#aws_access_key_id)
    1. [AWS_SECRET_ACCESS_KEY](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/set-aws-environment-variables.md#aws_secret_access_key)
1. :thinking: **Optional:**
   If multi-factor authentication is used to access AWS,
   see [How to set AWS multi-factor authentication credentials](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/set-aws-mfa-credentials.md).

### Identify metadata

#### AWS metadata

1. :pencil2: Set AWS [region](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html).
    1. Set [AWS_DEFAULT_REGION](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/set-aws-environment-variables.md#aws_default_region)
1. :pencil2: Set AWS [key pair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html).
   Example:

    ```console
    export SENZING_AWS_KEYPAIR=aws-default-key-pair
    ```

#### Identify project

1. :pencil2: Choose a prefix used in AWS object names.
   Example:

    ```console
    export SENZING_AWS_PROJECT=project01
    ```

#### EULA

To use the Senzing code, you must agree to the End User License Agreement (EULA).

1. :warning: This step is intentionally tricky and not simply copy/paste.
   This ensures that you make a conscious effort to accept the EULA.
   Example:

    <pre>export SENZING_ACCEPT_EULA="&lt;the value from <a href="https://github.com/Senzing/knowledge-base/blob/master/lists/environment-variables.md#senzing_accept_eula">this link</a>&gt;"</pre>

#### Synthesize variables

1. Create additional environment variables.
   Example:

    ```console
    export SENZING_AWS_ECS_CLUSTER=${SENZING_AWS_PROJECT}-cluster
    export SENZING_AWS_ECS_CLUSTER_CONFIG=${SENZING_AWS_PROJECT}-config-name
    export SENZING_AWS_ECS_PARAMS_FILE=${GIT_REPOSITORY_DIR}/resources/beginner-kafka/ecs-params.yaml
    ```

### Make AWS project directory

1. Make a new directory to handle AWS output.
   Example:

    ```console
    export SENZING_AWS_PROJECT_DIR=~/${SENZING_AWS_PROJECT}
    mkdir ${SENZING_AWS_PROJECT_DIR}
    ```

### Configure ECS CLI

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [configure](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-configure.html).
   Example:

    ```console
    ecs-cli configure \
       --cluster ${SENZING_AWS_ECS_CLUSTER} \
       --config-name ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
       --default-launch-type EC2 \
       --region ${AWS_DEFAULT_REGION}
    ```

1. :thinking: **Optional:**
   To view configuration values, see `~/.ecs/config`.

    ```console
    cat ~/.ecs/config
    ```

### Create cluster

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-up.html).
   Example:

    ```console
    ecs-cli up \
      --capability-iam \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --force \
      --instance-type t3a.xlarge \
      --keypair ${SENZING_AWS_KEYPAIR} \
      --size 1 \
    | tee ${SENZING_AWS_PROJECT_DIR}/ecs-cli-up.txt
    ```

1. :thinking: **Optional:**
   View aspects of AWS ECS cluster in AWS console.
    1. [cloudformation](https://console.aws.amazon.com/cloudformation/home?#/stacks)
    1. [ecs](https://console.aws.amazon.com/ecs/home)
    1. [ec2](https://console.aws.amazon.com/ec2/v2/home)
        1. [auto scaling groups](https://console.aws.amazon.com/ec2autoscaling/home?#/details)
        1. [instances](https://console.aws.amazon.com/ec2/v2/home?#Instances)
        1. [launch configurations](https://console.aws.amazon.com/ec2/autoscaling/home?#LaunchConfigurations)

1. :thinking: **Optional:** List AWS resources created.
   Run
   [aws](https://docs.aws.amazon.com/cli/latest/reference/index.html)
   [cloudformation](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/index.html)
   [list-stack-resources](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/list-stack-resources.html)
   Example:

    ```console
    aws cloudformation list-stack-resources \
      --stack-name amazon-ecs-cli-setup-${SENZING_AWS_ECS_CLUSTER}
    ```

### Find EC2 host address

1. Find the Amazon Resource Name (ARN) of the EC2 instance used in ECS.
   Run
   [aws](https://docs.aws.amazon.com/cli/latest/reference/index.html)
   [ecs](https://docs.aws.amazon.com/cli/latest/reference/ecs/index.html#cli-aws-ecs)
   [list-container-instances](https://docs.aws.amazon.com/cli/latest/reference/ecs/list-container-instances.html)
   Example:

    ```console
    aws ecs list-container-instances \
      --cluster ${SENZING_AWS_ECS_CLUSTER} \
    > ${SENZING_AWS_PROJECT_DIR}/aws-ecs-list-container-instances.json
    ```

1. Save container instance Amazon Resource Name (ARN) in `SENZING_CONTAINER_INSTANCE_ARN` environment variable.
   Example:

    ```console
    export SENZING_CONTAINER_INSTANCE_ARN=$(jq --raw-output ".containerInstanceArns[0]" ${SENZING_AWS_PROJECT_DIR}/aws-ecs-list-container-instances.json)
    ```

1. From the ARN, find the AWS EC2 Instance ID.
   Run
   [aws](https://docs.aws.amazon.com/cli/latest/reference/index.html)
   [ecs](https://docs.aws.amazon.com/cli/latest/reference/ecs/index.html#cli-aws-ecs)
   [describe-container-instances](https://docs.aws.amazon.com/cli/latest/reference/ecs/list-container-instances.html)
   Example:

    ```console
    aws ecs describe-container-instances \
      --cluster ${SENZING_AWS_ECS_CLUSTER} \
      --container-instances ${SENZING_CONTAINER_INSTANCE_ARN} \
    > ${SENZING_AWS_PROJECT_DIR}/aws-ecs-describe-container-instances.json
    ```

1. Save EC2 instance ID in `SENZING_EC2_INSTANCE_ID` environment variable.
   Example:

    ```console
    export SENZING_EC2_INSTANCE_ID=$(jq --raw-output ".containerInstances[0].ec2InstanceId" ${SENZING_AWS_PROJECT_DIR}/aws-ecs-describe-container-instances.json)
    ```

1. From the AWS EC2 Instance ID, find the instance host address.
   Run
   [aws](https://docs.aws.amazon.com/cli/latest/reference/index.html)
   [ec2](https://docs.aws.amazon.com/cli/latest/reference/ec2/index.html#cli-aws-ec2)
   [describe-instances](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-instances.html)
   Example:

    ```console
    aws ec2 describe-instances \
      --filters Name=instance-id,Values=${SENZING_EC2_INSTANCE_ID} \
    > ${SENZING_AWS_PROJECT_DIR}/aws-ec2-describe-instances.json
    ```

1. Save EC2 host IP address in `SENZING_EC2_HOST` environment variable.
   Example:

    ```console
    export SENZING_EC2_HOST=$(jq --raw-output ".Reservations[0].Instances[0].PublicIpAddress" ${SENZING_AWS_PROJECT_DIR}/aws-ec2-describe-instances.json)
    ```

1. :thinking: **Optional:** View EC2 IP address.
   Example:

    ```console
    echo ${SENZING_EC2_HOST}
    ```

### Find security group ID

1. Find the AWS security group for the EC2 instance used in ECS.
   Run
   [aws](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/index.html)
   [cloudformation](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/index.html)
   [list-stack-resources](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/list-stack-resources.html)
   Example:

    ```console
    aws cloudformation list-stack-resources \
      --stack-name amazon-ecs-cli-setup-${SENZING_AWS_ECS_CLUSTER} \
    > ${SENZING_AWS_PROJECT_DIR}/aws-cloudformation-list-stack-resources.json
    ```

1. Save EC2 security group in `SENZING_AWS_EC2_SECURITY_GROUP` environment variable.
   Example:

    ```console
    export SENZING_AWS_EC2_SECURITY_GROUP=$(jq --raw-output ".StackResourceSummaries[] | select(.LogicalResourceId == \"EcsSecurityGroup\").PhysicalResourceId" ${SENZING_AWS_PROJECT_DIR}/aws-cloudformation-list-stack-resources.json)
    ```

1. :thinking: **Optional:**
   View security group ID.
   Example:

    ```console
    echo ${SENZING_AWS_EC2_SECURITY_GROUP}
    ```

### Open inbound ports

:warning: **Warning:** The following inbound port specifications are **wide open**.
Meaning they can be accessed from anywhere.
For demonstration purposes, this is fine.
For production purposes it is not fine.

1. Open inbound ports.
   Run
   [aws](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/index.html)
   [ec2](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/index.html)
   [authorize-security-group-ingress](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/authorize-security-group-ingress.html).
   Example:

    ```console
    aws ec2 authorize-security-group-ingress \
      --group-id ${SENZING_AWS_EC2_SECURITY_GROUP} \
      --ip-permissions \
        IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges='[{CidrIp=0.0.0.0/0,Description="SSH"}]'

    aws ec2 authorize-security-group-ingress \
      --group-id ${SENZING_AWS_EC2_SECURITY_GROUP} \
      --ip-permissions \
        IpProtocol=tcp,FromPort=2181,ToPort=2181,IpRanges='[{CidrIp=0.0.0.0/0,Description="Zookeeper"}]'

    aws ec2 authorize-security-group-ingress \
      --group-id ${SENZING_AWS_EC2_SECURITY_GROUP} \
      --ip-permissions \
        IpProtocol=tcp,FromPort=5432,ToPort=5432,IpRanges='[{CidrIp=0.0.0.0/0,Description="PostgreSQL"}]'

    aws ec2 authorize-security-group-ingress \
      --group-id ${SENZING_AWS_EC2_SECURITY_GROUP} \
      --ip-permissions \
        IpProtocol=tcp,FromPort=8250,ToPort=8250,IpRanges='[{CidrIp=0.0.0.0/0,Description="Senzing API server"}]'

    aws ec2 authorize-security-group-ingress \
      --group-id ${SENZING_AWS_EC2_SECURITY_GROUP} \
      --ip-permissions \
        IpProtocol=tcp,FromPort=8251,ToPort=8251,IpRanges='[{CidrIp=0.0.0.0/0,Description="Senzing Web App"}]'

    aws ec2 authorize-security-group-ingress \
      --group-id ${SENZING_AWS_EC2_SECURITY_GROUP} \
      --ip-permissions \
        IpProtocol=tcp,FromPort=8254,ToPort=8254,IpRanges='[{CidrIp=0.0.0.0/0,Description="Senzing X-Term"}]'

    aws ec2 authorize-security-group-ingress \
      --group-id ${SENZING_AWS_EC2_SECURITY_GROUP} \
      --ip-permissions \
        IpProtocol=tcp,FromPort=9092,ToPort=9092,IpRanges='[{CidrIp=0.0.0.0/0,Description="Kafka service"}]'

    aws ec2 authorize-security-group-ingress \
      --group-id ${SENZING_AWS_EC2_SECURITY_GROUP} \
      --ip-permissions \
        IpProtocol=tcp,FromPort=9171,ToPort=9171,IpRanges='[{CidrIp=0.0.0.0/0,Description="phpPgAdmin"}]'

    aws ec2 authorize-security-group-ingress \
      --group-id ${SENZING_AWS_EC2_SECURITY_GROUP} \
      --ip-permissions \
        IpProtocol=tcp,FromPort=9178,ToPort=9178,IpRanges='[{CidrIp=0.0.0.0/0,Description="Senzing Jupyter notebooks"}]'

    aws ec2 authorize-security-group-ingress \
      --group-id ${SENZING_AWS_EC2_SECURITY_GROUP} \
      --ip-permissions \
        IpProtocol=tcp,FromPort=9179,ToPort=9179,IpRanges='[{CidrIp=0.0.0.0/0,Description="Kafdrop"}]'
    ```

1. :thinking: **Optional:**
   To view Security Group, run
   [aws](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/index.html)
   [ec2](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/index.html)
   [describe-security-groups](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-security-groups.html).
   Example:

    ```console
    aws ec2 describe-security-groups \
      --group-ids ${SENZING_AWS_EC2_SECURITY_GROUP}
    ```

1. :thinking: **Optional:**
   View Security Group in AWS console.
    1. View [ec2 instances](https://console.aws.amazon.com/ec2/v2/home?#Instances)
    1. Choose "ECS instance" for the cluster.
    1. **Security groups:**, click on security group.
    1. In "Security Groups", click on appropriate Security group ID link.

### Create tasks and services

#### Run install Senzing task

Install Senzing into `/opt/senzing` on the EC2 instance.

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [compose](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html)
   [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-up.html)
   Example:

    ```console
    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-init.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-init \
      up \
        --create-log-groups
    ```

1. This task is a short-lived "job", not a long-running service.
   When the task state is `STOPPED`, the job has finished.
1. :thinking: **Optional:**
   View progress in AWS Console.
    1. [ecs](https://console.aws.amazon.com/ecs/home)
        1. Select ${SENZING_AWS_ECS_CLUSTER}
        1. Click "Tasks" tab.
        1. If task is seen, it is still "RUNNING".  Wait until task is complete.
    1. [ec2](https://console.aws.amazon.com/ec2/v2/home)
        1. [instances](https://console.aws.amazon.com/ec2/v2/home?#Instances)
1. Wait until task has completed and is in the `STOPPED` state.

#### Create Postgres service

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [compose](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html)
   [service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service.html)
   [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service-up.html)
   to provision Postgres database service.
   Example:

    ```console
    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-postgres.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-postgres \
      service up
    ```

1. :thinking: **Optional:**
   To view service definition, run
   [aws](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/index.html)
   [ecs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/index.html)
   [describe-services](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/describe-services.html).
   Example:

    ```console
    aws ecs describe-services \
      --cluster ${SENZING_AWS_ECS_CLUSTER} \
      --services ${SENZING_AWS_PROJECT}-project-name-postgres
    ```

1. :thinking: **Optional:**
   View service in AWS Console.
    1. [ecs](https://console.aws.amazon.com/ecs/home)
        1. Select ${SENZING_AWS_ECS_CLUSTER}
        1. Click "Services" tab.
        1. Click on link in "Service Name" column.

#### Run create Senzing database schema task

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [compose](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html)
   [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-up.html)
   to create Senzing database schema.
   Example:

    ```console
    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-postgres-init.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-postgres-init \
      up
    ```

1. This task is a short-lived "job", not a long-running service.
   When the task state is `STOPPED`, the job has finished.

1. :thinking: **Optional:**
   View progress in AWS Console.
    1. [ecs](https://console.aws.amazon.com/ecs/home)
        1. Select ${SENZING_AWS_ECS_CLUSTER}
        1. Click "Tasks" tab.
        1. If task is seen, it is still "RUNNING".

#### Create phpPgAdmin service

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [compose](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html)
   [service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service.html)
   [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service-up.html)
   to provision phpPgAdmin database client service.
   Example:

    ```console
    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-phppgadmin.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-phppgadmin \
      service up
    ```

1. :thinking: **Optional:**
   To view phpPgAdmin,
   run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [ps](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-ps.html)
   to find IP address and port.
   Example:

    ```console
    ecs-cli ps \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
    | grep phppgadmin
    ```

   **URL:** [http://${SENZING_EC2_HOST}:9171](http://0.0.0.0:9171)
   **Username:** postgres
   **Password:** postgres

#### Run init-container task

Configure Senzing in `/etc/opt/senzing` and `/var/opt/senzing` files.

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [compose](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html)
   [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-up.html)
   Example:

    ```console
    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-init-container.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-init-container \
      up
    ```

1. This task is a short-lived "job", not a long-running service.
   When the task state is `STOPPED`, the job has finished.
1. :thinking: **Optional:**
   View progress in AWS Console.
    1. [ecs](https://console.aws.amazon.com/ecs/home)
        1. Select ${SENZING_AWS_ECS_CLUSTER}
        1. Click "Tasks" tab.
        1. If task is seen, it is still "RUNNING".
1. Wait until task has completed and is in the `STOPPED` state.

#### Create Kafka service

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [compose](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html)
   [service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service.html)
   [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service-up.html)
   to provision Kafka service.
   Example:

    ```console
    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-kafka.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-kafka \
      service up
    ```

1. :thinking: **Optional:**
   To view service definition, run
   [aws](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/index.html)
   [ecs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/index.html)
   [describe-services](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/describe-services.html).
   Example:

    ```console
    aws ecs describe-services \
      --cluster ${SENZING_AWS_ECS_CLUSTER} \
      --services ${SENZING_AWS_PROJECT}-project-name-kafka
    ```

#### Create Kafdrop service

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [compose](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html)
   [service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service.html)
   [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service-up.html)
   to provision Kafdrop service.
   Example:

    ```console
    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-kafdrop.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-kafdrop \
      service up
    ```

1. :thinking: **Optional:**
   To view service definition, run
   [aws](https://docs.aws.amazon.com/cli/latest/reference/index.html)
   [ecs](https://docs.aws.amazon.com/cli/latest/reference/ecs/index.html)
   [describe-services](https://docs.aws.amazon.com/cli/latest/reference/ecs/describe-services.html).
   Example:

    ```console
    aws ecs describe-services \
      --cluster ${SENZING_AWS_ECS_CLUSTER} \
      --services ${SENZING_AWS_PROJECT}-project-name-kafdrop
    ```

1. :thinking: **Optional:**
   To view Kafdrop,
   run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [ps](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-ps.html)
   to find IP address and port.
   Example:

    ```console
    ecs-cli ps \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
    | grep kafdrop
    ```

   **URL:** [http://${SENZING_EC2_HOST}:9179](http://0.0.0.0:9179)

#### Run Stream producer task

Read JSON lines from a URL-addressable file and send to Kafka.

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [compose](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html)
   [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-up.html)
   to send messages to Kafka.
   Example:

    ```console
    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-stream-producer.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-stream-producer \
      up
    ```

1. This task is a short-lived "job", not a long-running service.
   When the task state is `STOPPED`, the job has finished.
   However, this is a long-running job.
   There is no need to wait for its completion.

#### Create Stream loader service

The stream loader service reads messages from Kafka and inserts them into the Senzing Model.

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [compose](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html)
   [service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service.html)
   [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service-up.html)
   to provision stream-loader service.
   Example:

    ```console
    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-stream-loader.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-stream-loader \
      service up
    ```

1. :thinking: **Optional:**
   To view service definition, run
   [aws](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/index.html)
   [ecs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/index.html)
   [describe-services](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/describe-services.html).
   Example:

    ```console
    aws ecs describe-services \
      --cluster ${SENZING_AWS_ECS_CLUSTER} \
      --services ${SENZING_AWS_PROJECT}-project-name-stream-loader
    ```

#### Create Senzing API server service

The Senzing API server communicates with the Senzing Engine to provide an HTTP
[Senzing REST API](https://github.com/Senzing/senzing-rest-api-specification).

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [compose](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html)
   [service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service.html)
   [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service-up.html)
   to provision Senzing API server service.
   Example:

    ```console
    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-apiserver.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-apiserver \
      service up
    ```

1. :thinking: **Optional:**
   To view service definition, run
   [aws](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/index.html)
   [ecs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/index.html)
   [describe-services](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/describe-services.html).
   Example:

    ```console
    aws ecs describe-services \
      --cluster ${SENZING_AWS_ECS_CLUSTER} \
      --services ${SENZING_AWS_PROJECT}-project-name-apiserver
    ```

1. :thinking: **Optional:**
   To view API server, run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [ps](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-ps.html)
   to find IP address and port.
   Example:

    ```console
    ecs-cli ps \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
    | grep apiserver
    ```

1. :thinking: **Optional:**
   Verify Senzing API server is running.
   A JSON response should be given to the following `curl` request.
   Example:

    ```console
    curl -X GET "http://${SENZING_EC2_HOST}:8250/heartbeat"
    ```

1. :thinking: **Optional:**
   Play with
   [Senzing API in Swagger editor](http://editor.swagger.io/?url=https://raw.githubusercontent.com/Senzing/senzing-rest-api-specification/master/senzing-rest-api.yaml).
   In **Server variables** > **host** text field, enter value of `SENZING_EC2_HOST`.
   To find the value, run

    ```console
    echo $SENZING_EC2_HOST
    ```

#### Create Senzing Web App service

The Senzing Web App provides a user interface to Senzing functionality.

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [compose](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html)
   [service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service.html)
   [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service-up.html)
   to provision Senzing Web App service.
   Example:

    ```console
    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-webapp.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-webapp \
      service up
    ```

1. :thinking: **Optional:**
   To view service definition, run
   [aws](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/index.html)
   [ecs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/index.html)
   [describe-services](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/describe-services.html).
   Example:

    ```console
    aws ecs describe-services \
      --cluster ${SENZING_AWS_ECS_CLUSTER} \
      --services ${SENZING_AWS_PROJECT}-project-name-webapp
    ```

1. :thinking: **Optional:**
   To view Senzing web app, run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [ps](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-ps.html)
   to find IP address and port.
   Example:

    ```console
    ecs-cli ps \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
    | grep webapp
    ```

   **URL:** [http://${SENZING_EC2_HOST}:8251](http://0.0.0.0:8251)

#### Create Senzing X-Term service

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [compose](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html)
   [service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service.html)
   [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service-up.html)
   to provision Senzing X-Term service.
   Example:

    ```console
    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-xterm.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-xterm \
      service up
    ```

1. :thinking: **Optional:**
   To view service definition, run
   [aws](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/index.html)
   [ecs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/index.html)
   [describe-services](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/describe-services.html).
   Example:

    ```console
    aws ecs describe-services \
      --cluster ${SENZING_AWS_ECS_CLUSTER} \
      --services ${SENZING_AWS_PROJECT}-project-name-xterm
    ```

1. :thinking: **Optional:**
   To view Senzing X-Term, run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [ps](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-ps.html)
   to find IP address and port.
   Example:

    ```console
    ecs-cli ps \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
    | grep xterm
    ```

   **URL:** [http://${SENZING_EC2_HOST}:8254](http://0.0.0.0:8254)

#### Create Jupyter notebook service

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [compose](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html)
   [service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service.html)
   [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service-up.html)
   to provision Senzing Web App service.
   Example:

    ```console
    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-jupyter.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-jupyter \
      service up
    ```

1. :thinking: **Optional:**
   To view service definition, run
   [aws](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/index.html)
   [ecs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/index.html)
   [describe-services](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/describe-services.html).
   Example:

    ```console
    aws ecs describe-services \
      --cluster ${SENZING_AWS_ECS_CLUSTER} \
      --services ${SENZING_AWS_PROJECT}-project-name-jupyter
    ```

1. :thinking: **Optional:**
   To view Jupyter, run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [ps](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-ps.html)
   to find IP address and port.
   Example:

    ```console
    ecs-cli ps \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
    | grep jupyter
    ```

   **URL:** [http://${SENZING_EC2_HOST}:9178](http://0.0.0.0:9178)

### Service recap

Once the formation is running, the following services can be found at `SENZING_EC2_HOST`.

To find the value of `SENZING_EC2_HOST`, run

```console
echo $SENZING_EC2_HOST
```

1. [http://${SENZING_EC2_HOST}:8251](http://0.0.0.0:8251) - Senzing Entity Search Web App
1. [http://${SENZING_EC2_HOST}:8254](http://0.0.0.0:8254) - Senzing X-Term
1. [http://${SENZING_EC2_HOST}:9171](http://0.0.0.0:9171) - PhpPgAdmin
1. [http://${SENZING_EC2_HOST}:9178](http://0.0.0.0:9178) - Jupyter Notebooks
1. [http://${SENZING_EC2_HOST}:9179](http://0.0.0.0:9179) - Kafdrop
1. [Senzing API in Swagger editor](http://editor.swagger.io/?url=https://raw.githubusercontent.com/Senzing/senzing-rest-api-specification/master/senzing-rest-api.yaml).

## Cleanup

### Delete services

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [compose](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html)
   [service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service.html)
   [down](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service-rm.html)
   to stop services.
   Example:

    ```console
    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-jupyter.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-jupyter \
      service down

    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-xterm.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-xterm \
      service down

    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-webapp.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-webapp \
      service down

    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-apiserver.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-apiserver \
      service down

    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-stream-loader.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-stream-loader \
      service down

    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-kafdrop.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-kafdrop \
      service down

    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-kafka.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-kafka \
      service down

    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-phppgadmin.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-phppgadmin \
      service down

    ecs-cli compose \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG} \
      --ecs-params ${SENZING_AWS_ECS_PARAMS_FILE} \
      --file ${GIT_REPOSITORY_DIR}/resources/beginner-kafka/docker-compose-postgres.yaml \
      --project-name ${SENZING_AWS_PROJECT}-project-name-postgres \
      service down
    ```

### Delete tasks definitions

1. Identify suffixes for `${SENZING_AWS_PROJECT}-project-name-`.
   Example:

    ```console
    export SENZING_ECS_TASK_DEFINITIONS=( \
      "apiserver" \
      "init" \
      "init-container" \
      "jupyter" \
      "kafdrop" \
      "kafka" \
      "phppgadmin" \
      "postgres" \
      "postgres-init" \
      "stream-loader" \
      "stream-producer" \
      "webapp" \
      "xterm" \
    )
    ```

1. Delete task definitions. Run
   [aws](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/index.html)
   [ecs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/index.html)
   [deregister-task-definition](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/deregister-task-definition.html)
   in a loop over `SENZING_ECS_TASK_DEFINITIONS` values.
   Example:

    ```console
    for SENZING_ECS_TASK_DEFINITION in ${SENZING_ECS_TASK_DEFINITIONS[@]};\
    do \
      aws ecs deregister-task-definition \
        --task-definition $( \
          aws ecs list-task-definitions \
            --family-prefix "${SENZING_AWS_PROJECT}-project-name-${SENZING_ECS_TASK_DEFINITION}" \
          | jq --raw-output .taskDefinitionArns[0] \
        ) > /dev/null; \
    done
    ```

### Bring down cluster

1. Run
   [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
   [down](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-down.html).
   Example:

    ```console
    ecs-cli down \
      --force \
      --cluster-config ${SENZING_AWS_ECS_CLUSTER_CONFIG}
    ```

### Clean logs

1. Delete logs. Run
   [aws](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/index.html)
   [logs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/logs/index.html)
   [delete-log-group](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/logs/delete-log-group.html)
   Example:

    ```console
    aws logs delete-log-group \
      --log-group-name senzing-docker-compose-aws-ecscli-demo
    ```

### Review cleanup in AWS console

1. In AWS Console:
    1. [ec2](https://console.aws.amazon.com/ec2/v2/home)
        1. [auto scaling groups](https://console.aws.amazon.com/ec2/autoscaling/home?#AutoScalingGroups)
        1. [instances](https://console.aws.amazon.com/ec2/v2/home?#Instances)
        1. [launch configurations](https://console.aws.amazon.com/ec2/autoscaling/home?#LaunchConfigurations)
        1. [network interfaces](https://console.aws.amazon.com/ec2/v2/home?#NIC)
        1. [security groups](https://console.aws.amazon.com/ec2/v2/home?#SecurityGroups)
    1. [cloudformation](https://console.aws.amazon.com/cloudformation/home?#/stacks)
    1. [cloudwatch](https://console.aws.amazon.com/cloudwatch/home)
        1. [log groups](https://console.aws.amazon.com/cloudwatch/home?#logsV2:log-groups)
    1. [ecs](https://console.aws.amazon.com/ecs/home)
        1. [clusters](https://console.aws.amazon.com/ecs/home?#/clusters)
        1. [task definitions](https://console.aws.amazon.com/ecs/home?#/taskDefinitions)

## References

1. [AWS](https://aws.amazon.com/)
   &gt; [Documentation](https://docs.aws.amazon.com/index.html)
   &gt; [Amazon ECS](https://docs.aws.amazon.com/ecs/index.html)
   &gt; [Developer Guide](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html)
    1. [ecs-cli](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_reference.html)
        1. [compose](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html)
            1. [service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service.html)
                1. [rm](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service-rm.html)
                1. [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service-up.html)
            1. [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-up.html)
        1. [configure](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-configure.html)
        1. [down](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-down.html)
        1. [ps](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-ps.html)
        1. [up](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-up.html)
1. [AWS](https://aws.amazon.com/)
   &gt; [Documentation](https://docs.aws.amazon.com/index.html)
   &gt; [AWS CLI](https://awscli.amazonaws.com/v2/documentation/api/latest/index.html)
    1. [aws](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/index.html)
        1. [cloudformation](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/index.html)
            1. [list-stack-resources](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/list-stack-resources.html)
        1. [cloudwatch](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudwatch/index.html)
        1. [ec2](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/index.html)
            1. [authorize-security-group-ingress](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/authorize-security-group-ingress.html)
            1. [describe-security-groups](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-security-groups.html)
        1. [ecs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/index.html)
            1. [describe-services](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/describe-services.html)
        1. [logs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/logs/index.html)
1. AWS console
    1. [cloudformation](https://console.aws.amazon.com/cloudformation/home?#/stacks)
    1. [cloudwatch](https://console.aws.amazon.com/cloudwatch/home)
        1. [log groups](https://console.aws.amazon.com/cloudwatch/home?#logsV2:log-groups)
    1. [ec2](https://console.aws.amazon.com/ec2/v2/home)
        1. [auto scaling groups](https://console.aws.amazon.com/ec2autoscaling/home?#/details)
        1. [instances](https://console.aws.amazon.com/ec2/v2/home?#Instances)
        1. [launch configurations](https://console.aws.amazon.com/ec2/autoscaling/home?#LaunchConfigurations)
        1. [network interfaces](https://console.aws.amazon.com/ec2/v2/home?#NIC)
    1. [ecs](https://console.aws.amazon.com/ecs/home)
1. [Installing the Amazon ECS CLI](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.html).
1. [Using the awslogs Log Driver](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_awslogs.html)
1. YAML file formats
    1. [Using Docker Compose File Syntax](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-parameters.html)
    1. [Using Amazon ECS Parameters](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-ecsparams.html)
1. [Tutorial: Creating a Cluster with an EC2 Task Using the Amazon ECS CLI](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-cli-tutorial-ec2.html)
