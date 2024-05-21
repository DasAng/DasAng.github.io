+++
date = "2024-05-21T07:00:00-00:00"
title = "OPC UA data source plugin in Grafana: Part 1"
image = "/images/grafana_opcua.jpg"

+++

Welcome to the first installment of our series, where we’ll develop a Grafana plugin capable of streaming real-time data from any **OPC Unified Architecture (OPC UA)** server. In my prior projects, I’ve handled data collection from various IoT devices using the OPC UA client/server framework for seamless data exchange between devices and applications. The data was visualized and monitored using [Grafana](https://grafana.com/)

![](/images/grafana_dashboard.png "Figure 1: An example visualization of IoT sensor in Grafana")

Data collection from OPC UA servers, such as Kepware, IFM Moneo, and AspenTech Inmation, can be achieved using OPC UA clients like **Influx Telegraf, AWS IoT SiteWise, and Greengrass**, or through custom application code.

But there was also a need to empower subject matter experts (SMEs) to connect directly to the OPC UA servers using their preferred tool, such as **Grafana**, without requiring intricate and expensive infrastructure setups.

## Don't reinvent the wheel or do we?

After extensive research on the web and the official Grafana site, I discovered the [Grafana OPC UA plugin](https://grafana.com/docs/plugins/grafana-opcua-datasource/latest/). Unfortunately, this plugin was developed for an older Grafana version and is no longer maintained—it has been deprecated, as indicated in the [Github repo](https://github.com/grafana/opcua-datasource). Additionally, it lacks password authentication, which is a critical requirement for some of the OPC UA servers.

That gives me no choice but to build my own plugin. The result is an open-source OPC UA grafana plugin at this [Github repository](). You can refer to the source code to understand how I’ve implemented the plugin.

## What we will be building

So in this series we will be building a data source plugin for Grafana that enables the following functionalities:

- Connect to an OPC UA server
- Authenticate against OPC UA server using Anonymous or Password/Username methods
- Support encrypted communication between Grafana and OPC UA server
- Browse nodes on the OPC UA server
- Stream real-time data
- Visualize data
- Provide alerting capabilities

## What we will cover in Part 1

In this part of the series we will be covering the following topics:

- Overview of Grafana data source plugin architecture
- Tools needed to start development

## The foundation

As with any topic, you start with the foundation, which means reading through the Grafana documentation on how to build a plugin.

[Grafana plugin development documentation](https://grafana.com/developers/plugin-tools/introduction/)

Based on my experience, the documentation is excellent for getting started, but it could benefit from additional examples and more in-depth information on topics such as streaming data, plugin protocol and the workings of alerting queries.

## Plugin types

Grafana allows you to extend its functionality through the following plugin types:

1. **Data Source Plugins**:
   - Data source plugins connect to external systems (databases, APIs, etc.) and retrieve data in a format that Grafana understands.
   - By adding a data source plugin, you can use the data in your existing dashboards.
   - Grafana launches each plugin as a subprocess and communicates with it over gRPC.

2. **Panel Plugins**:
   - Panel plugins visualize data within Grafana dashboards. They create charts, graphs, and other visual elements.
   - These plugins allow you to visualize data returned by data source queries and control external systems.

3. **App Plugins**:
   - App plugins bundle data sources and panels to provide cohesive experiences. For example, the Prometheus and Kubernetes apps.
   - Use app plugins to create custom, out-of-the-box monitoring experiences.

In this series we will be creating a **Data Source Plugin** that can connect directly with any OPC UA server.

## Prerequisites

To build our plugin, we need to install the following tools and software, you can get more information from [plugin tools](https://grafana.com/developers/plugin-tools/):

- Go (The Golang programming language)
- Mage (https://magefile.org/)
- NodeJs
- create-plugin (the cli to scaffold a new plugin)

If you’re developing on a Windows operating system, I recommend installing the following tools for smoother local development

- **Docker Desktop**
- **Windows Subsystem For Linux (WSL)**

You’ll need WSL because the `create-plugin` tool only supports WSL. See this [documentation](https://grafana.com/developers/plugin-tools/get-started/set-up-development-environment/) on how to setup a development environment.
