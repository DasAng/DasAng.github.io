+++
date = "2024-05-30T07:00:00-00:00"
title = "OPC UA realtime streaming in Grafana: Part 1"
image = "/images/grafana_opcua.jpg"

+++

Welcome to the first installment of our series, where we’ll develop a Grafana plugin capable of streaming realtime data from any **OPC Unified Architecture (OPC UA)** server. In my prior projects, I’ve handled data collection from various IoT devices using the OPC UA client/server framework for seamless data exchange between devices and applications. The data was visualized and monitored using [Grafana](https://grafana.com/)

![](/images/grafana_dashboard.png "Figure 1: An example visualization of IoT sensor in Grafana")

Data collection from OPC UA servers, such as Kepware, IFM Moneo, and AspenTech Inmation, can be achieved using OPC UA clients like **Influx Telegraf, AWS IoT SiteWise, and Greengrass**, or through custom application code.

But there was also a need to empower subject matter experts (SMEs) to connect directly to the OPC UA servers using their preferred tool, such as **Grafana**, without requiring intricate and expensive infrastructure setups.

## Don't reinvent the wheel or do we?

After extensive research on the web and the official Grafana site, I discovered the [Grafana OPC UA plugin](https://grafana.com/docs/plugins/grafana-opcua-datasource/latest/). Unfortunately, this plugin was developed for an older Grafana version and is no longer maintained—it has been deprecated, as indicated in the [Github repo](https://github.com/grafana/opcua-datasource). Additionally, it lacks password authentication, which is a critical requirement for some of the OPC UA servers.

That gives me no choice but to build my own plugin. The result is an open-source OPC UA grafana plugin at this [Github repository]() (**COMING SOON**). You can refer to the source code to understand how I’ve implemented the plugin.

## Inside look of the Grafana OPC UA plugin

So in this series I'll show you how I have built a data source plugin for Grafana that enables the following functionalities:

- Connect to an OPC UA server
- Authenticate against OPC UA server using Anonymous or Password/Username methods
- Support encrypted communication between Grafana and OPC UA server
- Browse nodes on the OPC UA server
- Stream real-time data
- Visualize data
- Provide alerting capabilities

## What we will cover in Part 1

In this part of the series we will be covering the following topics:

- Overview of Grafana plugin architecture
- Tools needed to start development
- Start Grafana locally in a Docker Container

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
- `create-plugin` (A CLI to scaffold new plugins)
- `sign-plugin` (A CLI to sign plugins for distribution)

If you’re developing on a Windows operating system, I recommend installing the following tools for smoother local development

- **Docker Desktop**
- **Windows Subsystem For Linux (WSL)**

You’ll need WSL because the `create-plugin` tool only supports WSL. See this [documentation](https://grafana.com/developers/plugin-tools/get-started/set-up-development-environment/) on how to setup a development environment.


## Scaffold new plugin

The first step is to scaffold a new plugin. To do this we will need to install the `create-plugin` tool. Execute the following command:

{{< codetabs>}}
{{< codetab npm shell>}}
npx @grafana/create-plugin@latest
{{< /codetab>}}
{{< codetab yarn shell>}}
yarn create @grafana/plugin
{{< /codetab>}}
{{< /codetabs>}}

Follow the prompts to create a data source plugin with a backend. Give it a name like for example *grafana-opcuasource*. Once you have scaffolded the plugin you should have a folder structure similar to this

{{< collapsible shell>}}
grafana-opcuasource-datasource/
├── .config/
├── cypress
│   ├── integration
│   │   ├── 01-smoke.spec.ts
├── pkg
│   ├── main.go
│   └── plugin
├── provisioning
│   ├── datasources
│   │   ├── .gitkepp
│   │   ├── .datasources.yml
│   ├── READNE.md
├── src
│   ├── README.md
│   ├── components
│   │   ├── ConfigEditor.tsx
│   │   ├── QueryEditor.tsx
│   ├── datasource.ts
│   ├── img
│   │   ├── logo.svg
│   ├── module.ts
│   ├── plugin.json
│   └── types.ts
├── .eslintrc
├── .gitignore
├── .nvmrc
├── .prettierrc.js
├── cypress.json
├── CHANGELOG.md
├── LICENSE
├── Magefile.go
├── README.md
├── docker-compose.yaml
├── go.mod
├── go.sum
├── jest-setup.js
├── jest.config.js
├── node_modules
├── package.json
├── tsconfig.json
{{< /collapsible>}}

You are now ready to start development.

## Run Grafana in Docker Container

To develop and test our plugin, we require Grafana to be running in our local development environment. I’ll use Docker Desktop for this purpose since I’m working on a Windows environment. However, if you’re using MacOS or Linux, you can run Grafana in your own Docker environment.

Start Docker Desktop and then execute the following command on your terminal:

{{< codetabs>}}
{{< codetab npm shell>}}
npm run server
{{< /codetab>}}
{{< codetab yarn shell>}}
yarn server
{{< /codetab>}}
{{< /codetabs>}}

This command will execute `docker-compose`, which will download the Grafana Docker image if it’s not already present, and then create a Docker container using that image.

![](/images/grafana_docker_image.png "Figure 2: Docker Desktop showing the newly downloaded Grafana Docker image")

![](/images/grafana_docker_container.png "Figure 3: A Docker container has been created and is running the Grafana software")

Now open a browser and navigate to `http://localhost:3000` and you will be able to to see the Grafana dashboard. Navigate to the menu **Connection->Data sources** and you can see your newly created data source plugin displayed

![](/images/grafana_data_connection.png "Figure 4: Our data source plugin is displayed in the list of data sources available")

## Conclusion

In this part of our series, I’ve covered the various plugin types in Grafana, and how to scaffold a datasource plugin. Additionally, I've shown how to run Grafana locally inside a Docker container.

This concludes part 1 of our series. In [part2]({{< ref "grafana_opcua_part2" >}}), I’ll guide you through the basic structure of a data source plugin.