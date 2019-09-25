# Hello World Eventing

As of v0.5, Knative Eventing defines Broker and Trigger to receive and filter messages. This is explained in more detail on [Knative Eventing](https://www.knative.dev/docs/eventing/) page:

![Broker and Trigger](https://www.knative.dev/docs/eventing/images/broker-trigger-overview.svg)

Knative Eventing has a few different types of [event sources](https://knative.dev/docs/eventing/sources/) (Kubernetes, GitHub, GCP Pub/Sub etc.) that it can listen. In this tutorial, we will focus on listening Google Cloud related event sources such as Google Cloud Pub/Sub. 

## Install Knative Eventing

You probably installed [Knative Eventing](https://www.knative.dev/docs/eventing/) when you [installed Knative](https://www.knative.dev/docs/install/). If not, follow the Knative installation instructions and take a look at the installation section in [Knative Eventing](https://www.knative.dev/docs/eventing/) page. In the end, you should have pods running in `knative-eventing`. Double check that this is the case:

```bash
kubectl get pods -n knative-eventing
```
## Install Knative with GCP 

[Knative with GCP](https://github.com/google/knative-gcp) builds on Kubernetes to enable easy configuration and consumption of Google Cloud Platform events and services. From Knative v0.9 onwards, this is the preferred method to receive Google Cloud events into Knative. 

[Installing Knative with GCP](https://github.com/google/knative-gcp/blob/master/docs/install/README.md) page has instructions but it essentially involves pointing to `cloud-run-events.yaml`:

```bash
kubectl apply -f https://github.com/google/knative-gcp/releases/download/v0.9.0/cloud-run-events.yaml
```

Knative with GCP implements a few difference sources (Storage, Scheduler, Channel, PullSubscription, Topic). We're interested in [PullSubscription](https://github.com/google/knative-gcp/blob/master/docs/pullsubscription/README.md) to listen for Pub/Sub messages directly from GCP. 

## Create a Service Account and a Pub/Sub Topic

In order to use [PullSubscription](https://github.com/google/knative-gcp/blob/master/docs/pullsubscription/README.md), we need a Pub/Sub enabled Service Account and instructions on how to set that up on Google Cloud is [here](https://github.com/google/knative-gcp/tree/master/docs/pubsub). 

Once you have it setup, you should have a `google-cloud-key` secret in Kubernetes:

```bash
kubectl get secret

NAME                  TYPE                                  DATA   AGE
google-cloud-key      Opaque                                1      20h
```
You should also create a Pub/Sub Topic to send messages too:

```bash
export TOPIC_NAME=testing
gcloud pubsub topics create $TOPIC_NAME
```
We're finally ready to receive Pub/Sub messages into Knative! 

## Create an Event Display

Follow the instructions for your preferred language to create a service to log out messages:

* [Create Event Display - C#](08-helloworldeventing-csharp.md)

* [Create Event Display - Python](08-helloworldeventing-python.md)

## Build and push Docker image

Build and push the Docker image (replace `{username}` with your actual DockerHub):

```bash
docker build -t {username}/event-display:v1 .

docker push {username}/event-display:v1
```

## Create Event Display

Create a [service.yaml](../eventing/event-display/service.yaml) file:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: event-display
spec:
  selector:
    matchLabels:
      app: event-display
  template:
    metadata:
      labels:
        app: event-display
    spec:
      containers:
      - name: user-container
        # Replace {username} with your actual DockerHub
        image: docker.io/{username}/event-display:v1
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: event-display
spec:
  selector:
    app: event-display
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

This defines a Kubernetes Deployment and Service to receive messages. 

Create the Event Display service:

```bash
kubectl apply -f service.yaml

deployment.apps/event-display created
service/event-display created
```

## Create PullSubscription

Last but not least, we need connect Event Display service to Pub/Sub messages with a PullSubscription. 

Create a [pullsubscription.yaml](../eventing/event-display/pullsubscription.yaml):

```yaml
apiVersion: pubsub.cloud.run/v1alpha1
kind: PullSubscription
metadata:
  name: testing-source-event-display
spec:
  topic: testing
  sink:
    apiVersion: v1
    kind: Service
    name: event-display
```
This connects the `testing` topic to `event-display` Service. 

Create the PullSubscription:

```bash
kubectl apply -f pullsubscription.yaml

pullsubscription.pubsub.cloud.run/testing-source-event-display created
```

## Test the service

We can now test our service by sending a message to Pub/Sub topic:

```bash
gcloud pubsub topics publish testing --message="Hello World"

messageIds:
- '198012587785403'
```

Wait a little and check that a pod is created:

```bash
kubectl get pods
```

You can inspect the logs of the pod (replace `<podid>` with actual pod id):

```bash
kubectl logs --follow -c user-container <podid>
```

You should see something similar to this:

```text
Event Display received message: Hello World
```

## What's Next?

[Integrate with Translation API](09-translationeventing.md)
