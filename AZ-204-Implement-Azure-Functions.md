# Implement Azure Functions

# Commands


# Notes

1. Code
2. Configuration
    - `functions.json` file
    - generated automatically from attributes in compiled languages
    - manually created for scriptiong languages
    - defines **trigger** and **bindings**
        - one and only one trigger
        - trigger is what starts execution of code
        - bindings - ways to simplify input/output
        
Great solution for processing data, integrating systems, IoT, APIs, microservices
- image processing
- file maintenance
- tasks run on a schedule
        
Azure Functions is serverless compute, where Azure Logic Apps are serverless workflows
- Serverless means no need to worry about infrastructure
- both can create complex orchestrations
    - Azure Functions: durable functions
    - Azure Logic Apps: GUI

Azure functions is built on top of the WebJobs SDK

Requires a general Azure Storage account

## Azure Functions vs Azure Logic Apps
|  | Azure Functions | Azure Logic Apps |
| :--- | :--- | :--- |
| Development | Code First | Design First |
| Connectivity | ~12 built in binding types.  Custom bindings can be coded | Large collection of connectors, Enterprise Integration Pack for B2B scenarios, build custom connectors |
| Actions | Each action is an Azure function | Large collection of ready-made actions |
| Monitoring | Azure Application Insights | Azure portal, Azure Monitor logs |
| Management | REST API, Visual Studio | Azure portal, REST API, PowerShell, Visual Studio |
| Execution Context |  |  |

## Azure Functions vs WebJobs with WebJobs SDK
|  | Azure Functions | WebJobs with WebJobs SDK |
| :--- | :--- | :--- |
| Serverless app model with automatic scaling | Yes | No |
| Develop and test in browser | Yes | No |
| Pay-per-use pricing | Yes | No |
| Integration with Logic Apps | Yes | No |
| Trigger events | Timer<br>Azure storage queues and blobs<br>Azure Service Bus queues and topics<br>Azure Cosmos DB<br>Azure Event Hubs<br>HTTP/WebHook(GitHub/Slack)<br>Azure Event Grid | Timer<br>Azure storage queues and blobs<br>Azure Service Bus queues and topics<br>Azure Cosmos DB<br>Azure Event Hubs<br>File system |

## Hosting
Plans available for both Linux and Windows VMs

Hosting plan decides
- How function app is scaled
- resources available to each function app instance
- support for advanced functionality, sucha as Azure Virtual Network


- Consumption
    - default
    - scales automatically
    - pay per use
    - 1.5 GB memory and 1 CPU per instance
- Functions Premium
    - auto scales
    - pre-warmed workers
    - more powerful compute instances
    - can connect to virtual networks
- App Service (dedicated)
    - as per usually
    - best for long run scenarios where durable functions can't be used
    - always enable the `Always On` setting (only HTTP triggers will wake up functions otherwise)

### Scaling
Unit of scale is the Function App (not per function)
- it is possible to scale to 0
- will require a cold start (latency of scaling from 0 to 1)

- max 200 instances for Consumption.  100 for Premium.  (but you can limit using `functionAppScaleLimit`)
- max 1 instance per second for http triggers
- max 1 instance per 30 seconds for all other triggers
    
### Other
- App Service Environment - app service feature that provides fully isolated and dedicated
- Kubernetes - provides fully isolated and dedicated on top of Kubernetes platform

Available both on Linux and Windows

## Trigger
one and only one per function
    - type (e.g., queueTrigger)
    - direction (in/out)
    - name
    
What causes the function to run

Binding direction is always in

## Binding
Declarative way of connecting funciton to other resources

Possible to have multiple input and output bindings
- binding direction is `in` or `out` (or `inout`)

Can have associate data (payload)

## Defining Triggers and Bindings

C# - use attributes
Java - use annotations
JavaScript/PowerShell/Python/TypeScript - `function.json`

In .NET and Java, the parameter type defines the data type for input data

For languages that are dynamically typed such as JavaScript, use the dataType property in the function.json file
- binary
- stream
- string

## function.json

```
{
  "bindings": [
    {
      "type": "queueTrigger",
      "direction": "in",
      "name": "order",
      "queueName": "myqueue-items",
      "connection": "MY_STORAGE_ACCT_APP_SETTING"
    },
    {
      "type": "table",
      "direction": "out",
      "name": "$return",
      "tableName": "outTable",
      "connection": "MY_TABLE_STORAGE_ACCT_APP_SETTING"
    }
  ]
}
```

Note: connections can be set with environment variables

You can treat environment variables as collections by setting connection value to a shared prefix that ends in double underscores `__`

Then it will look up collection values as `{prefix}__serviceUri` for example

Can also used identity-based connection (using managed identity or user assigned (set credential and clientID properties))

Remember to grant permissions to resources being accessed through identity (RBAC or IAM (access control))
    
## Function App
Execution context in which functions are run
Contains one or more functions
All functions are deployed and scaled together
All functions must be authored in the same language (`2.x`)

Function App defined by a host configuration file `host.json` file.
    - in root folder of function app
    - `bin` folder contains packages and library files
    
Possible to do **local** OR **portal** development.  Cannot mix.

Can dubug locally!

## Durable Functions
Stateful workflows 
- write orchestrator functions

Stateful Entities
- write entity functions

System handles state, checkppoints, restarts

Used for simplifying complex, stateful coordination requirements in serverless applications
- function chaining
    - `F1->F2->F3->F4`
- fan-out/fan-in
    - execute multiple in parallel, and wait for all to finish
- Async HTTP APIs
    - long running tasks with get status
- Monitor
    - polling
- Human interaction
    - workflow approval
    
### Function Types
    - Orchestrator
        - how actions are executed and the order
    - Activity Functions
        - like a task
    - Entity function
        - read and update state information
    - Client Functions
        - orchestrator and entity functions cannot be triggered directly
        - use client to start
        
#### Durable Orchestrations
- define function workflows with procedural code
- can call sync or async
- automatiically checkpointed when function `awaits` or `yields`.  Local state is never lost
- can be long running (no limit)

Each orchestration is given a unique `instance ID` in task hub.

- Sub-orchestrations: An orchestration can call other orchestrations or other activity tasks.
- durable timers: to implement delays
    - call `CreateTimer` of trigger binding
- external events: can wait for (e.g., human interaction)
    - use `WaitForExternalEvent` of trigger binding to wait for event
    - use `RaiseEventAsync` of trigger binding to create event
- error handling: `try/catch`
- critical sections: no need to worry about race condidions within an orchestration, but external systems can cause.  Use `LockAsync`
- HTTP endpoints: No I/O permitted in orchestration.  Wrap in activity function
- multiple parameters: need to be an array of objects
        

### Languages
- C#
- JavaScript
    - 2.x Azure Functions runtime
    - 1.7.0 Durable Functions extension
- Python
    - 2.3.1 Azure Functions runtime
- F#
    - 1.x Azure Functions runtime
- PowerShell
    - 3.x Azure Functions runtime
    - 2.x Durable Functions extension

## Task Hub
Logical container for resources used by orchestrations and entities.  Only functions that share the same task hub can interact.

A storage account can contain many (uniquely named) task hubs

Task hub consists of
- one or more control queues
- one work-item queue
- one history table
- one instances table
- one storage container containing one or more lease blobs
- one storage container containing large message payloads (if applicable)

### Task hub name
- only alphanumeric characters
- starts with a letter
- min 3, max 45 characters
- defined in host.json file