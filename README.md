# Overview

Knative (pronounced kay-nay-tiv) extends Kubernetes to provide a set of middleware components that are essential to build modern, source-centric, and container-based applications that can run anywhere: on premises, in the cloud, or even in a third-party data center.

## Installation



GKE


1 - Create a cluster


gcloud container clusters create knative-cluster --zone=us-central1-a --cluster-version=latest --machine-type=n1-standard-4 --enable-autoscaling --min-nodes=1 --max-nodes=10 --enable-autorepair --scopes=service-control,service-management,compute-rw,storage-ro,cloud-platform,logging-write,monitoring-write,pubsub,datastore --num-nodes=3


2 - Install istio


kubectl apply --filename https://github.com/knative/serving/releases/download/v0.4.0/istio-crds.yaml && \
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.4.0/istio.yaml


2.2 Label the default namespace with istio-injection=enabled:


kubectl label namespace default istio-injection=enabled


2.3 Monitor the Istio components until all of the components show a STATUS of Running or Completed:


kubectl get pods --namespace istio-system --watch


3 - Install Knative


kubectl apply --filename https://github.com/knative/serving/releases/download/v0.4.0/serving.yaml \
--filename https://github.com/knative/build/releases/download/v0.4.0/build.yaml \
--filename https://github.com/knative/eventing/releases/download/v0.4.0/in-memory-channel.yaml \
--filename https://github.com/knative/eventing/releases/download/v0.4.0/release.yaml \
--filename https://github.com/knative/eventing-sources/releases/download/v0.4.0/release.yaml \
--filename https://github.com/knative/serving/releases/download/v0.4.0/monitoring.yaml \
--filename https://raw.githubusercontent.com/knative/serving/v0.4.0/third_party/config/build/clusterrole.yaml


3.1 - Monitor the Knative components until all of the components show a STATUS of Running:


kubectl get pods --namespace knative-serving
kubectl get pods --namespace knative-build
kubectl get pods --namespace knative-eventing
kubectl get pods --namespace knative-sources
kubectl get pods --namespace knative-monitoring

## Hello World

1 - Create a service.yaml


apiVersion: serving.knative.dev/v1alpha1 # Current version of Knative
kind: Service
metadata:
  name: helloworld-go # The name of the app
  namespace: default # The namespace the app will use
spec:
  runLatest:
    configuration:
      revisionTemplate:
        spec:
          container:
            image: gcr.io/knative-samples/helloworld-go # The URL to the image of the app
            env:
              - name: TARGET # The environment variable printed out by the sample app
                value: "Go Sample v1"


2 - Execute the command


kubectl apply -f service.yaml && kubectl get pods --watch


3 - Discovery the IP


kubectl get svc istio-ingressgateway --namespace istio-system


4 - Discovery the service HOST


kubectl get ksvc {service-name} --output=custom-columns=NAME:.metadata.name,DOMAIN:.status.domain


5 - Execute the curl


curl -H "Host: ${SERVICE_HOST}" http://${ISTIO_INGRESS_GATEWAY}

## Canary release

1 - Create a canary-v1.yaml


apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: canary
spec:
  runLatest:
    configuration:
      revisionTemplate:
        spec:
          container:
            image: gcr.io/knative-samples/knative-route-demo:blue
            env:
            - name: T_VERSION
              value: "blue"


2 - Execute the command


kubectl apply -f canary-v1.yaml


3 - Discovery the IP


kubectl get svc istio-ingressgateway --namespace istio-system


4 - Discovery the service HOST


kubectl get ksvc {service-name} --output=custom-columns=NAME:.metadata.name,DOMAIN:.status.domain


5 - Execute the curl


curl -H "Host: ${SERVICE_HOST}" http://${ISTIO_INGRESS_GATEWAY}


6 - Create a canary-v2.yaml


apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: canary
spec:
  release:
    revisions: ["canary-00001", "canary-00002"]
    rolloutPercent: 50
    configuration:
      revisionTemplate:
        spec:
          container:
            image: gcr.io/knative-samples/knative-route-demo:green
            env:
            - name: T_VERSION
              value: "green"


7- Execute the command


kubectl apply -f canary-v2.yaml


8 - Execute the curl
 
while true; do
    curl -s -H "Host: ${SERVICE_HOST}" http://${ISTIO_INGRESS_GATEWAY} | grep -E 'blue|green';
done

## Build

1 - Create a hello-knative-build.yaml


apiVersion: build.knative.dev/v1alpha1
kind: Build
metadata:
  name: example-build
spec:
  source:
    git:
      url: "https://github.com/knative/docs.git"
      revision: "v0.1.x"
    subPath: "serving/samples/helloworld-go/"
  steps:
  - name: build-and-push
    image: "gcr.io/kaniko-project/executor:v0.6.0"
    args:
    - "--dockerfile=/workspace/Dockerfile"
    - "--destination=gcr.io/supple-flux-233615/helloworld-go:v1"


2 - Execute the command:


kubectl apply -f hello-knative-build.yaml


3 - Follow the build


kubectl log -f -c build-step-build-and-push <pod-name>


4 - After the build is complete, you should see the Pod showing with Completed status


kubectl get pod <pod-name>

## Monitoring

1 - Runt he command:


kubectl port-forward --namespace knative-monitoring \
  $(kubectl get pods --namespace knative-monitoring \
      --selector=app=grafana --output=jsonpath="{.items..metadata.name}")\
  8080:3000


2 - Open the link:


http://localhost:8080/
