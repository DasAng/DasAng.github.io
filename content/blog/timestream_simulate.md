+++
date = "2024-05-07T08:00:00-00:00"
title = "Time travelling with AWS Timestream: Simulate data in the future"
image = "/images/timestream_simulator.jpg"

+++

No, this isn’t the next movie title from *Back to the Future*, although it would be pretty cool if they made another film. In this post, I’ll share a project where I’ve used AWS Timestream to store both real-time and historical data. We’ll explore how it can be used to replay historical data in the present or even predict future scenarios.

## What is AWS Timestream

To provide a brief overview of Timestream, it’s an AWS serverless service that allows you to ingest and store time-series data for subsequent analytical queries. Think of it as a managed version of **InfluxDB**. As of this writing AWS now also offers *InfluxDB* as a managed service (not serverless though).

## Background

In this project IoT data were collected and ingested into AWS Timestream using AWS IoT Core rules. The data would then be consumed by microservices implemented as AWS Lambda for further analysis. Finally the data would then be visualized using AWS Managed Grafana and monitored.

![](/images/timestream_background.png "Figure 1: IoT Data ingested into AWS Timestream where it is being used by microservices")

### E2E test

An automated end-to-end (E2E) test ensures that the entire system functions as expected and can be run on-demand. The test would be responsible for ingesting live data through the system, and the **idempotent** operation ensures that the system’s success can be reliably verified.

To achieve this, we've established an additional Timestream database for ingesting historical data for various scenarios, and during end-to-end testing, the microservices seamlessly transition to utilizing the new Timestream.

![](/images/timestream_e2e_test.png "Figure 2: E2E test where historical data are replayed into a new Timestream database")

In this manner, the original Timestream database can continue to function normally, receiving live data without any contamination from simulated data.