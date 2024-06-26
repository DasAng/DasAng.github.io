+++
date = "2024-06-26T14:00:00-00:00"
title = "OPC UA realtime streaming in Grafana: Part 3"
image = "/images/grafana_opcua.jpg"

+++

Welcome to the third installment of our series, where we’ll develop a Grafana plugin capable of streaming realtime data from any **OPC Unified Architecture (OPC UA)** server. In my prior projects, I’ve handled data collection from various IoT devices using the OPC UA client/server framework for seamless data exchange between devices and applications and there was a need for direct integration between Grafana and OPC UA streaming.

## What we will cover in Part 3

In [part2]({{< ref "grafana_opcua_part2" >}}) we've explained the basic structure of a Grafana plugin and how the backend and frontend part of the plugin communicates through Grafana Server. In this part of the series I will guide you through the implementation of the backend component.

- [OPC-UA library](#opc-ua-library)
- [Plugin folder structure](#plugin-folder-structure)
- [Datasource lifecycle Management](#datasource-lifecycle-management)
- [Self-signed client SSL certificate](#self-signed-client-ssl-certificate)
- [Health check](#health-check)
- [Frontend communication](#frontend-communication)
- [Browsing OPC-UA nodes](#browsing-opc-ua-nodes)
- [Streaming data](#streaming-data)
- [Alerting](#alerting)

## OPC-UA library

To implement the OPC-UA binary protocol I will be using the following open source [Go OPC-UA](https://github.com/gopcua/opcua) library from Github.

## Plugin folder structure

The following represents the folder structure for the backend code

{{< collapsible shell>}}
grafana-opcuasource-datasource/
├── pkg
│   ├── plugin
│   │   ├── api.go
│   │   ├── browse.go
│   │   ├── dataframe.go
│   │   ├── datasource.go
│   │   ├── datatype.go
│   │   ├── health.go
│   │   ├── query.go
│   │   ├── security.go
│   │   └── streams.go
│   └── main.go
{{< /collapsible>}}

## Datasource lifecycle Management

The **main.go** file serves as the entry point for our plugin, handling both its instantiation and lifecycle.

{{< collapsible go>}}
// main.go
package main

import (
	"os"

	"github.com/grafana/grafana-plugin-sdk-go/backend/datasource"
	"github.com/grafana/grafana-plugin-sdk-go/backend/log"
	"github.com/ang/opc-ua-source/pkg/plugin"
)

func main() {
	// Start listening to requests sent from Grafana. This call is blocking so
	// it won't finish until Grafana shuts down the process or the plugin choose
	// to exit by itself using os.Exit. Manage automatically manages life cycle
	// of datasource instances. It accepts datasource instance factory as first
	// argument. This factory will be automatically called on incoming request
	// from Grafana to create different instances of Datasource (per datasource
	// ID). When datasource configuration changed Dispose method will be called and
	// new datasource instance created using NewDatasource factory.
	if err := datasource.Manage("dasang-opcuasource-datasource", plugin.NewDatasource, datasource.ManageOpts{}); err != nil {
		log.DefaultLogger.Error(err.Error())
		os.Exit(1)
	}
}
{{< /collapsible>}}
\
The above code is responsible for starting a **gRPC server** to listen for messages sent by the **Grafana Server**, in addition it handles the instantiation of new Datasources using the factory method **NewDatasource**. We will need to provide the implementation of the **NewDataSource** method in our code.

![](/images/datasource_instance.png "Figure 1: Datasource lifecycle")

Let's take a look at how we implement the **NewDatasource** method. The implementation will be inside the **datasource.go** file.

{{< collapsible go>}}
// datasource.go
package plugin

import (
	"context"
	"encoding/json"

	"github.com/grafana/grafana-plugin-sdk-go/backend"
	"github.com/grafana/grafana-plugin-sdk-go/backend/instancemgmt"
	"github.com/grafana/grafana-plugin-sdk-go/backend/log"

	"github.com/gopcua/opcua"
	"github.com/gopcua/opcua/ua"
)

// Make sure Datasource implements required interfaces. This is important to do
// since otherwise we will only get a not implemented error response from plugin in
// runtime. In this example datasource instance implements backend.QueryDataHandler,
// backend.CheckHealthHandler interfaces. Plugin should not implement all these
// interfaces - only those which are required for a particular task.
var (
	_ backend.QueryDataHandler      = (*Datasource)(nil)
	_ backend.CheckHealthHandler    = (*Datasource)(nil)
	_ instancemgmt.InstanceDisposer = (*Datasource)(nil)
	_ backend.CallResourceHandler   = (*Datasource)(nil)
	_ backend.StreamHandler         = (*Datasource)(nil)
)

// This is our datasource type, it contains a client connection to OPC-UA server and other settings.
type Datasource struct {
	client  *opcua.Client
	options *Settings
}

// Settings for connecting to OPC-UA server
type Settings struct {
	Endpoint       string
	SecurityPolicy string
	SecurityMode   string
	AuthMode       string
	Username       string
	Password       string
}

type Query struct {
	QueryNodes      []QueryNodeRequest
	RefreshInterval int
}

type ReadNodeRequest struct {
	NodeId string
	Path   string
}

// This factory method is responsible for instantiating a new Datasource instance
func NewDatasource(_ context.Context, settings backend.DataSourceInstanceSettings) (instancemgmt.Instance, error) {
	ms := Settings{"", "None", "None", "Anonymous", "", ""}
	var client *opcua.Client
	var secOpts []opcua.Option
	opt, err := settings.JSONData.MarshalJSON()
	if err != nil {
		log.DefaultLogger.Error("Could not parse settings: ", "error", err.Error())
	} else {
		json.Unmarshal(opt, &ms)
		ms.Username = settings.DecryptedSecureJSONData["userName"]
		ms.Password = settings.DecryptedSecureJSONData["password"]
		secOpts = append(secOpts, opcua.SecurityModeString(ms.SecurityMode), opcua.SecurityPolicy(ms.SecurityPolicy))
		if ms.AuthMode == "Anonymous" {
			secOpts = append(secOpts, opcua.AuthAnonymous())
		}
		if ms.AuthMode == "UserName" {
			secOpts = append(secOpts, opcua.AuthUsername(ms.Username, ms.Password))
		}

		pk, cert, err := CreateOrLoadCertificate(settings.Name)
		if err != nil {
			log.DefaultLogger.Error(err.Error())
		} else {
			secOpts = append(secOpts, opcua.PrivateKey(pk), opcua.Certificate(cert))
		}

		log.DefaultLogger.Info("Getting endpoint: ", "endpoint", ms.Endpoint)
		eps, err := opcua.GetEndpoints(context.Background(), ms.Endpoint)
		if err != nil {
			log.DefaultLogger.Error("Failed to get endpoint:", "error", err.Error())
		} else {
			log.DefaultLogger.Info("Endpoint received:", "endpoint", eps)
			ep := opcua.SelectEndpoint(eps, ua.FormatSecurityPolicyURI(ms.SecurityPolicy), ua.MessageSecurityModeFromString(ms.SecurityMode))
			secOpts = append(secOpts, opcua.SecurityFromEndpoint(ep, ua.UserTokenTypeFromString(ms.AuthMode)))
		}

		client, err = opcua.NewClient(ms.Endpoint, secOpts...)
		if err != nil {
			log.DefaultLogger.Error("Could not create OPC-UA client", "error", err.Error())
		}

	}

	return &Datasource{client, &ms}, nil
}

// Dispose here tells plugin SDK that plugin wants to clean up resources when a new instance
// created. As soon as datasource settings change detected by SDK old datasource instance will
// be disposed and a new one will be created using NewSampleDatasource factory function.
func (d *Datasource) Dispose() {
	// Clean up datasource instance resources.
	if d.client != nil {
		d.client.Close(context.Background())
	}
}
{{< /collapsible>}}
\
The **Datasource** struct is instantiated by the **NewDatasource** method and includes a reference to the OPC-UA client connector, along with settings for connecting to the OPC-UA server.

It is important that we implement the following interfaces to handle streaming, health checks, and query capabilities for our datasource:

- QueryDataHandler
- CheckHealthHandler
- StreamHandler
- CallResourceHandler
- InstanceDisposer

We will get back to the above interface implementations later in this post. But for now we will ensure that our datasource have registered to implement the above interfaces. Let's break down the implementation of the **NewDatasource** method.

1. **Parsing Settings**:
   - The function receives settings from the frontend component in the form of `backend.DataSourceInstanceSettings` struct and save the settings.
   - It also retrieves the username and password from the `DecryptedSecureJSONData` field which are stored encrypted.

2. **Certificate Handling**:
   - The function calls `CreateOrLoadCertificate` to either load an existing Client SSL Certificate or create a new one.

3. **Endpoint Handling**:
   - The function will try and get tha available security endpoints for the OPC-UA server.

4. **Creating OPC-UA Client**:
   - Finally, it creates an OPC-UA client using the specified endpoint and security options.
   - If any error occurs during this process, it logs an error message.

5. **Instantiates datasource**:
   - The function returns an instance of the `Datasource` struct

![](/images/newdatasource_method.png "Figure 2: NewDatasource method. Parse settings received from frontend and instantiates a new datasource")

## Self-signed client SSL certificate

In our implementation of NewDatasource, we also manage the client SSL certificate. The method responsible for managing the certificate is called `CreateOrLoadCertificate` and is implemented in the **security.go** file.

{{< collapsible go>}}
// security.go
package plugin

import (
	"crypto/rsa"
	"crypto/tls"
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"runtime"

	"github.com/grafana/grafana-plugin-sdk-go/backend/log"
)

func CreateOrLoadCertificate(certName string) (*rsa.PrivateKey, []byte, error) {
	ex, err := os.Executable()
	if err != nil {
		return nil, nil, fmt.Errorf("could not get plugin executable: %s", err.Error())
	}
	exPath := filepath.Dir(ex)
	log.DefaultLogger.Info("Plugin exectuable path:", "path", exPath)

	certFile := fmt.Sprintf("%s/%s.pem", exPath, certName)
	keyfile := fmt.Sprintf("%s/%s.key", exPath, certName)
	log.DefaultLogger.Info("Check if certificate file exists on disk:", "certFile", certFile)
	log.DefaultLogger.Info("Check if private key file on disk:", "keyFile", keyfile)
	certFileExists, keyFileExists := true, true
	if _, err := os.Stat(certFile); err != nil {
		certFileExists = false
	}
	if _, err := os.Stat(certFile); err != nil {
		keyFileExists = false
	}

	if !keyFileExists && !certFileExists {
		log.DefaultLogger.Info("Could not find any existing certificate and private key")
		certExe := "./certificate"
		if runtime.GOOS == "windows" {
			certExe = "./certificate.exe"
		} else if runtime.GOOS == "linux" {
			certExe = "./certificate"
		}
		log.DefaultLogger.Info("Generate new certificate and private key with name: ", "name", certName)
		cmnd := exec.Command(certExe, certName)
		cmnd.Dir = exPath
		out, err := cmnd.Output()
		if err != nil {
			return nil, nil, fmt.Errorf("error from running certificate executable: %s", err.Error())
		}
		log.DefaultLogger.Info("Result from generating new certificate:", "output", string(out))
	}

	if _, err := os.Stat(certFile); err != nil {
		return nil, nil, fmt.Errorf("failed to generate certificate file: %s", err)
	}
	if _, err := os.Stat(certFile); err != nil {
		return nil, nil, fmt.Errorf("failed to generate private key file: %s", err)
	}

	c, err := tls.LoadX509KeyPair(certFile, keyfile)
	if err != nil {
		return nil, nil, fmt.Errorf("failed to load x509 certificate: %s", err)
	} else {
		pk, ok := c.PrivateKey.(*rsa.PrivateKey)
		if !ok {
			return nil, nil, fmt.Errorf("invalid private key")
		}
		cert := c.Certificate[0]
		return pk, cert, nil
	}
}
{{< /collapsible>}}
\
So when connecting to OPC-UA server that requires encryption and signed messages we will need to provide a client SSL certificate that can be used for the signed and encrypted communication. Each datasource will have it's own client certificate and the method above is responsible for detecing whether a client certificate for a datasource has already been created and if not it will try and create a new certificate for that datasource. The certificate will be a self-signed certificate using SHA256 hashing algorithm.

The actual certificate generation is done by an external executable program named `certificate` or in windows `certificate.exe`. This program is a **Python** program that I have written to manage the creation of self-signed certificates and is packaged as a standalone executable binary. The executable will be packaged alongside our grafana plugin.

The method will spawn the `certificate` executable providing the name of the certificate to generate. The executable will then create a private key and certificate and store them on disk.

## Health check

Remember earlier that we have registered our datasource to implement the `CheckHelathHandler` interface. This interface has a single method named `CheckHealth` which will be called by the Grafana server to check the health of our datasource. This method will also be called whenever we press the test button on the datasource configuration page in the frontend.

![](/images/grafana_datasource_test_button.png "Figure 3: Datasource configuration page with Save & Test button")

Let's look at the implementation of `CheckHealth` which is inside the **health.go** file.

{{< collapsible go>}}
// health.go
package plugin

import (
	"context"
	"errors"
	"fmt"
	"time"

	"github.com/gopcua/opcua"
	"github.com/grafana/grafana-plugin-sdk-go/backend"
	"github.com/grafana/grafana-plugin-sdk-go/backend/log"
)

// CheckHealth handles health checks sent from Grafana to the plugin.
// The main use case for these health checks is the test button on the
// datasource configuration page which allows users to verify that
// a datasource is working as expected.
func (d *Datasource) CheckHealth(ctx context.Context, req *backend.CheckHealthRequest) (*backend.CheckHealthResult, error) {
	var status = backend.HealthStatusOk
	var message = "Data source is working"
	log.DefaultLogger.Info("Checking datasource health")

	err := d.CheckconnectionHealth(ctx)
	if err != nil {
		return &backend.CheckHealthResult{
			Status:  backend.HealthStatusError,
			Message: err.Error(),
		}, nil
	}

	return &backend.CheckHealthResult{
		Status:  status,
		Message: message,
	}, nil
}

func (d *Datasource) CheckconnectionHealth(ctx context.Context) error {

	if d.client == nil {
		log.DefaultLogger.Error("Could not create OPC-UA client")
		return errors.New("could not create OPC-UA client")
	}

	if state := d.client.State(); state == opcua.Connected {
		log.DefaultLogger.Info("Already connected to data source")
		return nil
	}

	if err := d.client.Connect(ctx); err == nil {
		log.DefaultLogger.Info("Connected successfully to data source")
		return nil
	} else {
		log.DefaultLogger.Error("Could not connect to data source:", "error", err.Error())
	}

	ticker := time.NewTicker(time.Duration(3) * time.Second)
	defer ticker.Stop()

	maxConnectionAttempts := 5
	log.DefaultLogger.Info(fmt.Sprintf("Attempting connection %d more times before giving up...", maxConnectionAttempts))

	for i := 0; i < maxConnectionAttempts; i++ {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-ticker.C:

			if state := d.client.State(); state != opcua.Connected {
				log.DefaultLogger.Info(fmt.Sprintf("Retry connection to data source attempt %d...", i+1))
				if err := d.client.Connect(ctx); err != nil {
					log.DefaultLogger.Error("Could not connect to data source", "error", err.Error())
				} else {
					log.DefaultLogger.Info("Connected successfully to data source")
					return nil
				}
			} else {
				log.DefaultLogger.Info("Connected successfully to data source")
				return nil
			}
		}
	}
	return errors.New("failed to establish connection to OPC-UA server")
}
{{< /collapsible>}}
\
The health check mechanism ensures that the OPC-UA server connection is functional. Here's how it works:

1. **Purpose**:
   - The health check verifies whether the OPC-UA server is reachable and can establish a connection.

2. **Retry Strategy**:
   - The health check attempts to connect to the OPC-UA server.
   - If the initial connection fails, it retries every 3 seconds.
   - It makes up to 5 attempts before giving up.
   - This retry strategy helps handle transient issues (e.g., network glitches) and ensures robustness.

## Frontend communication

Now that we've set up the basic datasource lifecycle implementation and health checks, our next step is to manage communication between the frontend and backend. In Part 2, we discussed how frontend and backend components communicate via the Grafana server. Specifically, the frontend component will make HTTP requests to the Grafana server, which in turn forwards those calls to the backend component using gRPC.

The backend will implement the `CallCallResourceHandler` interface which has a single method named `CallResource`. This `CallResource` method will be called whenever we receives a request from the frontend. Let's take a look at the implementation of the `CallResource` method inside the **api.go** file.

{{< collapsible go>}}
// api.go
package plugin

import (
	"context"
	"encoding/json"
	"fmt"

	"github.com/grafana/grafana-plugin-sdk-go/backend"
	"github.com/grafana/grafana-plugin-sdk-go/backend/log"
)

func (d *Datasource) CallResource(ctx context.Context, req *backend.CallResourceRequest, sender backend.CallResourceResponseSender) error {
	log.DefaultLogger.Info("Received API Call from server: " + string(req.Body))
	switch req.Path {
	case "browse":
		nodeIdReq := ReadNodeRequest{}
		if err := json.Unmarshal(req.Body, &nodeIdReq); err != nil {
			log.DefaultLogger.Error("Failed to parse request body:", "error", err.Error())
			return sender.Send(&backend.CallResourceResponse{
				Status: int(backend.StatusInternal),
				Body:   []byte("{\"message\": \"Failed to parse body\"}"),
			})
		}
		nodeList, err := d.Browse(nodeIdReq.NodeId, nodeIdReq.Path)
		if err != nil {
			errMsg := fmt.Sprintf("Failed to browse node: %s", err.Error())
			log.DefaultLogger.Error(errMsg)
			return sender.Send(&backend.CallResourceResponse{
				Status: int(backend.StatusInternal),
				Body:   []byte(fmt.Sprintf("{\"message\": \"%s\"}", errMsg)),
			})
		}

		obj, err := json.Marshal(nodeList)
		if err != nil {
			log.DefaultLogger.Error("Failed to parse browse response:", "error", err.Error())
			return sender.Send(&backend.CallResourceResponse{
				Status: int(backend.StatusInternal),
				Body:   []byte("{\"message\": \"Failed to parse browse response\"}"),
			})
		} else {
			return sender.Send(&backend.CallResourceResponse{
				Status: int(backend.StatusOK),
				Body:   []byte(obj),
			})
		}
	default:
		return sender.Send(&backend.CallResourceResponse{
			Status: int(backend.StatusBadRequest),
			Body:   []byte("{\"message\": \"Bad request\"}"),
		})
	}
}
{{< /collapsible>}}
\
![](/images/backend_callresource.png "Figure 4: Frontend component wants to browse OPC-UA node, CallResource method is called on the backend component")

The `req.Path` indicates the action that the frontend wants to perform. In our case, we only handle one action: browsing OPC-UA nodes. The response will be a JSON body containing the OPC-UA nodes.

## Browsing OPC-UA nodes

Before we delve into implementing OPC-UA node browsing, let’s first explore how OPC-UA nodes are structured. Consider the following example:

```
/Root
├── Objects
│   ├── PLC 1
│   │   ├── Temperature
│   │   ├── Flow
│   └── PLC 2
│   │   ├── Temperature
│   │   ├── Flow
│   └── Building A
│   │   ├── Floor B
|   │   │   ├── PLC 3
|   |   │   │   ├── Temperature
|   |   │   │   ├── Flow
```

OPC-UA nodes are organized in a hierarchical structure each node identified by a unique node identifier. Each node can have a value that can be of different types like string, bool, numeric, list, etc. In the above example we have the following nodes:

**/Root**:
   - This node represents the root node

**Objects**:
   - The node with the identifier `i=85` in OPC-UA corresponds to the Objects folder. This folder is a fundamental part of the OPC-UA address space and typically contains other nodes representing various objects and components within the system


All other nodes are organized within the Objects node. If we want to retrieve all nodes for **Building A**, we can simply browse the node identifier for **Building A** recursively to obtain its children. As you can imagine there can be a lot of nodes and if we were browsing a very large hierarchy it might take a very long time to finish. To optimize the browsing we can reduce the amount of nodes returned to a subset containing only the direct children of the browsed node. To better clarify this let's look at an example.

Let's say we wanted to browse all nodes under the `Objects` node. This would only give us **PLC 1, PLC 2** and **Building A**. It would not give us any children of those three nodes. Now if we furthermore browsed **Building A** it would give us **Floor B** but not children of **Floor B**.

Now that we have a better understanding of how browsing works, let's take a look at the actual implementation which is found in **browse.go**

{{< collapsible go>}}
// browse.go
package plugin

import (
	"context"
	"fmt"
	"strconv"

	"github.com/gopcua/opcua"
	"github.com/gopcua/opcua/id"
	"github.com/gopcua/opcua/ua"
	"github.com/grafana/grafana-plugin-sdk-go/backend/log"
	"github.com/pkg/errors"
)

type NodeDef struct {
	NodeID      *ua.NodeID
	NodeClass   ua.NodeClass
	BrowseName  string
	Description string
	AccessLevel ua.AccessLevelType
	Path        string
	DataType    string
	Writable    bool
	Unit        string
	Scale       string
	Min         string
	Max         string
	Children    []NodeDef
	Value       interface{}
}

func (n NodeDef) Records() []string {
	return []string{n.BrowseName, n.DataType, n.NodeID.String(), n.Unit, n.Scale, n.Min, n.Max, strconv.FormatBool(n.Writable), n.Description}
}

func join(a, b string) string {
	if a == "" {
		return b
	}
	return a + "." + b
}

func (d *Datasource) browseRecursive(ctx context.Context, n *opcua.Node, path string, level int) (*NodeDef, error) {
	if level > 1 {
		return nil, nil
	}

	attrs, err := n.Attributes(ctx, ua.AttributeIDNodeClass, ua.AttributeIDBrowseName, ua.AttributeIDDescription, ua.AttributeIDAccessLevel, ua.AttributeIDDataType, ua.AttributeIDValue)
	if err != nil {
		return nil, err
	}

	var def = NodeDef{
		NodeID: n.ID,
	}

	switch err := attrs[0].Status; err {
	case ua.StatusOK:
		def.NodeClass = ua.NodeClass(attrs[0].Value.Int())
	default:
		return nil, err
	}

	switch err := attrs[1].Status; err {
	case ua.StatusOK:
		def.BrowseName = attrs[1].Value.String()
	default:
		return nil, err
	}

	switch err := attrs[2].Status; err {
	case ua.StatusOK:
		def.Description = attrs[2].Value.String()
	case ua.StatusBadAttributeIDInvalid:
		// ignore
	default:
		return nil, err
	}

	switch err := attrs[3].Status; err {
	case ua.StatusOK:
		def.AccessLevel = ua.AccessLevelType(attrs[3].Value.Int())
		def.Writable = def.AccessLevel&ua.AccessLevelTypeCurrentWrite == ua.AccessLevelTypeCurrentWrite
	case ua.StatusBadAttributeIDInvalid:
		// ignore
	default:
		return nil, err
	}

	switch err := attrs[4].Status; err {
	case ua.StatusOK:
		switch v := attrs[4].Value.NodeID().IntID(); v {
		case id.DateTime:
			def.DataType = "time.Time"
		case id.Boolean:
			def.DataType = "bool"
		case id.SByte:
			def.DataType = "int8"
		case id.Int16:
			def.DataType = "int16"
		case id.Int32:
			def.DataType = "int32"
		case id.Byte:
			def.DataType = "byte"
		case id.UInt16:
			def.DataType = "uint16"
		case id.UInt32:
			def.DataType = "uint32"
		case id.UtcTime:
			def.DataType = "time.Time"
		case id.String:
			def.DataType = "string"
		case id.Float:
			def.DataType = "float32"
		case id.Double:
			def.DataType = "float64"
		default:
			def.DataType = attrs[4].Value.NodeID().String()
		}
	case ua.StatusBadAttributeIDInvalid:
		// ignore
	default:
		return nil, err
	}

	switch err := attrs[5].Status; err {
	case ua.StatusOK:
		if attrs[5].Value != nil {
			def.Value = attrs[5].Value.Value()
		}
	case ua.StatusBadAttributeIDInvalid:
		// ignore
	default:
		return nil, err
	}

	def.Path = join(path, def.NodeID.String())
	//log.DefaultLogger.Info(fmt.Sprintf("Node: %s", def.BrowseName))

	browseChildren := func(refType uint32) error {
		refs, err := n.ReferencedNodes(ctx, refType, ua.BrowseDirectionForward, ua.NodeClassAll, true)
		if err != nil {
			return errors.Errorf("References: %d: %s", refType, err)
		}
		if len(refs) > 0 && def.Children == nil {
			def.Children = make([]NodeDef, 0)
		}

		for _, rn := range refs {
			children, err := d.browseRecursive(ctx, rn, def.Path, level+1)
			if err != nil {
				return errors.Errorf("browse children: %s", err)
			}
			if children != nil {
				if !d.isChildAlreadyExists(&def, children) {
					def.Children = append(def.Children, *children)
				} else {
					log.DefaultLogger.Info(fmt.Sprintf("Child %s already exists in %s", children.Path, def.Path))
				}
			}
		}
		return nil
	}

	if err := browseChildren(id.HasComponent); err != nil {
		log.DefaultLogger.Info("Browse components error:", "error", err.Error())
		return nil, err
	}
	if err := browseChildren(id.Organizes); err != nil {
		log.DefaultLogger.Info("Browse organizes error:", "error", err.Error())
		return nil, err
	}
	if err := browseChildren(id.HasProperty); err != nil {
		log.DefaultLogger.Info("Browse properties error:", "error", err.Error())
		return nil, err
	}
	return &def, nil
}

func (d *Datasource) Browse(nodeId string, path string) (*NodeDef, error) {
	log.DefaultLogger.Info(fmt.Sprintf("Start browsing node id: %s", nodeId))
	ctx := context.Background()

	if state := d.client.State(); state != opcua.Connected {
		log.DefaultLogger.Info("Not connected. Start a new connection....")
		if err := d.client.Connect(ctx); err != nil {
			return nil, err
		}
	}

	id, err := ua.ParseNodeID(nodeId)
	if err != nil {
		return nil, err
	}

	nodeList, err := d.browseRecursive(ctx, d.client.Node(id), path, 0)
	if err != nil {
		return nil, err
	}

	return nodeList, nil
}

func (d *Datasource) isChildAlreadyExists(parent *NodeDef, child *NodeDef) bool {
	for _, val := range parent.Children {
		if child.NodeID.String() == val.NodeID.String() {
			return true
		}
	}
	return false
}
{{< /collapsible>}}
\
The code above might look intimidating, but all it does is ensure that we can browse a node by its node identifier and only return the direct children of that node.

The `Browse` method is invoked as part of the action requested by the frontend component, and it returns a JSON containing the `NodeDef` structure to the frontend.

Figure 5 shows an example of how the frontend looks like when user requested browsing

![](/images/grafana_browse_ui.png "Figure 5: Browse OPC-UA nodes")

## Streaming data

Now, let’s explore our approach to implementing data streaming. Specifically, we aim to stream values only for the OPC-UA nodes that the user has subscribed to (as shown in Figure 6).

![](/images/grafana_subscribe_node.png "Figure 6: Subscribe to OPC-UA nodes")

When subscribing to an OPC-UA node, we aim to store its Node identifier and transmit it to the backend. This allows us to retrieve the corresponding values for the nodes.

To enable data streaming whenever the data value changes or at fixed intervals, we can leverage a feature called **Grafana Live**. This feature allows plugins to push event data directly to the frontend, eliminating the need for dashboard data refreshes. Streaming is done by creating a datasource channel that has the following format: `ds/<DATASOURCE_UID>/<CUSTOM_PATH>`.

From the backend perspective we will need to implement the `StreamHandler` interface:

{{< collapsible go>}}
// StreamHandler handles streams.
type StreamHandler interface {
	// SubscribeStream called when a user tries to subscribe to a plugin/datasource
	// managed channel path – thus plugin can check subscribe permissions and communicate
	// options with Grafana Core. As soon as first subscriber joins channel RunStream
	// will be called.
	SubscribeStream(context.Context, *SubscribeStreamRequest) (*SubscribeStreamResponse, error)
	// PublishStream called when a user tries to publish to a plugin/datasource
	// managed channel path. Here plugin can check publish permissions and
	// modify publication data if required.
	PublishStream(context.Context, *PublishStreamRequest) (*PublishStreamResponse, error)
	// RunStream will be initiated by Grafana to consume a stream. RunStream will be
	// called once for the first client successfully subscribed to a channel path.
	// When Grafana detects that there are no longer any subscribers inside a channel,
	// the call will be terminated until next active subscriber appears. Call termination
	// can happen with a delay.
	RunStream(context.Context, *RunStreamRequest, *StreamSender) error
}
{{< /collapsible>}}
\
So we need to implement the three methods `SubscribeStream`, `PublishStream` and `RunStream` in our backend. We will implement the methods inside the **streams.go** file.

{{< collapsible go>}}
// streams.go
package plugin

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"github.com/gopcua/opcua"
	"github.com/grafana/grafana-plugin-sdk-go/backend"
	"github.com/grafana/grafana-plugin-sdk-go/backend/log"
	"github.com/grafana/grafana-plugin-sdk-go/data"
)

func (d *Datasource) SubscribeStream(context.Context, *backend.SubscribeStreamRequest) (*backend.SubscribeStreamResponse, error) {
	log.DefaultLogger.Info("Subscribed to stream")
	return &backend.SubscribeStreamResponse{
		Status: backend.SubscribeStreamStatusOK,
	}, nil
}

func (d *Datasource) PublishStream(_ context.Context, req *backend.PublishStreamRequest) (*backend.PublishStreamResponse, error) {
	log.DefaultLogger.Info("Publish to stream: ", req)
	q := Query{}
	json.Unmarshal(req.Data, &q)
	log.DefaultLogger.Info("stream data: ", q)

	return &backend.PublishStreamResponse{
		Status: backend.PublishStreamStatusOK,
	}, nil
}

func (d *Datasource) RunStream(ctx context.Context, req *backend.RunStreamRequest, sender *backend.StreamSender) error {
	q := Query{}
	json.Unmarshal(req.Data, &q)

	log.DefaultLogger.Info(fmt.Sprintf("run stream: %s", req.Path))
	log.DefaultLogger.Info(fmt.Sprintf("stream data: %v", q))

	if state := d.client.State(); state != opcua.Connected {
		if err := d.client.Connect(ctx); err != nil {
			log.DefaultLogger.Error("Could connect to data source:", "error", err.Error())
		}
	}

	ticker := time.NewTicker(time.Duration(q.RefreshInterval) * time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-ticker.C:

			timeNow := time.Now().UTC()
			frame := data.NewFrame("response", data.NewField("time", nil, []time.Time{timeNow}))
			for _, qn := range q.QueryNodes {
				if d.IsReadableDataType(qn.DataType) {
					val, err := d.ReadNode(qn.NodeID)
					if err == nil && val != nil {
						frame = d.AddDataFrameField(qn.NodeID, val, frame)
					}
				}
			}

			err := sender.SendFrame(
				frame,
				data.IncludeAll,
			)

			if err != nil {
				log.DefaultLogger.Error("Failed send frame", "error", err)
			}
		}
	}
}
{{< /collapsible>}}
\
Let's break down the code.

1. **SubscribeStream:**
	- SubscribeStream is called when a user tries to subscribe to a plugin/datasource managed channel path. This method will return a status OK back to Grafana indicating that we have successfully subscribed to the stream channel.

2. **PublishStream:**
	- PublishStream called when a user tries to publish to a plugin/datasource managed channel path. We will return a status OK back to Grafana indicating that we have received any event published to the stream channel.

3. **RunStream:**
	- RunStream will be initiated by Grafana to consume a stream. This is where we implement logic to retrieve values for the subscribed OPC-UA nodes and send the result back to the frontend. The RunStream runs in it's own thread and keeps reading values at an interval that can be configured by the frontend.

The code for reading the value of an OPC-UA node and add it to the response are implemented in the **readNode.go** and **dataframe.go** file.

{{< collapsible go>}}
// readNode.go
package plugin

import (
	"context"
	"errors"
	"fmt"
	"io"
	"time"

	"github.com/gopcua/opcua"
	"github.com/gopcua/opcua/ua"
	"github.com/grafana/grafana-plugin-sdk-go/backend/log"
)

func (d *Datasource) ReadNode(nodeId string) (*ua.Variant, error) {
	ctx := context.Background()

	if state := d.client.State(); state != opcua.Connected {
		if err := d.client.Connect(ctx); err != nil {
			return nil, err
		}
	}

	id, err := ua.ParseNodeID(nodeId)
	if err != nil {
		log.DefaultLogger.Error(fmt.Sprintf("Failed to parse node: %s", err.Error()))
		return nil, err
	}

	req := &ua.ReadRequest{
		MaxAge: 2000,
		NodesToRead: []*ua.ReadValueID{
			{NodeID: id},
		},
		TimestampsToReturn: ua.TimestampsToReturnBoth,
	}

	var resp *ua.ReadResponse

	for {
		resp, err = d.client.Read(ctx, req)
		if err == nil {
			break
		}

		// Following switch contains known errors that can be retried by the user.
		// Best practice is to do it on read operations.
		switch {
		case err == io.EOF && d.client.State() != opcua.Closed:
			// has to be retried unless user closed the connection
			time.After(1 * time.Second)
			continue

		case errors.Is(err, ua.StatusBadSessionIDInvalid):
			// Session is not activated has to be retried. Session will be recreated internally.
			time.After(1 * time.Second)
			continue

		case errors.Is(err, ua.StatusBadSessionNotActivated):
			// Session is invalid has to be retried. Session will be recreated internally.
			time.After(1 * time.Second)
			continue

		case errors.Is(err, ua.StatusBadSecureChannelIDInvalid):
			// secure channel will be recreated internally.
			time.After(1 * time.Second)
			continue

		default:
			return nil, err
		}
	}

	if resp != nil && resp.Results[0].Status != ua.StatusOK {
		return nil, fmt.Errorf("status not ok: %d", resp.Results[0].Status)
	}
	return resp.Results[0].Value, nil
}
{{< /collapsible>}}
\
The provided code is straightforward: it reads the value of the node and includes retry logic to handle network glitches. In the **dataframe.go** file, there exists a single method called `AddDataFrameField`. This method is responsible for formatting the value of the read node into a dataframe that can be transmitted back to the frontend.

{{< collapsible go>}}
// dataframe.go
package plugin

import (
	"time"

	"github.com/gopcua/opcua/ua"
	"github.com/grafana/grafana-plugin-sdk-go/data"
)

func (d *Datasource) AddDataFrameField(fieldName string, val *ua.Variant, frame *data.Frame) *data.Frame {
	switch val.Type() {
	case ua.TypeIDFloat:
		if value, ok := val.Value().(float32); ok {
			frame.Fields = append(frame.Fields,
				data.NewField(fieldName, nil, []float32{value}))
		}
	case ua.TypeIDDouble:
		if value, ok := val.Value().(float64); ok {
			frame.Fields = append(frame.Fields,
				data.NewField(fieldName, nil, []float64{value}))
		}
	case ua.TypeIDBoolean:
		if value, ok := val.Value().(bool); ok {
			frame.Fields = append(frame.Fields,
				data.NewField(fieldName, nil, []bool{value}))
		}
	case ua.TypeIDInt16:
		if value, ok := val.Value().(int16); ok {
			frame.Fields = append(frame.Fields,
				data.NewField(fieldName, nil, []int16{value}))
		}
	case ua.TypeIDInt32:
		if value, ok := val.Value().(int32); ok {
			frame.Fields = append(frame.Fields,
				data.NewField(fieldName, nil, []int32{value}))
		}
	case ua.TypeIDInt64:
		if value, ok := val.Value().(int64); ok {
			frame.Fields = append(frame.Fields,
				data.NewField(fieldName, nil, []int64{value}))
		}
	case ua.TypeIDString:
		if value, ok := val.Value().(string); ok {
			frame.Fields = append(frame.Fields,
				data.NewField(fieldName, nil, []string{value}))
		}
	case ua.TypeIDUint16:
		if value, ok := val.Value().(uint16); ok {
			frame.Fields = append(frame.Fields,
				data.NewField(fieldName, nil, []uint16{value}))
		}
	case ua.TypeIDUint32:
		if value, ok := val.Value().(uint32); ok {
			frame.Fields = append(frame.Fields,
				data.NewField(fieldName, nil, []uint32{value}))
		}
	case ua.TypeIDUint64:
		if value, ok := val.Value().(uint64); ok {
			frame.Fields = append(frame.Fields,
				data.NewField(fieldName, nil, []uint64{value}))
		}
	case ua.TypeIDDateTime:
		if value, ok := val.Value().(time.Time); ok {
			frame.Fields = append(frame.Fields,
				data.NewField(fieldName, nil, []time.Time{value}))
		}
	case ua.TypeIDByte:
		if value, ok := val.Value().(uint8); ok {
			frame.Fields = append(frame.Fields,
				data.NewField(fieldName, nil, []uint8{value}))
		}
	case ua.TypeIDSByte:
		if value, ok := val.Value().(int8); ok {
			frame.Fields = append(frame.Fields,
				data.NewField(fieldName, nil, []int8{value}))
		}
	}

	return frame
}
{{< /collapsible>}}


## Alerting

How do we handle alerting? As it turns out this is a simple task. Alert rules in Grafana works almost the same way as a normal query, which means that Grafana will send a query request to the backend. Our backend need to implement the `QueryHandler` interface. This interface caontains a single method named `QueryData`.

In figure 7 you can see a how an alert rule is created and used to construct a query.

![](/images/grafana_alert.png "Figure 7: Grafana alert rule").

As you can see we will be using the same query editor for our datasource as when we are streaming data. So the user will be browsing for the OPC-UA node to subscribe to, and those nodes will be part of the query that our backend receives.

Let's take a look at the implementation of the `QueryData` method in our backend code.

{{< collapsible go>}}
// query.go
package plugin

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"github.com/grafana/grafana-plugin-sdk-go/backend"
	"github.com/grafana/grafana-plugin-sdk-go/backend/log"
	"github.com/grafana/grafana-plugin-sdk-go/data"
)

// QueryData handles multiple queries and returns multiple responses.
// req contains the queries []DataQuery (where each query contains RefID as a unique identifier).
// The QueryDataResponse contains a map of RefID to the response for each query, and each response
// contains Frames ([]*Frame).
func (d *Datasource) QueryData(ctx context.Context, req *backend.QueryDataRequest) (*backend.QueryDataResponse, error) {
	log.DefaultLogger.Info("Received query:", "data", req)
	response := backend.NewQueryDataResponse()

	// loop over queries and execute them individually.
	for _, q := range req.Queries {
		res := d.query(ctx, req.PluginContext, q)

		// save the response in a hashmap
		// based on with RefID as identifier
		response.Responses[q.RefID] = res
	}

	return response, nil
}

func (d *Datasource) query(_ context.Context, pCtx backend.PluginContext, query backend.DataQuery) backend.DataResponse {
	var response backend.DataResponse

	// Unmarshal the JSON into our queryModel.
	var qm Query

	err := json.Unmarshal(query.JSON, &qm)
	if err != nil {
		log.DefaultLogger.Error(fmt.Sprintf("Failed to parse query data: %s", err.Error()))
		return backend.ErrDataResponse(backend.StatusBadRequest, fmt.Sprintf("json unmarshal: %v", err.Error()))
	}

	log.DefaultLogger.Info("Query data:", "refId", query.RefID, "data", qm)

	timeNow := time.Now()
	frame := data.NewFrame("response", data.NewField("time", nil, []time.Time{timeNow}))
	for _, qn := range qm.QueryNodes {
		if d.IsReadableDataType(qn.DataType) {
			val, err := d.ReadNode(qn.NodeID)
			if err == nil {
				frame = d.AddDataFrameField(qn.NodeID, val, frame)
			}
		}
	}

	// add the frames to the response.
	response.Frames = append(response.Frames, frame)

	return response
}
{{< /collapsible>}}
\
In this implementation, the structure closely resembles the streaming scenario. However, instead of continuously reading values for the OPC-UA nodes and streaming them back, we only retrieve the value on demand whenever the query is triggered.

In the alerting scenario, users create alert rules and configure the queries that need to be run. Then, on a schedule, these queries are triggered, and our backend receives them, providing node values for each query.

## Conclusion

In this segment of our series, we’ve explored the backend implementation. We demonstrated how to connect to OPC-UA servers, browse OPC-UA nodes, and implemented streaming functionality. This allows data to be pushed near real-time to the frontend while also providing alerting capabilities.

This concludes Part 3 of our series. In Part 4, I'll guide you through the impementation of the frontend component.