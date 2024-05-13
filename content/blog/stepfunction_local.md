+++
date = "2024-05-13T09:00:00-00:00"
title = "Test AWS Stepfunctions locally with CDK"
image = "/images/stepfunction_local.jpg"

+++

While AWS Step Functions offers robust capabilities for workflow management in the cloud, testing these workflows locally before deploying them can be essential for ensuring smooth execution and debugging potential issues. Fortunately, AWS provides tools and resources for local testing, allowing developers to validate their workflows in a controlled environment before deploying them to the cloud.

![](/images/aws_stepfunction_flow.png "Figure 1: Example stepfunction workflow")

AWS offers the **AWS Step Functions Local Docker container**, which enables you to run Step Functions workflows locally in an isolated Docker environment. This containerized approach allows you to replicate the behavior of Step Functions in the cloud while providing flexibility and control over your testing environment.

By leveraging this local testing tool, you can iterate rapidly on your Step Functions workflows, validate their behavior under various conditions, and ensure they meet your requirements before deploying them to production. This iterative development process not only accelerates your development lifecycle but also enhances the reliability and resilience of your workflows in production environments.

In this guide, we'll explore how to set up and use this local testing tool effectively in conjunction with CDK (Cloud Development Kit), empowering you to build and deploy robust workflows with confidence using AWS Step Functions.

## Run AWS Stepfunction in Docker

The following command will run AWS Stepfunction in a docker container:

```shell
docker run -p 8083:8083 amazon/aws-stepfunctions-local
```

It will pull the docker image *amazon/aws-stepfunctions-local* if not already present and then run it in the foreground on port 8083. This is the bare minimum to start AWS Stepfunction in Docker and we will later in this guide revise this command to add additional configuration options needed.

## AWS Stepfunction in CDK

We'll delve into the integration of AWS Step Functions with CDK projects. If you're already acquainted with CDK and its transformative approach to infrastructure-as-code, contrasted with the more traditional CloudFormation templates, then you might also have written countless of Stepfunction definition in CDK.

Step Functions, with their intricate state definition language, can sometimes pose challenges in organization and readability within a CDK project, particularly as workflows grow in complexity.

By testing your Step Functions locally, you can gain a comprehensive understanding and proactively identify any potential issues before deploying them to your production environment. We'll explore how to conduct this testing directly from within a CDK project.


### The DRY principle

No, we’re not discussing the art of drying clothes. Instead, we’re talking about the **"Don't Repeat Yourself"** (DRY) principle, which advises against writing the same code twice.

Why am I mentioning this? Well, for testing our Step Functions, we’ll need to run the local AWS Step Functions Docker container and create a step function definition to execute. We want to avoid duplicating the step function definition that we’ve already written as part of our CDK code. However, since the step function definition is expressed in Amazon States Language (ASL), we can’t directly pass the CDK code to our local Step Functions instance, as demonstrated below:

```shell
aws stepfunctions --endpoint http://localhost:8083 create-state-machine --name "MyStepfunction" --definition "${definition}" --role-arn "arn:aws:iam::1123456789:role/service-role/MyTestRole"
```

Instead, we aim to seamlessly **reuse the Step Functions** that we’ve already written in CDK and directly test them within our local Docker container.

### CDK Synthesize

To adhere to the DRY principle of not repeating ourself and be able to reuse the stepfunctions already written in CDK, we will use the **CDK synthesizer**:

```shell
cdk synth
```

The command above generates an AWS CloudFormation template for each stack in your CDK application. It prints these templates to the console and also stores them in the **cdk.out** folder.

Once we have the AWS Cloudformation template we can extract the Amazon States Language (ASL) from the template and pass that to our `aws stepfunction create-state-machine` command.

Below is an example snippet of how the CLoudFormation template for an ASL definition looks like:

{{< collapsible json>}}
"myStepfunction92E8D837": {
   "Type": "AWS::StepFunctions::StateMachine",
   "Properties": {
    "DefinitionString": {
     "Fn::Join": [
      "",
      [
       "{\"StartAt\":\"GetModelData\",\"States\":{\"GetModelData\":{\"Type\":\"Parallel\",\"Next\":\"Check for version\",\"Branches\":[{\"StartAt\":\"GetData\",\"States\":{\"GetData\":{\"End\":true,\"Retry\":[{\"ErrorEquals\":[\"Lambda.ClientExecutionTimeoutException\",\"Lambda.ServiceException\",\"Lambda.AWSLambdaException\",\"Lambda.SdkClientException\"],\"IntervalSeconds\":2,\"MaxAttempts\":6,\"BackoffRate\":2}],\"Type\":\"Task\",\"ResultPath\":\"$.mydata\",\"Resource\":\"arn:",
       {
        "Ref": "AWS::Partition"
       },
       ":lambda:",
       {
        "Ref": "AWS::Region"
       },
       ":",
       {
        "Ref": "AWS::AccountId"
       },
       ":function:MyFunction\",\"Parameters\":{\"end.$\":\"$.endTime\",\"start.$\":\"$.startedTime\",\"modelVersion\":\"_3\"}}}},{\"StartAt\":\"GetData\",\"States\":{\"GetData\":{\"End\":true,\"Retry\":[{\"ErrorEquals\":[\"Lambda.ClientExecutionTimeoutException\",\"Lambda.ServiceException\",\"Lambda.AWSLambdaException\",\"Lambda.SdkClientException\"],\"IntervalSeconds\":2,\"MaxAttempts\":6,\"BackoffRate\":2}],\"Type\":\"Task\",\"Resource\":\"arn:",
       {
        "Ref": "AWS::Partition"
       },
       ]
      ]
    }
   }
}
{{< /collapsible>}}
\
In the snippet above, we can observe that the Amazon States Language (ASL) definition resides within the **DefinitionString** property of the CloudFormation template. For those who have extensive experience with CloudFormation prior to the rise of CDK, this is analogous to what you would typically express in JSON or YAML.

Our objective is to extract that portion of the template and pass it to the **--definition** option of the command:

```shell
aws stepfunctions --endpoint http://localhost:8083 create-state-machine --name "MyStepfunction" --definition "${definition}" --role-arn "arn:aws:iam::1123456789:role/service-role/MyTestRole"
```

In the following let's look at how we can automate this using **bash scripts**.

## Automate tests using bash scripts

In this section, we’ll provide bash scripts to **automate the entire process of testing Step Functions locally using a Docker container** and also demonstrate how to reuse step function definitions from our CDK projects.


### Synthesize and Extract ASL from CloudFormation template

The following bash function will synthesize our CDK application and extract the ASL definition from the CloudFormation template:

{{< collapsible bash>}}
# this is just a fake account id used for local testing
AWS_ACCOUNT_ID=123456789012

# This function will parse the synthesized cloudformation template and extract the state machine definition string
# to be used when creating a state machine
# The function takes 2 arguments:
# 1: Name of the CloudFormation template, e.g MyTemplate.template.json
# 2: The resource identifier of the stepfunction to extract, e.g MyStepfunction1234
function get_synth_definition () {
    local templateFile=$1
    local stepfunctionName=$2
    local fnJoin=""
    local definition=""
    local asl=""
    local itemToAppend=""

    # Synthesize the CDK application
    local synth=$(cdk synth --no-staging)

    # these are pseudo parameters we want to process when found in the definition string of the state machine ASL
    local partition="{\"Ref\":\"AWS::Partition\"}"
    local accountId="{\"Ref\":\"AWS::AccountId\"}"

    # gets the Fn::Join field in the template
    fnJoin=$(cat cdk.out/${templateFile} | jq --arg sfname "$stepfunctionName" '.Resources | to_entries[] | select((.value.Type=="AWS::StepFunctions::StateMachine") and (.key == $sfname))|.value.Properties.DefinitionString')

    # extract the actual definition part of the template
    definition=$(echo $fnJoin | jq -rc '.["Fn::Join"] | .[1][]')

    # process each item in the definition string array and replace any found pseudo parameters
    # the final result is the ASL (State machine language)
    readarray -t arr < <(echo "${definition}"|tr -d '\r')
    for item in "${arr[@]}"; do
        itemToAppend=$item
        if [[ "${item}" =~ $partition ]]; then
            itemToAppend="aws"
        fi
        if [[ "${item}" =~ $accountId ]]; then
            itemToAppend="$AWS_ACCOUNT_ID"
        fi
        asl+="$itemToAppend"
    done

    echo $asl
}

{{< /collapsible>}}
\
You can then use the above function in your script like the following:

{{< collapsible bash>}}
asl=$(get_synth_definition MyProject.template.json MyStepfunction)
echo "asl: ${asl}"
{{< /collapsible>}}
\
The resulting ASL definition string will be saved inside the variable *asl* and also printed to the console.

### Mock stepfunctions

Once we define our ASL (Amazon States Language), we can test our step functions using mock configurations, eliminating the need to call external resources during testing.

To achieve this, we need to create a mock configuration JSON file named *MockConfigFile.json*. This file defines the test cases along with their expected outputs for your step functions. For a detailed description of the configuration file, refer to the [official documentation](https://docs.aws.amazon.com/step-functions/latest/dg/sfn-local-mock-cfg-file.html).

The following is an example mock configuration file where we return some mocked data in the *GetData* step of the *MyStepFunction*:

{{< collapsible json>}}
{
    "StateMachines": {
        "MyStepFunction": {
            "TestCases": {
                "HappyPath": {
                    "GetData": "MockGetData",
                }
            }
        }
    },
    "MockedResponses": {
        "MockGetData": {
            "0": {
                "Return": {
                    "age": 10,
                    "name": "bob"
                }
            }
        }
    }
}
{{< /collapsible>}}
\
We can then provide the mock configuration file to our local stepfunction Docker container. The container will then use the mocked output instead of making calls to external resources such as Lambda or other services.

### Configure local Stepfunction container

Let's use the mock configuration when starting the local stepfunction container, with the following function:

{{< collapsible bash>}}
# This function will download and start a local AWS Stepfunction docker image
# If an existing docker container is already running it will not create a new docker container
# The function will mount the MockConfigFile.json and configure credentials for the stepfunction.
# The credentials will be fake credentials since we don't need real credentials for running stepfunction locally with mocked data.
function start_step_function_container () {
    
    local image_name="amazon/aws-stepfunctions-local"

    if [ "$( docker inspect -f '{{.Config.Image}}' $(docker ps --format='{{.ID}}') )" == "$image_name" ]
    then
        echo "$image_name is already running"
        echo "continue without starting a new container"
    else
        # Run a local docker image of stepfunction, give it some invalid credentials since it requies credentials, but they don't need to be valid ones.
        CONTAINER_NAME=`docker run -d -p 8083:8083 --mount type=bind,readonly,source="$(pwd)"/MockConfigFile.json,destination=/home/stepfunctionslocal/MockConfigFile.json -e SFN_MOCK_CONFIG=MockConfigFile.json --env-file stepfunction-config.txt amazon/aws-stepfunctions-local`
        echo "$CONTAINER_NAME started"
        until [ "`docker inspect -f {{.State.Running}} $CONTAINER_NAME`"=="true" ]; do
            echo "wait for container to run..."
            sleep 0.1;
        done;
        echo "container started"
    fi
    sleep 4
}
{{< /collapsible>}}
\
The function above handles running and configuring the local Step Functions container, utilizing a Mock configuration file named **MockConfigFile.json** and a credentials file named **stepfunction-config.txt**.

Alternatively, you could simplify this process by using a **Docker Compose** file. This approach provides a cleaner and more concise way to start and configure the container

The credentials file **stepfunction-config.txt** will be utilized by the Docker container to load credentials for calling the AWS API. Since we’re mocking our Step Functions, genuine credentials aren’t necessary. Instead, we can set up fake credentials, as shown in the following *stepfunction-config.txt* content:

{{< collapsible txt>}}
AWS_DEFAULT_REGION=eu-west-1
AWS_ACCESS_KEY_ID=x
AWS_SECRET_ACCESS_KEY=x
{{< /collapsible>}}

### Let's put it all together

Here are all the scripts and configuration files that we have created.

**MocConfigFile.json**

{{< collapsible json>}}
{
    "StateMachines": {
        "MyStepFunction": {
            "TestCases": {
                "HappyPath": {
                    "GetData": "MockGetData",
                }
            }
        }
    },
    "MockedResponses": {
        "MockGetData": {
            "0": {
                "Return": {
                    "age": 10,
                    "name": "bob"
                }
            }
        }
    }
}
{{< /collapsible>}}
\
**stepfunction-config.txt**

{{< collapsible txt>}}
AWS_DEFAULT_REGION=eu-west-1
AWS_ACCESS_KEY_ID=x
AWS_SECRET_ACCESS_KEY=x
{{< /collapsible>}}
\
**stepfunction_local.sh**

{{< collapsible bash>}}
#!/bin/bash

#
# This script requires JQ to work. If you are running on Windows platform you can install the jq version for windows using the following command:
# curl -L -o /usr/bin/jq.exe https://github.com/stedolan/jq/releases/latest/download/jq-win64.exe
#
# or download the jq-win64.exe from above and rename it to jq.exe and place it inside the folder:
# C:/Program Files/Git/usr/bin/jq.exe
#

# this is just a fake account id used for local testing
AWS_ACCOUNT_ID=123456789012
CONTAINER_NAME=""

# This function will parse the synthesized cloudformation template and extract the state machine definition string
# to be used when creating a state machine
# The function takes 2 arguments:
# 1: Name of the CloudFormation template, e.g MyTemplate.template.json
# 2: The resource identifier of the stepfunction to extract, e.g MyStepfunction1234
function get_synth_definition () {
    local templateFile=$1
    local stepfunctionName=$2
    local fnJoin=""
    local definition=""
    local asl=""
    local itemToAppend=""

    # Synthesize the CDK application
    local synth=$(cdk synth --no-staging)

    # these are pseudo parameters we want to process when found in the definition string of the state machine ASL
    local partition="{\"Ref\":\"AWS::Partition\"}"
    local accountId="{\"Ref\":\"AWS::AccountId\"}"

    # gets the Fn::Join field in the template
    fnJoin=$(cat cdk.out/${templateFile} | jq --arg sfname "$stepfunctionName" '.Resources | to_entries[] | select((.value.Type=="AWS::StepFunctions::StateMachine") and (.key == $sfname))|.value.Properties.DefinitionString')

    # extract the actual definition part of the template
    definition=$(echo $fnJoin | jq -rc '.["Fn::Join"] | .[1][]')

    # process each item in the definition string array and replace any found pseudo parameters
    # the final result is the ASL (State machine language)
    readarray -t arr < <(echo "${definition}"|tr -d '\r')
    for item in "${arr[@]}"; do
        itemToAppend=$item
        if [[ "${item}" =~ $partition ]]; then
            itemToAppend="aws"
        fi
        if [[ "${item}" =~ $accountId ]]; then
            itemToAppend="$AWS_ACCOUNT_ID"
        fi
        asl+="$itemToAppend"
    done

    echo $asl
}

# This function will download and start a local AWS Stepfunction docker image
# If an existing docker container is already running it will not create a new docker container
function start_step_function_container () {
    
    local image_name="amazon/aws-stepfunctions-local"

    if [ "$( docker inspect -f '{{.Config.Image}}' $(docker ps --format='{{.ID}}') )" == "$image_name" ]
    then
        echo "$image_name is already running"
        echo "continue without starting a new container"
    else
        # Run a local docker image of stepfunction, give it some invalid credentials since it requies credentials, but they don't need to be valid ones.
        CONTAINER_NAME=`docker run -d -p 8083:8083 --mount type=bind,readonly,source="$(pwd)"/MockConfigFile.json,destination=/home/stepfunctionslocal/MockConfigFile.json -e SFN_MOCK_CONFIG=MockConfigFile.json --env-file stepfunction-config.txt amazon/aws-stepfunctions-local`
        echo "$CONTAINER_NAME started"
        until [ "`docker inspect -f {{.State.Running}} $CONTAINER_NAME`"=="true" ]; do
            echo "wait for container to run..."
            sleep 0.1;
        done;
        echo "container started"
    fi
    sleep 4
}

# This function will create a state machine given the definition retrieved from the synthesized cloudformation template
function create_state_machine () {
    local definition=""
    definition=$1
    aws stepfunctions --endpoint http://localhost:8083 create-state-machine --name "MyStepFunction" --definition "${definition}" --role-arn "arn:aws:iam::123456789012:role/service-role/MyTestRole"
}

cfTemplateFile=$1
stepFunctionResourceId=$2

# lets get the ASL definition for the state machine from our synthesized CF template
asl=$(get_synth_definition ${cfTemplateFile} ${stepFunctionResourceId})


echo "asl: ${asl}"

# download and start the AWS Stepfunction docker image
start_step_function_container

# create the actual state machine given the ASL from previous step
create_state_machine "$asl"

# export the container name to be used from outside the script
export CONTAINER_NAME=$CONTAINER_NAME
{{< /collapsible>}}
\
Now you can execute the **stepfunction_local.sh** providing the following arguments:

```shell
./stepfunction_local.sh MyTemplate.json MyStepfunction
```

The command above starts a new Docker container (if not already running) with our local Step Functions container and creates a Step Function named **MyStepfunction** using the ASL definition found inside the **MyTemplate.json** file.

Once the stepfunctions have been created we can go ahead and start an execution of the stepfunction using the mocked steps we have provided in the *MockConfigFile.json* file:

```shell
aws stepfunctions start-execution \
    --endpoint http://localhost:8083 \
    --name executionWithHappyPathMockedServices \
    --state-machine arn:aws:states:eu-west-1:123456789012:stateMachine:MyStepfunction#HappyPath
```

To view the results of the execution you can use the following command:

```shell
aws stepfunctions get-execution-history \
    --endpoint http://localhost:8083 \
    --execution-arn arn:aws:states:eu-west-1:123456789012:execution:MyStepfunction:executionWithHappyPathMockedServices
```

## Final thoughts

AWS Step Functions provide a powerful way to orchestrate workflows and manage complex business processes. However, testing these workflows can be challenging, especially when dealing with intricate state machines and service integrations.

In this blog post, we explored how to leverage **AWS Step Functions Local** for local testing. Here are the key takeaways:

- AWS offers a downloadable version of Step Functions called Step Functions Local.
- It allows developers to create and test applications within their local development environment.
- Note that Step Functions Local uses dummy accounts for its operations.
- How to extract and reuse existing Amazon State Language (ASL) from within a CDK project.

In summary, AWS Step Functions Local is a valuable tool for local testing, but it’s essential to understand its limitations and use it appropriately. Happy testing!