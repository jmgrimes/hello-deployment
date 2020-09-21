# Hello Deployment

Hello Deployment is a simple example of how to get Kubernetes manifests for 
a handful of trivial applications to deploy to a Docker Desktop installation 
of Kubernetes.

In particular, the NGINX Ingress Controller for Kubernetes is required to 
make this work, as the services are exposed using the NGINX virtual server 
construct, which is just significantly easier to reason about than the 
ever-changing Kubernetes Ingress resource.

## Prerequisites

### NGINX Ingress Controller for Kubernetes

To install the NGINX Ingress Controller, follow these instructions on your 
cluster.  We're installing as a Deployment with a NodePort service for local 
work, but using a DaemonSet on a non-local cluster is likely to make more 
sense.

```shell
git clone https://github.com/nginxinc/kubernetes-ingress/
cd kubernetes-ingress/deployments
git checkout v1.8.1

kubectl apply -f common/ns-and-sa.yaml
kubectl apply -f rbac/rbac.yaml

kubectl apply -f common/default-server-secret.yaml
kubectl apply -f common/nginx-config.yaml

kubectl apply -f common/vs-definition.yaml
kubectl apply -f common/vsr-definition.yaml
kubectl apply -f common/ts-definition.yaml
kubectl apply -f common/policy-definition.yaml

kubectl apply -f deployment/nginx-ingress.yaml
kubectl apply -f service/nodeport.yaml

kubectl get pods,services --namespace nginx-ingress
```

Take note of the ports used on your host for the HTTP and HTTPS endpoints.

### FluxCD Command Line Interface

To install the FluxCD CLI, homebrew is your best option on MacOS.  Simply run:

```shell
brew install fluxctl
```

### GitHub Packages Personal Access Token

To pull packages from GitHub into your Kubernetes cluster, you will need to 
either create an image pull secret with yours or your organization's personal
access token (PAT), or use the Docker CLI to have this added to your docker 
configuration.  Once you have your personal access token, run the following:

```shell
docker login docker.pkg.github.com -u <YOUR_GITHUB_USERNAME_OR_ORGANIZATION>
```

## Starting Up Flux

With all of our prerequisites in place, we're ready to install the FluxCD 
Operator and have it automatically deploy our manifests.

```shell
kubectl create ns flux

fluxctl install \
  --git-user <YOUR_GITHUB_USERNAME_OR_ORGANIZATION> \
  --git-email=<YOUR_GITHUB_USERNAME_OR_ORGANIZATION>@users.noreply.github.com \
  --git-url git@github.com:<YOUR_GITHUB_USERNAME_OR_ORGANIZATION>/hello-deployment \
  --namespace=flux | kubectl apply -f -

kubectl rollout status deployment/flux --namespace flux
```

Once this has completed, you'll need to give the FluxCD Operator access to the 
deployment GitHub repository with a deployment key.  When FluxCD initializes, 
it generates a key for this purpose.  You'll need to run the following command, 
then add this key to your repository's deployment keys section.  Make sure to 
allow this key write access for tagging purposes.

```shell
fluxctl identity --k8s-fwd-ns flux
```

At this point, you should be set up, and FluxCD will deploy this repository's 
contents to your cluster.  FluxCD polls for these changes every five minutes or so, 
so for the impatient, you can simply run:

```shell
fluxctl sync --k8s-fwd-ns flux
```

## Accessing the Application

The application is running on the "localhost" virtual server at the /hello-java 
path.  You can send a GET request to /hello-java/hello-world for the default greeting, 
or you can send a GET request to /hello-java/hello-world?name=YourNameHere to 
get a personalized message.