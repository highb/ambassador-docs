---
  reading_time: 10
  reading_time_text: tutorial
---

import Alert from '@material-ui/lab/Alert';
import KubeGSTabs from './gs-tabs.js'

# Get code running on my cluster

<div class="docs-article-toc">
<h3>Contents</h3>

* [Prerequisites](#prerequisites)
* [1. Install ingress controller](#1-install-ingress-controller)
* [2. Bulletin board web app](#2-bulletin-board-web-app)
* [3. Build container and test](#3-build-container-and-test)
* [4. Push to Docker Hub](#4-push-to-docker-hub)
* [5. Create a Deployment, Service, and Mapping in Kubernetes](#5-create-a-deployment-service-and-mapping-in-kubernetes)
* [6. Deploy app and test](#6-deploy-app-and-test)
* [7. Setup a Host and SSL (optional)](#7-setup-a-host-and-ssl-optional)
* [What's next?](#img-classos-logo-srcimageslogopng-whats-next)

</div>

This guide will walk you through going from code to building a Docker container to running that container in a Kubernetes cluster.

## Prerequisites

* [Kubectl](../../quick-start/#1-kubectl)
* a Kubernetes cluster <!--([minikube](../howtos/howtos/devenv/#minikube) would work if you don't have access to a real cluster)-->
* [Docker](https://docs.docker.com/get-docker/) and a basic knowledge of running and building containers
* A Docker Hub account ([sign up](https://hub.docker.com) if you don't have one)

## 1. Install ingress controller

We'll need an ingress controller for your cluster to get traffic from the internet to your app.  We'll use [Ambassador's Edge Stack](../../../../../products/edge-stack/) for this. 

**We recommend installing using Helm** but there are other options below to choose from.

<KubeGSTabs/>

If needed, see the list of <a href="../../../../edge-stack/latest/topics/install/" target="_blank">other installation options</a>.

Now that your cluster is ready to go, let's check out the app we're going to use.

## 2. Bulletin board web app

We're going to use a sample web app provided by Docker: a bulletin board written using Node.js.  The repo contains the app's code and a Dockerfile for building a container.

Start by cloning the repo and switching into the directory containing the Dockerfile.

```
git clone https://github.com/dockersamples/node-bulletin-board.git
cd node-bulletin-board/bulletin-board-app
```

Open and inspect the Dockerfile. It follows these basic steps:

* use the latest Node.js slim base image
* install dependencies using npm
* expose port 8080 on the container
* start the Node.js server using npm

## 3. Build container and test

Next, we'll build the container and start running it locally, to make sure everything works and see what the app looks like.  Make sure Docker is running, then run these commands to build and run the container (**substituting in your Docker Hub username**):

```
docker build . -t <your docker hub username>/nodebb:1.0
docker run -d -p 80:8080 --name nodebb <your Docker Hub username>/nodebb:1.0
```

Let's look at the flags on the `docker run` command:

* `-d` runs in detached mode, runs in the background after starting 
* `-p` sets a localhost port to map to a container port, in this case http://localhost (implying port 80) will map to 8080 on the container
* `--name` sets the name of the running container
* `<your Docker Hub username>/nodebb:1.0` is the built image that the container will run

Go to [http://localhost](http://localhost) to see the web app in action.

<Alert severity="success">
<strong>Success!</strong> The container is built and running locally!
</Alert>

Let's stop and remove the container before moving on:

```
docker stop nodebb
docker rm nodebb
```

## 4. Push to Docker Hub

To deploy the app to your cluster, Kubernetes must be able to pull the image from a repository.  In this case, we are using Docker Hub.  First, you must log in to Docker Hub on the Docker CLI to authorize it to push to your account:

`docker login`

Login to your account as prompted.  

Next, push the `nodebb` image to Docker Hub:

`docker push <your Docker Hub username>/nodebb:1.0`

When it finishes go to [Docker Hub](https://hub.docker.com/) and you should see your image as a public repository

## 5. Create a Deployment, Service, and Mapping in Kubernetes

Save this file as `nodebb.yaml`, **replacing the value for your Docker Hub username**.  

This manifest file first creates a Deployment, which defines and runs the Pod.  Pods in Kubernetes are usually made up of a single container, in this case, the `nodebb` container you pushed to Docker Hub. 

<Alert severity="info">
  <a href="../../concepts/basics">Learn more about the basics of Kubernetes.</a>
</Alert>

Next, it creates a Service, which handles getting the traffic on the specified port to the Pod.

Finally, it creates a [Mapping](../../../../edge-stack/latest/topics/using/intro-mappings/#introduction-to-the-mapping-resource), which is used by Edge Stack to expose a Service to the internet at a specific URL prefix, `/` in this case (as in the root of your hostname or IP address, like `http://google.com/` or `http://1.2.3.4/`)

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodebb-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nodebb
  template:
    metadata:
      labels:
        app: nodebb
    spec:
      containers:
      - name: backend
        image: docker.io/<your Docker Hub username>/nodebb:1.0
        ports:
        - name: http
          containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: nodebb-service
spec:
  ports:
  - name: http
    port: 80
    targetPort: 8080
  selector:
    app: nodebb
---
apiVersion: getambassador.io/v2
kind: Mapping
metadata:
  name: nodebb-mapping
spec:
  prefix: /
  service: nodebb-service

```

Compare the Deployment and Service to our previous `docker run` command.  You should see things that look familiar, like the image repository, name, and tag, and the port on the container being mapped to a port on the host.

Also, notice how certain values match across the different resources?  For example, the Deployment's `spec.template.metadata.labels` value and the Service's `spec.selector` value match.  This is essential to connect these different cluster resources together so that the Services knows what Deployment to send the traffic to.

## 6. Deploy app and test

Deploy the YAML file with `kubectl apply -f nodebb.yaml`.

Get IP address of Edge Stack (the ingress controller that you installed at the beginning of this guide):

```
kubectl -n ambassador get svc ambassador \
-o "go-template={{range .status.loadBalancer.ingress}}{{or .ip .hostname}}{{end}}"
```

Finally, go to `http://<load balancer IP>/` and you should see your app.

<Alert severity="info">
  If you get an error in your browser about the certificate being invalid, just click <strong>Proceed</strong>. Edge Stack forwarded you to an HTTPS version of the site by default and is using a self-signed certificate, which is fine for this guide.  In a production deployment you can use Edge Stack to generate a valid certificate automatically, which we'll do in the next step.
</Alert>

<Alert severity="success">
<strong>Victory!</strong> You went from code to a web app running in Kubernetes!
</Alert>

## 7. Setup a Host and SSL (optional)

If you have a registered domain name then you can take this one step further by setting up a [Host resource to provision an SSL certificate](../../../../edge-stack/latest/topics/running/host-crd/).

You'll first need to create an A record at your DNS provider.  We suggest a subdomain like `test.yourdomain.com`.  For the A record's IP address use the load balancer IP from the previous step.

Once the record is created and you give it a few minutes to propagate across the internet, save the following as `host.yaml`, filling in your domain name and email address.

```yaml
apiVersion: getambassador.io/v2
kind: Host
metadata:
  name: nodebb-host
spec:
  hostname: <your domain name>
  acmeProvider:
    email: <your email address>
```

This tells Edge Stack to use your domain name to generate and install an SSL certificate via Let's Encrypt.  Apply the file with `kubectl apply -f host.yaml` then wait a moment for the certificate to generate.  

You can check the certificate's status with `kubectl describe host nodebb-host`.  When complete you should see an event saying `Host with ACME-provisioned TLS certificate marked Ready`.

Finally, go to your domain in the browser and you should see the app with a valid SSL certificate installed.

## <img class="os-logo" src="../../../../../images/logo.png"/> What's next?

YAML files used to deploy Kubernetes resources are generally kept under version control.  You can automate deploying and updating resources as changes are committed to your code repositories using a CI/CD system like Argo!  [Check out our guide on getting started with Argo](../../../../argo/latest/quick-start/).

Debugging services running on Kubernetes can be a frustrating process, [make it easier and faster using Telepresence](../../../../telepresence/latest/quick-start/)!