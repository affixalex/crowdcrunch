# CrowdCrunch Microservices Example

There are a lot of moving parts here in a seemingly simple example!

I initially wanted to use the most recent Istio 1.0 snapshot, but I think it is 
more prudent here to use the 0.8 LTS release and work with a stable foundation.

## Development Environment

I'm going to assume a MacOS X 10.13 environment with Homebrew installed. This 
also requires XCode to be installed, but the full setup of this environement is 
beyond my scope here. The Docker for Mac edge release includes Kubernetes, and 
we can easily destroy our local cluster and recreate it if needed. Very nice!

    brew install stern # For easily viewing logs
    brew install siege # For load testing components
    brew install kubernetes-helm
    brew install kubernetes-cli
    brew install git
    brew install gettext
    brew link --force gettext # needed to envsubst

It's reasonably safe to have a newer version of the `kubectl` command than your server version. These should align more precisely in the near future.

## Getting Into The Kubernetes Ecosystem

### Getting Started

Let's begin by making a directory to contain our work here.

    mkdir k8s

### Helm

Helm Charts make it fairly easy to describe and install complex Kubernetes deployments and it is an official Kubernetes project now, so we'll be using 
Helm in this application. Helm and Kubernetes suffer from a very uncommon 
issue: at times, it feels too easy.

### Istio

Istio is, in a word, awesome. The service mesh is a vital component of a modern 
microservice stack, and Istio is rapidly becoming the de facto standard. A 
great many problems are solved out of the box, leaving us to focus on our own 
problems. It just so happens that the problem here, the service mesh, is hugely 
impactful and crucial to understanding our system and how data flows through it.

With the imminent 1.0 release of Istio, we should be able to install Istio with 
Homebrew and Helm, consolidating this a bit. For now, we'll manually download 
the 0.8 release snapshot and install it with `helm`.

    wget https://github.com/istio/istio/releases/download/0.8.0/istio-0.8.0-osx.tar.gz
    tar -xzf istio-0.8.0-osx.tar.gz
    cd istio-0.8.0
    # Initialize a service account for Tiller
    kubectl apply -f install/kubernetes/helm/helm-service-account.yaml
    # Intall Tiller on your cluster with the service account.
    # We won't worry about securing Tiller for now!
    helm init --service-account tiller
    # Install CoreDNS. This comes out of the box in K8S 1.11
    helm install --name coredns --namespace=kube-system stable/coredns
    # Install Istio with tracing, Grafana, and Prometheus
    helm install install/kubernetes/helm/istio --name istio --namespace istio-system \
    --set tracing.enabled=true \
    --set tracing.jaeger.enabled=true \
    --set prometheus.enabled=true \
    --set grafana.enabled=true \
    --set global.mtls.enabled=true \
    --set global.controlPlaneSecurityEnabled=true

As Helm works its magic, we can watch the progress!

    kubectl get pods -n istio-system

In only a few lines, we've done quite a bit. Particularly noteworthy is that 
TLS is enabled both in the control plane and between service instances. We also 
have automatic sidecar container injection, so we don't really have to think 
about Envoy as application developers. This is pretty nice.

Let's investigate a bit.

    alex@amnesiac ~/crowdcrunch $ kubectl get pods -n istio-system
    NAME                                       READY     STATUS      RESTARTS   AGE
    grafana-6f6dff9986-pdmrc                   1/1       Running     0          1m
    istio-citadel-7bdc7775c7-2fd46             1/1       Running     0          1m
    istio-egressgateway-78dd788b6d-lxx64       1/1       Running     0          1m
    istio-ingress-98c5cdf88-nlffx              1/1       Running     0          1m
    istio-ingressgateway-7dd84b68d6-tthmf      1/1       Running     0          1m
    istio-mixer-post-install-65kxn             0/1       Completed   0          1m
    istio-pilot-d5bbc5c59-5trvf                2/2       Running     0          1m
    istio-policy-64595c6fff-wcnxq              2/2       Running     0          1m
    istio-sidecar-injector-645c89bc64-qkn82    1/1       Running     0          1m
    istio-statsd-prom-bridge-949999c4c-vcsdh   1/1       Running     0          1m
    istio-telemetry-cfb674b6c-ll8z2            2/2       Running     0          1m
    istio-tracing-754cdfd695-mwrqn             1/1       Running     0          1m
    prometheus-86cb6dd77c-7vhw6                1/1       Running     0          1m

Off to the races!

There are still many caveats to production deployment, but we're getting there.

### Observability with Kiali

Kiali is cool. I mean, it's really cool.

    curl https://raw.githubusercontent.com/kiali/kiali/v0.5.0/deploy/kubernetes/kiali-configmap.yaml | \
    VERSION_LABEL=master envsubst | kubectl create -n istio-system -f -

    curl https://raw.githubusercontent.com/kiali/kiali/v0.5.0/deploy/kubernetes/kiali-secrets.yaml | \
    VERSION_LABEL=master envsubst | kubectl create -n istio-system -f -

    curl https://raw.githubusercontent.com/kiali/kiali/v0.5.0/deploy/kubernetes/kiali.yaml | \
    IMAGE_NAME=kiali/kiali \
    IMAGE_VERSION=v0.5.0 \
    NAMESPACE=istio-system \
    VERSION_LABEL=master \
    VERBOSE_MODE=4 envsubst | kubectl create -n istio-system -f -

We're in business. Let's make Kiali accessible now...

     kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') 3000:20001
     open http://localhost:3000 # In a seperate shell

You can login with username and password, "admin"

I recommend using `tmux` or `screen` so you can easily keep a few sessions 
going at once without running kubectl in the background for port forwarding.

### Observability with Prometheus and Grafana

Kiali provides insight into the service mesh and the way that traffic is 
flowing through it, but sometimes we also want specific insights into our own 
application according to metrics and dashboards that we define. This is where 
Promethus and Grafana become invaluable, allowing us to get a very complete 
picture of how our application is behaving in realtime.

    kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000
    open http://localhost:3000 # In a seperate shell


## Writing Our Application 

We'll be using Node 10.x to mock up our microservices and present them to the 
world with [GraphQL stitching](https://codeburst.io/nodejs-graphql-micro-services-using-remote-stitching-7540030a0753). This is actually as cool as it sounds! One great 
thing about this approach is that it is language agnostic, so you can choose a 
different programming language if it is more well suited to a particular 
service. One current drawback of this approach is that there isn't a good 
solution for GraphQL Subscriptions but I'm confident that patterns will emerge 
soon.

The next series of commits to this repository will walk through the finer 
points of building our services and the decisions involved. Just now, at a 
glance, you can see that this repository is itself a Helm chart. Deployment and 
rolling upgrades are going to be a fundamental part of the design here!

## Chaos Engineering

Coming soon! We'll introduce failures to our system and see how it responds.

## Preparing for Production

In a real application, we'll want a multi master Kubernetes installation...