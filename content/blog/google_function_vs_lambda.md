+++
date = "2024-06-04T21:00:00-00:00"
title = "Google Cloud Functions versus AWS Lambda: Pricing nuances"
image = "/images/google_cloud_function_lambda.jpg"

+++

In this short post, I’ll delve into the nuances of pricing when deploying Google Cloud Functions in comparison to AWS Lambda.

When evaluating technology or cloud providers, you frequently consider their offered functionality and associated pricing. In this scenario, I will examine two cloud providers: Google Cloud Platform (GCP) and Amazon Web Services (AWS), focusing on their serverless offerings in the form of Function-as-a-Service (FaaS). I will primarily concentrate on the pricing aspect.

## AWS Lambda

Amazon Web Services (AWS) provides **Lambda Functions** as their Function-as-a-Service (FaaS) offering. You can write your function in any of the supported programming languages and deploy the source code as a zip file or provide a Docker image.

You can find a detailed pricing table [here](https://aws.amazon.com/lambda/pricing/)

In short you pay for the amount of requests and the duration of the execution.

## Google Cloud Function

Google has a similar offering called **Cloud Functions** which is divided into *1st gen* and *2nd gen* functions. For a detailed pricing table see [here](https://cloud.google.com/functions/pricing).

The pricing model is similar as **AWS Lambda functions**, where you pay for the amount of requests and the duration of the execution.

When deploying a Cloud Function, GCP constructs a container from your source code, a process facilitated by the **Google Cloud Build** service. You are charged for the amount of **Cloud Build** usage. This implies that you will incur additional charges associated with the Cloud Build service.

Even if you attempt to reduce deployment time by using prebuilt source code through your CI/CD pipeline or locally precompile your code, the Cloud Build service will still be triggered, resulting in charges for Cloud Build usage.

This **'hidden'** fee is often overlooked when reviewing the pricing.

## Final thoughts

In this post, I’ve outlined the pricing models for using AWS Lambda functions versus Google Cloud Functions and emphasized their similarities. Additionally, I’ve highlighted the additional charges associated with deploying Google Cloud Functions.

Selecting a technology and evaluating pricing models are crucial steps. Utilizing a price calculator provided by the vendor can significantly aid in the assessment.