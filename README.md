# JMS with External Provider in webMethods.io

## Scenario

This demo package showcases a complete roundtrip integration between **webMethods.io (wm.io)** and an external **AMQP messaging provider**. A JMS message is sent from a **wm.io Flow Service** to an AMQP provider queue (`MyQueue`). A **JMS Trigger** in wm.io listens to that queue, retrieves the message, and invokes a simple **logging flow** in wm.io.

While this demo uses **Solace** as the AMQP provider (via **Apache Qpid JMS**), the same pattern can be applied to other JMS providers.

![Overview Diagram](doc/overview.drawio.svg)

---

## Detailed Flow

The flow consists of the following steps:

1. A wm.io **Flow Service** invokes the `send` service from the imported **JMSDemo** package.
2. The `send` service internally calls `pub.jms:send`, using the JMS connection alias `AMQP_SOLACE` to publish a message to the Solace queue `MyQueue` via AMQP.
3. The **JMS Trigger** `jmsdemo.trigger:wmioTrigger_sample` listens to `MyQueue` and triggers the `jmsdemo.services:triggerWMIO` service.
4. `triggerWMIO` uses an HTTP client to invoke the wm.io Flow Service `jmsReceive` (exposed as a REST endpoint).
5. The `jmsReceive` service logs the incoming JMS message.

![Detail Flow](doc/detail-flow.drawio.svg)

---

## Prerequisites

To enable this scenario, the following configurations must be made using the **Runtime API** of wm.io:

![Provisioning](doc/provision.svg)

---

### 1. Create JNDI Alias

Create a JNDI alias for the AMQP connection:

```bash
curl --request POST \
  --url https://<wmio-url>/apis/v1/rest/control-plane/runtimes/default/configurations/jndi/AMQP_SOLACE \
  --header 'Authorization: Basic <base64-credentials>' \
  --header 'Content-Type: application/json' \
  --data '{
    "properties": [
      { "propertyKey": "description", "value": "Connect to Solace" },
      { "propertyKey": "initialContextFactory", "value": "org.apache.qpid.jms.jndi.JmsInitialContextFactory" },
      { "propertyKey": "providerURL", "value": "file:/opt/softwareag/IntegrationServer/packages/JMSDemo/jndi.properties" },
      { "propertyKey": "securityPrincipal", "value": "Administrator" },
      { "propertyKey": "securityCredentials", "value": "manage" }
    ]
}'
```

> **Note**: This AMQP JNDI alias uses a **file-based configuration**. The `securityPrincipal` and `securityCredentials` have no effect in this setup.

Your `jndi.properties` file (included in the package) must define the factory and queue mappings:

```properties
connectionfactory.myFactoryLookup = amqp://x.x.x.x:5672
queue.MyQueue = MyQueue
```

---

### 2. Sync JNDI Alias

Synchronize the JNDI alias with the runtime:

```bash
curl --request POST \
  --url https://<wmio-url>/apis/v1/rest/control-plane/runtimes/default/configurations/jndi/AMQP_SOLACE/sync \
  --header 'Authorization: Basic <base64-credentials>' \
  --header 'Content-Type: application/json'
```

---

### 3. Create JMS Alias

Set up the JMS connection alias using the JNDI alias:

```bash
curl --request POST \
  --url 'https://<wmio-url>/apis/v1/rest/control-plane/runtimes/default/configurations/jms/SOLACE_AMQP?updateRuntime=true' \
  --header 'Authorization: Basic <base64-credentials>' \
  --header 'Content-Type: application/json' \
  --data '{
    "properties": [
      { "propertyKey": "description", "value": "SOLACE AMQP JMS" },
      { "propertyKey": "clientID", "value": "WM_IO_CLIENT" },
      { "propertyKey": "user", "value": "<AMQP Provider User>" },
      { "propertyKey": "password", "value": "<AMQP Provider Credentials>" },
      { "propertyKey": "jndi_jndiAliasName", "value": "AMQP_SOLACE" },
      { "propertyKey": "jndi_connectionFactoryLookupName", "value": "myFactoryLookup" }
    ]
}'
```

---

## Global Variable Configuration

This demo uses **global variables** to store runtime configuration such as:

- The Flow Service URL for receiving JMS messages
- wm.io user credentials for the HTTP call from `triggerWMIO`

These global variables can be configured via the Runtime API. Required variables:

```json
{
  "assetId": "globalvariable.JMSDemo.wmio..serviceurl",
  "displayName": "wmio.serviceurl",
  "type": "globalvariable"
},
{
  "assetId": "globalvariable.JMSDemo.wmio..user",
  "displayName": "wmio.user",
  "type": "globalvariable"
},
{
  "assetId": "globalvariable.JMSDemo.wmio..password",
  "displayName": "wmio.password",
  "type": "globalvariable"
}
```

- `wmio.serviceurl`: Synchronous URL of the exposed `jmsReceive` Flow Service  
- `wmio.user`: User with permission to invoke the service  
- `wmio.password`: Password for the wm.io user  

---

## JMS Send from wm.io

Create a Flow Service named `jmsSend` in wm.io that calls the `send` service from the JMSDemo package. Provide the following inputs:

- **Destination type**: `queue`
- **JMS Alias**: `SOLACE_AMQP`
- **Destination name**: `MyQueue`
- **JMS message** content

---

## JMS Receive in wm.io

Create a Flow Service called `jmsReceive` in wm.io using the **Messaging Service specification**.

Steps:

1. Add your custom logic (e.g., debug logging).
2. Enable **HTTP invocation** under the service settings.
3. Copy the **Synchronous URL** and assign it to the `wmio.serviceurl` global variable.

---

## Sample Flow

The following diagram summarizes the overall roundtrip:

- A **wm.io Flow Service** uses `jmsSend` from the JMSDemo package to send a message via AMQP.
- The **JMS Provider** receives the message on `MyQueue`.
- A **JMS Trigger** in the demo package picks up the message and calls `triggerWMIO`.
- This service transforms the message into an HTTP call to `jmsReceive` in wm.io.

![Sample Flow](doc/sample-flow.svg)