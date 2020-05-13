# Cloud Pub/Sub triggered service

In this sample, we'll take a look at how to connect GCP Pub/Sub messages to a service with Knative Eventing. We'll roughly be following [CloudPubSubSource Example](https://github.com/google/knative-gcp/blob/master/docs/examples/cloudpubsubsource/README.md) docs page.

## Knative with GCP & PubSub Topic

We're assuming that you already went through [Install Knative with GCP](../setup/README.md) section of the setup.

## Create a Pub/Sub topic

Create a Pub/Sub topic where messages will be sent:

```bash
gcloud pubsub topics create testing
```

## CloudPubSubSource

Create a CloudPubSubSource to connect PubSub messages to Knative Eventing. The default [cloudpubsubsource.yaml](https://github.com/google/knative-gcp/blob/master/docs/examples/cloudpubsubsource/cloudpubsubsource.yaml) connects Pub/Sub messages to a service directly.

Instead, you the following [cloudpubsubsource.yaml](../eventing/pubsub/cloudpubsubsource.yaml) to connect Pub/Sub messages to a Broker, so, we can have multiple triggers to invoke multiple services on the same message.

Note that there's an alternative
[cloudpubsubsource-workload.yaml](../eventing/pubsub/cloudpubsubsource-workload.yaml).
Use this version instead if you setup workload identity on GKE.

Create the CloudPubSubSource:

```bash
kubectl apply -f cloudpubsubsource.yaml

cloudpubsubsource.events.cloud.google.com/cloudpubsubsource-test created
```

## Broker

If there's no Broker in the default namespace already, label the namespace:

```bash
kubectl label namespace default knative-eventing-injection=enabled
```

You should see a Broker in the namespace:

```bash
kubectl get broker

NAME      READY   REASON   URL                                               AGE
default   True             http://default-broker.default.svc.cluster.local   52m
```

## Consumer

For the event consumer, we can use the Event Display service in [Hello World Eventing](helloworldeventing.md) sample. Go through the steps mentioned there to create and deploy the Event Display service.

## Trigger

Connect the Event Display service to the Broker with a Trigger.

Create a [trigger.yaml](../eventing/pubsub/trigger.yaml):

Create the trigger:

```bash
kubectl apply -f trigger-event-display-pubsub.yaml

trigger.eventing.knative.dev/trigger-event-display-pubsub created
```

Check that the trigger is ready:

```bash
kubectl get trigger

NAME                           READY   REASON   BROKER    SUBSCRIBER_URI                                   AGE
trigger-event-display-pubsub   True             default   http://event-display.default.svc.cluster.local   95s
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
kubectl logs --follow <podid>
```

You should see something similar to this:

```text
Event Display received message: Hello World
```
