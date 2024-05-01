+++
date = "2024-05-01T11:00:00-00:00"
title = "How to get around AWS WAF resource policy limits"
image = "/images/aws_waf_resource_policy.jpg"
+++

While working on a project involving AWS WAF, I encountered a rather peculiar issue related to AWS resource policy size limitations.

At first it seems odd but after a bit of mucking around the AWS documentation it started to make sense. Without further ado, let's get into the issue.

## Background

The project entailed hosting a Single Page Application (SPA) on Amazon S3, with a Web Application Firewall (WAF) deployed to regulate access to the site. See figure 1.

![](/images/waf_resource_policy.png "Figure 1: WAF setup with IP restriction rules to control access to SPA")

The WAF is configured to store access logs to AWS Cloudwatch Logs.

The setup is relatively simple and straightforward, with a CI/CD pipeline in place that utilizes AWS Cloud Development Kit (CDK) to automate the deployment of the infrastructure.

![](/images/cicd_pipeline.png "Figure 2: CI/CD pipeline to automate deployment to multiple environments")

When a pull request is opened, the CI/CD pipeline initiates the building and deployment of AWS resources to an AWS Account designated as the integration environment. This environment is dedicated to running integration tests against the deployed infrastructure. Once the tests are completed, the infrastructure is dismantled to maintain a clean and efficient testing environment.

Each infrastructure deployment is assigned a unique stack name prefixed with the commit ID. This approach enables multiple deployments to coexist within the environment without conflicts.

## WAF logs

With the context established, let's delve into the specific issue at hand. To enable the AWS WAF service to store its logs in CloudWatch Logs, we need to create a CloudWatch Log group.

It's essential to grasp the next part as it directly pertains to understanding the issue. AWS mandates that the log group must be named with a prefix of `aws-waf-logs-` see [WAF log group naming](https://docs.aws.amazon.com/waf/latest/developerguide/logging-cw-logs.html#logging-cw-logs-naming). As a consequence, every deployment of the WAF infrastructure generates a new log group with unique names, such as:

- aws-waf-logs-4ab9754
- aws-waf-logs-f311ad2
- ...

Subsequently, AWS includes a statement in the resource policy of the AWS CloudWatch Logs service, granting the aforementioned resources permissions to create log streams and put log events, see [enabling logging from AWS services](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AWS-logs-and-resource-policy.html). This occurs during the creation of the WAF infrastructure stack, as AWS attempts to deliver logs to the log group. If this process fails, the stack will not be successfully created.

Unfortunately, after some time, we encountered an issue where we were unable to create the WAF infrastructure stack in our integration environment. The error message indicated that the resource policy for the logs had exceeded the maximum limit of 5120 characters.

## Size does matter!

So there is a hard limit of 5120 characters for a Cloudwatch Logs resource policy. To verify if that truly was the case we issued the following command:

```bash
aws logs describe-resource-policies
```

The following is a condensed version of the output for highlighting purposes. In reality, the resource policy exceeds the maximum limit of 5120 characters:

```bash
{
    "resourcePolicies": [
        {
            "policyName": "AWSLogDeliveryWrite20150319",
            "policyDocument": {
				"Version":"2012-10-17",
				"Statement":[
					{
						"Sid":"AWSLogDeliveryWrite",
						"Effect":"Allow",
						"Principal":{
							"Service":"delivery.logs.amazonaws.com"
						},
						"Action":[
							"logs:CreateLogStream",
							"logs:PutLogEvents"
						],
						"Resource":[
							"arn:aws:logs:us-east-1:xxxxx:log-group:aws-waf-logs-4ab9754:log-stream:*",
							"arn:aws:logs:us-east-1:xxxxx:log-group:aws-waf-logs-f311ad2:log-stream:*",
							"arn:aws:logs:us-east-1:xxxxx:log-group:aws-waf-logs-cb3ad94:log-stream:*"
						]
					}
				]
			}
		}
	]
}
```

As observed, the resource policy dynamically includes the names of our WAF log groups in its list of resources. This mechanism grants the AWS WAF service the necessary permissions to publish its logs to the corresponding CloudWatch log groups.
After multiple deployments of our WAF infrastructure stack, AWS actively monitors the resource policy size. If the policy exceeds the 5120 characters limit during stack deployment, AWS prevents the creation of the WAF.

## What about /aws/vendedlogs/?

You may wonder why we don't simply adhere to the naming convention `/aws/vendedlogs/` as described in the AWS documentation [enabling logging from AWS services](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AWS-logs-and-resource-policy.html) to circumvent the resource policy limit?

The reason for this is quite simple: we are unable to do so. As mentioned earlier in this post, AWS mandates that the WAF log group must be named with the prefix `aws-waf-logs-`. Therefore, using the `/aws/vendedlogs/` naming convention is not an option for us.

## Solution

The solution we devised is rather straightforward. We simply added another resource policy to AWS CloudWatch Logs that permits wildcards in the `aws-waf-logs-*` name.

This approach streamlines the resource policy by requiring only one statement to cover all log groups starting with `aws-waf-logs-`. AWS will monitor the resource policy and refrain from adding additional statements, as the existing wildcard statement already encompasses any newly created log groups with the specified prefix.

To do this you could use the AWS command

```bash
aws logs put-resource-policy
```

But we created a CDK construct for the new resource policy like so in Typescript:

```js
export class WafResourcePolicy extends Construct {
    constructor(scope: Construct, id: string) {
        super(scope, id)

        const policy = new cdk.aws_logs.ResourcePolicy(this, `${id}-waf-resource-policy`, {
            policyStatements: [
                new cdk.aws_iam.PolicyStatement({
                    actions: [
                        'logs:CreateLogStream',
                        'logs:PutLogEvents',
                    ],
                    resources: [
                        cdk.Stack.of(this).formatArn({
                            arnFormat: cdk.ArnFormat.COLON_RESOURCE_NAME,
                            service: 'logs',
                            resource: 'log-group',
                            resourceName: 'aws-waf-logs-*',
                        }),
                    ],
                    principals: [new cdk.aws_iam.ServicePrincipal('delivery.logs.amazonaws.com')],
                    conditions: {
                        StringEquals: {
                            'aws:SourceAccount': [cdk.Stack.of(this).account]
                        },
                        ArnLike: {
                            'aws:SourceArn': [`arn:aws:logs:${cdk.Stack.of(this).region}:${cdk.Stack.of(this).account}:*`],
                        },
                    },
                    effect: cdk.aws_iam.Effect.ALLOW,
                }),
            ],
        })
        
        policy.applyRemovalPolicy(cdk.RemovalPolicy.DESTROY)
    }
}
```

This construct will be deployed once in the integration environment, and this is how the resource policy will appear after the command is executed again:

```bash
aws logs describe-resource-policies
```

output:

```bash
{
    "resourcePolicies": [
        {
            "policyName": "AWSLogDeliveryWrite20150319",
            "policyDocument": {
				"Version":"2012-10-17",
				"Statement":[
					{
						"Sid":"AWSLogDeliveryWrite",
						"Effect":"Allow",
						"Principal":{
							"Service":"delivery.logs.amazonaws.com"
						},
						"Action":[
							"logs:CreateLogStream",
							"logs:PutLogEvents"
						],
						"Resource":[
						]
					}
				]
			}
		},
		{
            "policyName": "WafResourcePolicy8865B63E",
			"policyDocument": {
				"Version":"2012-10-17",
				"Statement":[
					{
						"Sid":"AWSLogDeliveryWrite",
						"Effect":"Allow",
						"Principal":{
							"Service":"delivery.logs.amazonaws.com"
						},
						"Action":[
							"logs:CreateLogStream",
							"logs:PutLogEvents"
						],
						"Resource": "arn:aws:logs:us-east-1:xxxxxx:log-group:aws-waf-logs-*",
						"Condition": {
							"StringEquals": {
								"aws:SourceAccount": "xxxxxx"
							},
							"ArnLike": {
								"aws:SourceArn": "arn:aws:logs:us-east-1:xxxxxx:*"
							}
						}
					}
				]
			}
        }
	]
}
```

We now have an additional resource policy containing our wildcard statement, alongside the previous resource policy. Importantly, no new resource statements were added to the original policy.

It's worth noting that each AWS account is limited to a maximum of 10 resource policies per region.

## Conclusion

In this post, we addressed the limitations of AWS resource policies, including the **5120-character** constraint and the mandatory naming convention for WAF log groups (`aws-waf-logs-`). This could potentially pose challenges when repeatedly creating WAF resources, leading to eventual limitations on deployment.

However, we demonstrated a solution by introducing an additional resource policy that allows wildcards in the log group names. This prevents AWS from appending new resources to the policy, thus avoiding the 5120-character limitation.