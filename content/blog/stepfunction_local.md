+++
date = "2024-05-08T15:00:00-00:00"
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