---
title: Write your Keptn-service
description:  Explains to you how to implement your Keptn-service that listens to Keptn events and extends your Keptn with certain functionality.
weight: 90
keywords: [service, custom]
aliases:
---

Explains to you how to implement your Keptn-service that listens to Keptn events and extends your Keptn with certain functionality.

## About this tutorial

The goal of this tutorial is to describe how you can add additional functionality to your Keptn installation with a custom [*Keptn-service*](#keptn-service) or [*SLI-provider*](#sli-provider). While a *Keptn-service* enriches a continuous delivery or operational workflow with additional functionality or with an extra tool, a *SLI-provider* is used to query Service Level Indicators (SLI) from an external source like a monitoring or testing solution.  

## Template Repository

We provide a fully functioning template for writing new services: [keptn-service-template-go](https://github.com/keptn-sandbox/keptn-service-template-go).

## Keptn-service

A *Keptn-service* is intended to react to certain events that occur during a continuous delivery or operational workflow. After getting triggered by an event, a *Keptn-service* processes some functionality and can therefore integrate additional tools by accessing their REST interfaces.

A Keptn-service can subscribe to various [Keptn CloudEvents](https://github.com/keptn/spec/blob/0.1.3/cloudevents.md), e.g.:

- sh.keptn.events.configuration-changed
- sh.keptn.events.deployment-finished
- sh.keptn.events.tests-finished
- sh.keptn.events.evaluation-done
- sh.keptn.events.problem

### Implement custom Keptn-service

If you are interested in writing your own testing service, have a look at the [jmeter-service](https://github.com/keptn/keptn/blob/0.6.2/jmeter-service).

### Example: JMeter Service

**Incoming Keptn CloudEvent:** The *jmeter-service* is a *Go* application that accepts POST requests at its `/` endpoint. To be more specific, the request body needs to follow the [CloudEvent specification](https://github.com/keptn/spec/blob/0.1.3/cloudevents.md) and the HTTP header attribute `Content-Type` has to be set to `application/cloudevents+json`. Of course, you can write your service in any language, as long as it provides the endpoint to receive events.

**Functionality:** The functionality of your *Keptn-service* depends on the capability you want to add to the continuous delivery or operational Keptn workflow. In many cases, the event payload -- containing meta-data such as the project, stage, or service name as well as shipyard information -- is first processed and then used to call the REST API of another tool.  

**Outgoing Keptn CloudEvent:** After your *Keptn-service* has completed its functionality, it has to send a  CloudEvent to Keptn's event broker. This informs Keptn to continue a particular workflow.  

**Deployment and service template:** A *Keptn-service* is a regular Kubernetes service with a deployment and service template. As a starting point for your service the deployment and service manifest of the *jmeter-service* can be used, which can be found in the [deploy/service.yaml](https://github.com/keptn/keptn/blob/0.6.2/jmeter-service/deploy/service.yaml):

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jmeter-service
  namespace: keptn
spec:
  selector:
    matchLabels:
      run: jmeter-service
  replicas: 1
  template:
    metadata:
      labels:
        run: jmeter-service
    spec:
      containers:
      - name: jmeter-service
        image: keptn/jmeter-service:0.6.2
        ports:
        - containerPort: 8080
        env:
        - name: EVENTBROKER
          value: 'http://event-broker.keptn.svc.cluster.local/keptn'
---
apiVersion: v1
kind: Service
metadata:
  name: jmeter-service
  namespace: keptn
  labels:
    run: jmeter-service
spec:
  ports:
  - port: 8080
    protocol: TCP
  selector:
    run: jmeter-service
```

### Subscribe service to Keptn event

**Distributor:** To subscribe your service to a Keptn event, a distributor is required. A distributor comes with a deployment manifest as shown by the example below:

```yaml
## jmeter-service: sh.keptn.events.deployment-finished
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jmeter-service-deployment-distributor
  namespace: keptn
spec:
  selector:
    matchLabels:
      run: distributor
  replicas: 1
  template:
    metadata:
      labels:
        run: distributor
    spec:
      containers:
      - name: distributor
        image: keptn/distributor:0.6.2
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        env:
        - name: PUBSUB_URL
          value: 'nats://keptn-nats-cluster'
        - name: PUBSUB_TOPIC
          value: 'sh.keptn.events.deployment-finished'
        - name: PUBSUB_RECIPIENT
          value: 'jmeter-service'
```

To configure this distributor for your *Keptn-service*, two environment variables need to be adapted: 

* `PUBSUB_RECIPIENT`: Defines the service name as specified in the Kubernetes service manifest.
* `PUBSUB_TOPIC`: Defines the event type your *Keptn-service* is listening to. 

## SLI-provider

An *SLI-provider* is an implementation of a *Keptn-service* with a dedicated purpose. This type of service is responsible to query an external data source for SLIs that are then used by Keptn to evaluate an SLO. To configure a query for an indicator, Keptn provides the concept of an [SLI configuration](https://github.com/keptn/spec/blob/0.1.3/sre.md#service-level-indicators-sli-configuration).

* Create a SLI configuration defining tool-specific queries for indicators. An example of an SLI configuration looks as follows:

```yaml
spec_version: '1.0'
indicators:
 throughput: "builtin:service.requestCount.total:merge(0):count?scope=tag(keptn_project:$PROJECT),tag(keptn_stage:$STAGE),tag(keptn_service:$SERVICE),tag(keptn_deployment:$DEPLOYMENT)"
 error_rate: "builtin:service.errors.total.count:merge(0):avg?scope=tag(keptn_project:$PROJECT),tag(keptn_stage:$STAGE),tag(keptn_service:$SERVICE),tag(keptn_deployment:$DEPLOYMENT)"
```

**Note:** This SLI configuration file will then be stored in Keptn's configuration store using the [keptn add-resource](../../reference/cli/commands/keptn_add-resource) command.


The [Keptn CloudEvents](#cloudevents) a SLI-provider has to subscribe to is:

- sh.keptn.internal.event.get-sli

### Implement custom SLI-provider

**Incoming Keptn CloudEvent:** An *SLI-provider* listens to one specific Keptn CloudEvent type, which is the [sh.keptn.internal.event.get-sli](https://github.com/keptn/spec/blob/0.1.2/cloudevents.md#get-sli) event. Next to event meta-data such as project, stage or service name, this type of event contains information about the indicators, time frame, and labels to query. For more details, please see the specification [here](https://github.com/keptn/spec/blob/0.1.3/cloudevents.md#get-sli). 

**Functionality:** The functionality of an *SLI-provider* focuses on querying indicator values from an external data source. Before a query can be fired towards the data source, the following steps are necessary:

1. Process the incoming event to get the project, stage, and service name. Besides, you will need the indicators and time frame to query.  

1. Get the SLI configuration from Keptn's configuration-service. This SLI configuration is identified by the `resourceURI`, which follows the pattern: `[tool-name]/sli.yaml` (e.g., `dynatrace/sli.yaml`). 
  * Service URL: http://configuration-service.keptn.svc.cluster.local:8080
  * Endpoint: `v1/project/{projectName}/stage/{stageName}/service/{serviceName}/resource/{resourceURI}`

1. Process the SLI configuration and use the defined queries to retrieve the values of each indicator. 

**Outgoing Keptn CloudEvent:** An *SLI-provider* returns one specific Keptn CloudEvent type, which is the [sh.keptn.internal.event.get-sli.done](https://github.com/keptn/spec/blob/0.1.3/cloudevents.md#get-sli-done) event. This event contains the retrieved value of each queried indicator.  

**Deployment and service template:** Like any custom *Keptn-service*, an SLI-provider is a regular Kubernetes service with a deployment and service template. See above how to define those templates for your SLI-provider. 

### Subscribe SLI-provider to Keptn event

**Distributor:** To subscribe your SLI-provider to the `sh.keptn.internal.event.get-sli` event, a distributor is required. A distributor comes with a deployment manifest as shown by the example below:

```yaml
## jmeter-service: sh.keptn.events.deployment-finished
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dynatrace-sli-provider-distributor
  namespace: keptn
spec:
  selector:
    matchLabels:
      run: distributor
  replicas: 1
  template:
    metadata:
      labels:
        run: distributor
    spec:
      containers:
      - name: distributor
        image: keptn/distributor:0.6.2
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        env:
        - name: PUBSUB_URL
          value: 'nats://keptn-nats-cluster'
        - name: PUBSUB_TOPIC
          value: 'sh.keptn.internal.event.get-sli'
        - name: PUBSUB_RECIPIENT
          value: 'dynatrace-sli-provider'
```

To configure this distributor for your *SLI-provider*, the environment variables `PUBSUB_RECIPIENT` has to refer to the service name of the SLI-provider in the Kubernetes service manifest. Besides, make sure the environment variable `PUBSUB_TOPIC` has the value `sh.keptn.internal.event.get-sli`.

## Deploy Keptn-service and distributor

With a service and deployment manifest for your custom *Keptn-service* (`service.yaml`) and a deployment manifest for the distributor (`distributor.yaml`), you are ready to deploy both components in the K8s cluster where Keptn is installed: 

```console
kubectl apply -f service.yaml
```

```console
kubectl apply -f distributor.yaml
```

## Summary
You will need to provide the following when you want to write a custom service:

- Your *Keptn-service* implementation including a Docker container. We recommend writing the service in *Go*
- The application needs to provide a REST endpoint at `/` that accepts `POST` requests for JSON objects.
- A `service.yaml` file containing the templates for the service and deployment manifest of your *Keptn-service*.
- A `distributor.yaml` file containing the template for the distributor and properly configured for your *Keptn-service*.


## CloudEvents

Please note that CloudEvents have to be sent with the HTTP header `Content-Type: application/cloudevents+json` to be set.
For a detailed look into CloudEvents, please go the Keptn [CloudEvent specification](https://github.com/keptn/spec/blob/0.1.3/cloudevents.md). 

## Logging

To inspect your service's log messages for a specific deployment run, you can use the `shkeptncontext` property of the incoming CloudEvents. Your service has to output its log messages in the following format:

```json
{
  "keptnContext": "<KEPTN_CONTEXT>",
  "logLevel": "INFO | DEBUG | WARNING | ERROR",
  "keptnService": "<YOUR_SERVICE_NAME>",
  "message": "logging message"
}
```

**Note:** For implementing logging into your *Go* service, you can import the [go-utils](https://github.com/keptn/go-utils) package that already provides common logging functions. 
