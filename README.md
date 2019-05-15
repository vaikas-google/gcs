# Knative`Google Cloud Storage Source` CRD.

## Overview

This repository implements an Event Source for
[Knative Eventing](http://github.com/knative/eventing) defined with a
CustomResourceDefinition (CRD). This Event Source represents
[Google Cloud Storage](https://cloud.google.com/storage/). Point is to
demonstrate an Event Source that does not live in the
[Knative Eventing Sources](http://github.com/knative/eventing-sources) that can
be independently maintained, deployed and so forth.

This particular example demonstrates how to perform basic operations such as:

- Create a Cloud Storage Notification when a Google Cloud Storage object is
  created
- Delete a Notification when that Source is deleted
- Update a Notification when that Source spec changes

## Details

Actual implementation contacts the Cloud Storage API and creates a Notification
as specified in the GCSSource CRD Spec. Upon success a Knative service is
created to receive calls from the Cloud Storage via GCP Pub Sub and will then
forward them to the Channel or a Knative Service.

## Purpose

Provide an Event Source that allows subscribing to Cloud Storage Object
Notifications and processing them in Knative.

Another purpose is to serve as an example of how to build an Event Source using
a [Warm Image[(https://github.com/mattmoor/warm-image) as a starting point.

## Prerequisites

1. Create a
   [Google Cloud project](https://cloud.google.com/resource-manager/docs/creating-managing-projects)
   and install the `gcloud` CLI and run `gcloud auth login`. This sample will
   use a mix of `gcloud` and `kubectl` commands. The rest of the sample assumes
   that you've set the `$PROJECT_ID` environment variable to your Google Cloud
   project id, and also set your project ID as default using
   `gcloud config set project $PROJECT_ID`.

1. Setup [Knative Serving](https://github.com/knative/docs/blob/master/install)

1. Configure
   [outbound network access](https://github.com/knative/docs/blob/master/serving/outbound-network-access.md)

1. Setup
   [Knative Eventing](https://github.com/knative/docs/tree/master/eventing)
   using the `release.yaml` file. This example does not require GCP.

1. Setup
   [GCP PubSub Source](https://github.com/knative/eventing-sources/tree/master/contrib/gcppubsub/samples)
   Just need to do Prerequisites, no need to deploy anything unless you just
   want to make sure that everything is up and running correctly.

1. Have an existing bucket in GCS (or create a new one) that you have
   permissions to manage. Let's set up an environmental variable for that which
   we'll use in the rest of this document.

   ```shell
   export MY_GCS_BUCKET=<YOUR_BUCKET_NAME>
   ```

## Create a GCP Service Account and a corresponding secret in Kubernetes

1. Create a
   [GCP Service Account](https://console.cloud.google.com/iam-admin/serviceaccounts/project).
   This sample creates one service account for both registration and receiving
   messages, but you can also create a separate service account for receiving
   messages if you want additional privilege separation.

   1. Create a new service account named `gcs-source` with the following
      command:
      ```shell
      gcloud iam service-accounts create gcs-source
      ```
   1. Give that Service Account the Admin role for storage your GCP project:
      ```shell
      gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member=serviceAccount:gcs-source@$PROJECT_ID.iam.gserviceaccount.com \
        --role roles/storage.admin
      ```
   1. Give that Service Account the Editor role for pubsub your GCP project:

      ```shell
      gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member=serviceAccount:gcs-source@$PROJECT_ID.iam.gserviceaccount.com \
        --role roles/pubsub.editor
      ```

   1. Create a namespace for where the secret is created and where our
      controller will run

      ```shell
      kubectl create namespace gcssource-system
      ```

   1. Give Google Cloud Storage permissions to publish to GCP Pub Sub.

      1. First find the Service Account that GCS uses to publish to Pub Sub
         (Either using UI, or using curl as shown below)

         1. Use the
            [Cloud Console or the JSON API](https://cloud.google.com/storage/docs/getting-service-account)
            Assume the service account you found from above was
            `service-XYZ@gs-project-accounts.iam.gserviceaccount.com`, you'd do:
            `shell export GCS_SERVICE_ACCOUNT=service-XYZ@gs-project-accounts.iam.gserviceaccount.com`

         1. Use `curl` to fetch the email:

         ```shell
         export GCS_SERVICE_ACCOUNT=`curl -s -X GET -H "Authorization: Bearer \`GOOGLE_APPLICATION_CREDENTIALS=./gcs-source.json gcloud auth application-default print-access-token\`" "https://www.googleapis.com/storage/v1/projects/$PROJECT_ID/serviceAccount" | grep email_address | cut -d '"' -f 4`
         ```

      1. Then grant rights to that Service Account to publish to GCP PubSub.

         ```shell
         gcloud projects add-iam-policy-binding $PROJECT_ID \
           --member=serviceAccount:$GCS_SERVICE_ACCOUNT \
           --role roles/pubsub.publisher
         ```

   1. Download a new JSON private key for that Service Account. **Be sure not to
      check this key into source control!**

      ```shell
      gcloud iam service-accounts keys create gcs-source.json \
       --iam-account=gcs-source@$PROJECT_ID.iam.gserviceaccount.com
      ```

   1. Create two secrets on the kubernetes cluster with the downloaded key:

      ```shell
      # Secret for gcssource-system:
      kubectl --namespace gcssource-system create secret generic gcs-source-key --from-file=key.json=gcs-source.json

      # Secret for default:
      kubectl --namespace default create secret generic google-cloud-key --from-file=key.json=gcs-source.json
      ```

      `gcs-source-key` and `key.json` are pre-configured values in the
      `gcssource-controller` Deployment which manages your GCS sources.

      `google-cloud-key` and `key.json` are pre-configured values in
      [`one-to-one-gcs.yaml`](./one-to-one-gcs.yaml).

## Install Cloud Storage Source

```shell
kubectl apply -f https://raw.githubusercontent.com/vaikas-google/gcs/master/release.yaml
```

## Inspect the Cloud Storage Source

First list the available sources, you might have others available to you, but
this is the one we'll be using in this example

```shell
 kubectl get crds -l "eventing.knative.dev/source=true"
```

You should see something like this:

```shell
NAME                                      AGE
gcssources.sources.aikas.org              13d
```

you can then get more details about it, for example what are the available
configuration options for it:

```shell
kubectl get crds gcssources.sources.aikas.org -oyaml
```

And in particular the Spec section is of interest, because it shows
configuration parameters and describes them as well as what the required
parameters are:

```shell
  validation:
    openAPIV3Schema:
      properties:
        apiVersion:
          type: string
        kind:
          type: string
        metadata:
          type: object
        spec:
          properties:
            bucket:
              description: GCS bucket to subscribe to. For example my-test-bucket
              type: string
            gcpCredsSecret:
              description: Optional credential to use for subscribing to the GCP PubSub
                topic. If omitted, uses gcsCredsSecret. Must be a service account
                key in JSON format (see https://cloud.google.com/iam/docs/creating-managing-service-account-keys).
              type: object
            gcsCredsSecret:
              description: Credential to use for creating a GCP notification. Must
                be a service account key in JSON format (see https://cloud.google.com/iam/docs/creating-managing-service-account-keys).
              type: object
            googleCloudProject:
              description: Google Cloud Project ID to create the scheduler job in.
              type: string
            objectNamePrefix:
              description: Optional prefix to only notify when objects match this
                prefix.
              type: string
            payloadFormat:
              description: Optional payload format. Either NONE or JSON_API_V1. If
                omitted, uses JSON_API_V1.
              type: string
            serviceAccountName:
              description: Service Account to run Receive Adapter as. If omitted,
                uses 'default'.
              type: string
            sink:
              description: Where to sink the notificaitons to.
              type: object
          required:
          - gcsCredsSecret
          - googleCloudProject
          - bucket
          - sink
```

## Create a Knative Service that will be invoked for each Cloud Storage object notification

To verify the `Cloud Storage` is working, we will create a simple Knative
`Service` that dumps incoming messages to its log. The `service.yaml` file
defines this basic service. Image might be different if a new version has been
released.

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: gcs-message-dumper
spec:
  runLatest:
    configuration:
      revisionTemplate:
        spec:
          container:
            image: us.gcr.io/probable-summer-223122/eventdumper-833f921e52f6ce76eb11f89bbfcea1df@sha256:7edb9fc190dcf350f4c49c48d3ff2bf71de836ff3dc32b1d5082fd13f90edee3
```

Enter the following command to create the service from `service.yaml`:

```shell
kubectl --namespace default apply -f https://raw.githubusercontent.com/vaikas-google/gcs/master/service.yaml
```

## Configure Cloud Storage to send events directly to the service

The simplest way to consume events is to wire the Source directly into the
consuming function. The logical picture looks like this:

![Source Directly To Function](csr-1-1.png)

## Wire Cloud Storage events to the function

Create a Cloud Storage instance targeting your function with the following:

```shell
curl https://raw.githubusercontent.com/vaikas-google/gcs/master/one-to-one-gcs.yaml | \
sed "s/MY_GCP_PROJECT/$PROJECT_ID/g" | sed "s/MY_GCS_BUCKET/$MY_GCS_BUCKET/g" | kubectl apply -f -
```

## Check that the Cloud Storage Source was created

```shell
kubectl get gcssources
```

And you should see something like this:

```shell
vaikas@penguin:~/projects/go/src/github.com/vaikas-google/gcs$ kubectl get gcssources
NAME                AGE
notification-test   8d
```

And inspecting the Status field of it via:

```shell
vaikas@penguin:~/projects/go/src/github.com/vaikas-google/gcs$ kubectl get gcssources -oyaml
apiVersion: v1
items:
- apiVersion: sources.aikas.org/v1alpha1
  kind: GCSSource
  metadata:
    creationTimestamp: 2018-12-27T03:43:26Z
    finalizers:
    - gcs-controller
    generation: 1
    name: notification-test
    namespace: default
    resourceVersion: "12485601"
    selfLink: /apis/sources.aikas.org/v1alpha1/namespaces/default/gcssources/notification-test
    uid: 9176c51c-0989-11e9-8605-42010a8a0205
  spec:
    bucket: vaikas-knative-test-bucket
    gcsCredsSecret:
      key: key.json
      name: google-cloud-key
    googleCloudProject: quantum-reducer-434
    sink:
      apiVersion: serving.knative.dev/v1alpha1
      kind: Service
      name: message-dumper
  status:
    conditions:
    - lastTransitionTime: 2019-01-09T01:46:54Z
      severity: Error
      status: "True"
      type: GCSReady
    - lastTransitionTime: 2019-01-09T01:46:54Z
      severity: Error
      status: "True"
      type: PubSubSourceReady
    - lastTransitionTime: 2019-01-09T01:46:53Z
      severity: Error
      status: "True"
      type: PubSubTopicReady
    - lastTransitionTime: 2019-01-09T01:46:54Z
      severity: Error
      status: "True"
      type: Ready
    notificationID: "5"
    sinkUri: http://message-dumper.default.svc.cluster.local/
    topic: gcs-67b38ee6-64a4-4867-892d-33ee53ff24d4
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

We can see that the Conditions 'type: Ready" is set to 'status: "True"'
indicating that the notification was correctly created. We can see that the
notificationID has been filled in indicating the GCS notification was created.
sinkUri is the Knative Service where the events are being delivered to. topic
idenfities the GCP Pub Sub topic that we are using as a transport mechanism
between GCS and your function.

## (Optional) Check that the Cloud Storage notification was created with gsutil

```shell
vaikas@vaikas:~/projects/go/src/github.com$ gsutil notification list gs://$MY_GCS_BUCKET
projects/_/buckets/vaikas-knative-test-bucket/notificationConfigs/5
	Cloud Pub/Sub topic: projects/quantum-reducer-434/topics/gcs-67b38ee6-64a4-4867-892d-33ee53ff24d4
```

Then [upload some file](https://cloud.google.com/storage/docs/uploading-objects)
to your bucket to trigger an Object Notification.

## Check that Cloud Storage invoked the function

```shell
kubectl -l 'serving.knative.dev/service=gcs-message-dumper' logs -c user-container
```

And you should see an entry like this there

```shell
2019/01/09 17:56:01 Received Cloud Event Context as: {CloudEventsVersion:0.1 EventID:303284831868154 EventTime:2019-01-09 17:56:00.16 +0000 UTC EventType:google.pubsub.topic.publish EventTypeVersion: SchemaURL: ContentType:application/json Source://pubsub.googleapis.com/quantum-reducer-434/topics/gcs-e29de50b-e416-44fd-9c28-2ea33764096a Extensions:map[]}
2019/01/09 17:56:01 object notification metadata is: &{Name:dummytextfile Bucket:vaikas-knative-test-bucket Size:37}
```

Where the headers displayed are the Cloud Events Context and last few lines are
the actual Notification Details.

## Uninstall

```shell
kubectl delete gcssources notification-test
kubectl delete services.serving gcs-message-dumper
gcloud iam service-accounts delete gcs-source@$PROJECT_ID.iam.gserviceaccount.com
gcloud projects remove-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:gcs-source@$PROJECT_ID.iam.gserviceaccount.com \
  --role roles/storage.admin

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:gcs-source@$PROJECT_ID.iam.gserviceaccount.com \
  --role roles/pubsub.editor

gcloud projects remove-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$GCS_SERVICE_ACCOUNT \
  --role roles/pubsub.publisher
kubectl delete secrets gcs-source-key
kubectl delete services.serving gcs-message-dumper
```

# **REST OF THIS DOCUMENT NEEDS UPDATING**

## More complex examples

- [Multiple functions working together](MULTIPLE_FUNCTIONS.md)

## Usage

### Specification

The specification for a scheduler job looks like:

```yaml
apiVersion: sources.aikas.org/v1alpha1
kind: CloudSchedulerSource
metadata:
  name: scheduler-test
spec:
  googleCloudProject: quantum-reducer-434
  location: us-central1
  schedule: "every 1 mins"
  body: "{test does this work}"
  sink:
    apiVersion: eventing.knative.dev/v1alpha1
    kind: Channel
    name: scheduler-demo
```

### Creation

With the above in `foo.yaml`, you would create the Cloud Scheduler Job with:

```shell
kubectl create -f foo.yaml
```

### Listing

You can see what Cloud Scheduler Jobs have been created:

```shell
$ kubectl get cloudschedulersources
NAME             AGE
scheduler-test   4m
```

### Updating

You can upgrade `foo.yaml` jobs by updating the spec. For example, say you
wanted to change the above job to send a different body, you'd update the
foo.yaml from above like so:

```yaml
apiVersion: sources.aikas.org/v1alpha1
kind: CloudSchedulerSource
metadata:
  name: scheduler-test
spec:
  googleCloudProject: quantum-reducer-434
  location: us-central1
  schedule: "every 1 mins"
  body: "{test does this work, hopefully this does too}"
  sink:
    apiVersion: eventing.knative.dev/v1alpha1
    kind: Channel
    name: scheduler-demo
```

And then update the spec.

```shell
kubectl replace -f foo.yaml
```

Of course you can also do this in place by using:

```shell
kubectl edit cloudschedulersources scheduler-test
```

And on the next run (or so) the body send to your function will by changed to
'{test does this work, hopefully this does too}' instead of '{test does this
work}' like before.

### Removing

You can remove a Cloud Scheduler jobs via:

```shell
kubectl delete cloudschedulersources scheduler-test
```
