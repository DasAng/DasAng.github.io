+++
date = "2024-05-02T12:00:00-00:00"
title = "Troubleshoot network connection with traffic mirroring"
image = "/images/traffic_mirroring.jpg"

+++

Sometimes you have to look at the network packets to really understand what is going on. This is exactly what happened during a project involving misconfigured network setup from on-premise to AWS VPC. I'll walk you through my experience of how AWS traffic mirroring helped me get to the root cause.

## Background

The client had chosen AWS Redshift as their data warehousing solution, hosting it within an AWS VPC. To seamlessly integrate it with their on-premise network, they leveraged AWS Transit Gateway for secure and efficient connectivity.

![](/images/redshift_onpremise.png "Figure 1: On-premise application connected to Redshift via Transit Gateway")

Their custom application, operating within their on-premise network, establishes connections directly with Redshift for seamless data interaction. This application is running Python and connects to the Redshift cluster using Psycopg2 library.

Everything was smooth sailing for weeks until they underwent a switch and router replacement in their network. Suddenly, the application hit a snag and couldn't connect to the Redshift cluster—or so we initially believed.

## The investigation begins

The investigation began with a deep dive into the Python code itself, searching for any potential issues. Despite its apparent simplicity, this crucial snippet responsible for connecting to the Redshift cluster consistently timed out:

```python
def connect():
    secret_arn = ssm.get_parameter(
        Name="/redshift_secret_arn", WithDecryption=True
    )["Parameter"]["Value"]
    secret_dict = get_secret_dict(secret_arn)
    db_host = secret_dict["host"]
    db_port = secret_dict["port"]
    db_name = secret_dict["dbname"]
    db_user = secret_dict["username"]
    db_password = secret_dict["password"]
    con = psycopg2.connect(
        dbname=db_name, host=db_host, port=db_port, user=db_user, password=db_password
    )
    con.autocommit = True
```

Since there were no recent changes to the application code, we ruled out the application itself as the culprit. To further diagnose the issue, we attempted to create a new Redshift cluster and connect to it instead. Surprisingly, we encountered the same connection timeout, deepening the mystery.

With no clear solution in sight, it was time to get our hands dirty and delve into the world of network packets.

## Open the packets please

Equipped with Wireshark installed on the on-premise network, we commenced capturing and analyzing the network traffic traversing in and out of the application.