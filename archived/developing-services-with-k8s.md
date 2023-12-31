---
title: Locally developing microservices with Google Kubernetes Engine
description: Learn how to set up a development environment that lets you code and test changes locally, while connecting to other services running in Google Kubernetes Engine.
author: richarddli
tags: microservices, Kubernetes Engine, telepresence, PHP, Redis
date_published: 2017-04-05
---

<p style="background-color:#D9EFFC;"><i>Contributed by the Google Cloud community. Not official Google documentation.</i></p>

The [guestbook](https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook) tutorial for Kubernetes shows how to get a simple PHP and Redis application running in Kubernetes, but doesn't explain how you can actually *change* the code. We'll show you how to set up a fast, productive development environment for coding on Kubernetes. In particular, we'll show how you can make changes locally on your laptop, and see those changes reflected instantly on your externally exposed IP.

## Technologies used

* [Kubernetes](https://kubernetes.io)
* [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/)
* [Telepresence](http://www.telepresence.io)
* [PHP](http://www.php.net/) and [Redis](https://redis.io/)

## Prerequisites and setup

In order to use this demo, you're going to need the following:

* A local system running either Linux or macOS
* Access to a Kubernetes cluster (This tutorial will walk through setting up a cluster using Google Kubernetes Engine.)

## Microservices

Microservices are an increasingly popular design pattern for cloud applications. In a microservices architecture, an individual application is broken into many small services that can be independently developed, tested, and released. While this approach has numerous benefits, microservices can also bring additional complexity into your overall architecture and workflow.

One of the areas of complexity is setting up a productive development environment for microservices. In a traditional web application, a development environment may consist of a database and the actual web application. In a microservices cloud application, an individual service may depend on multiple other services. Moreover, the service may also utilize cloud resources such as Amazon RDS or Google Cloud Pub/Sub. Setting up and maintaining a development environment with multiple services and cloud resources can be a lot of work. While there are [multiple approaches to setting up a development environment for microservices](https://www.datawire.io/guide/deployment/development-environments-microservices/), this tutorial will walk through setting up a local development environment for microservices with your services running on a remote Kubernetes cluster.

In this tutorial, we're going to use the [Guestbook](https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook) sample application to illustrate a simple "microservices" architecture: the PHP service will represent one service, and the Redis database will represent another.

### Setting up your local computer

To set up your computer, you'll need to install a few basic components.

First, install the `gcloud` and `kubectl` command line tools. Follow the instructions at [https://cloud.google.com/sdk/downloads](https://cloud.google.com/sdk/downloads) to download and install the Cloud SDK. Then, ensure that `kubectl` is installed:

```
% sudo gcloud components update kubectl
```

You need to install Telepresence, which will proxy your locally running service to Google Kubernetes Engine. (For the latest 
installation instructions and documentation, visit [the Telepresence website](http://www.telepresence.io).)

On macOS (Homebrew 3 or later):

```
brew install --cask osxfuse
brew install datawire/blackbird/telepresence
```

On Ubuntu (16.04 or later):

```
curl -s https://packagecloud.io/install/repositories/datawireio/telepresence/script.deb.sh | sudo bash
sudo apt install --no-install-recommends telepresence
```

On Fedora (25 or later):

```
curl -s https://packagecloud.io/install/repositories/datawireio/telepresence/script.rpm.sh | sudo bash
sudo dnf install telepresence
```

We'll also need to configure a local development environment for PHP. The Guestbook application is fairly simple, but it does depend on the Predis library. We'll need to install the [PEAR package manager](https://pear.php.net/manual/en/installation.getting.php), and then install the Predis library.

```
% curl -O https://pear.php.net/go-pear.phar
% php go-pear.par
% pear channel-discover pear.nrk.io   # You may need to add pear to your path
% pear install nrk/Predis
```

Finally, this tutorial uses a number of Kubernetes configuration files. To save some typing, clone the [telepresence GitHub](https://github.com/datawire/telepresence/) repository:

```
% git clone https://github.com/datawire/telepresence.git
```

All example files are in the [`examples/guestbook`](https://github.com/datawire/telepresence/tree/master/examples/guestbook) directory.

### Setting up Kubernetes in Google Kubernetes Engine

Setting up a production-ready Kubernetes cluster can be fairly complex, so we're going to use Google Kubernetes Engine in our example. If you already have a Kubernetes cluster handy, you can skip this section.

To set up a Kubernetes cluster in Google Kubernetes Engine, go to
[https://console.cloud.google.com](https://console.cloud.google.com), choose the Google Kubernetes Engine option from the 
menu, and then **Create a cluster**.

The following `gcloud` command will create a small 2 node cluster in the us-central1-a region:

```
% gcloud container --project "PROJECT" clusters create "EXAMPLE_NAME" --zone "us-central1-a" --machine-type "n1-standard-1" --disk-size "100" --scopes "https://www.googleapis.com/auth/compute","https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "2" --network "default" --enable-cloud-logging --enable-cloud-monitoring
```

Finally, we can authenticate to our cluster:

```
% gcloud container clusters get-credentials CLUSTER_NAME
% gcloud auth application-default login
```

### The Guestbook application

Now that we have our laptop and cloud Kubernetes installation configured, we're going to start setting up the Guestbook application. We'll start by installing Redis in the cluster. We'll need to set up the Redis master deployment ([config](https://github.com/datawire/telepresence/blob/master/examples/guestbook/redis-master-deployment.yaml)), the Redis master service ([config](https://github.com/datawire/telepresence/blob/master/examples/guestbook/redis-master-service.yaml)), the Redis slave deployment ([config](https://github.com/datawire/telepresence/blob/master/examples/guestbook/redis-slave-deployment.yaml)), the Redis slave service ([config](https://github.com/datawire/telepresence/blob/master/examples/guestbook/redis-slave-service.yaml)), and the frontend PHP service ([config](https://github.com/datawire/telepresence/blob/master/examples/guestbook/main-service.yaml)) and deployment ([config](https://github.com/datawire/telepresence/blob/master/examples/guestbook/main-deployment.yaml)). If you don't want to download each of these files manually, these files are in the [`examples/guestbook`](https://github.com/datawire/telepresence/tree/master/examples/guestbook) directory of the Telepresence repository.

```
% kubectl apply -f redis-master-deployment.yaml
% kubectl apply -f redis-master-service.yaml
% kubectl apply -f redis-slave-deployment.yaml
% kubectl apply -f redis-slave-service.yaml
% kubectl apply -f main-deployment.yaml
% kubectl apply -f main-service.yaml
```

You can verify that everything is running:

```
% kubectl get pods
NAME                           READY     STATUS    RESTARTS   AGE
redis-master-343230949-lpw91   1/1       Running   0          1m
redis-slave-132015689-dpp46    1/1       Running   0          1m
redis-slave-132015689-v06md    1/1       Running   0          1m
frontend-34242342432-zx235     3/3       Running   0          1m
```

It's time to check out our app in the browser. Let's look up the IP address of our external load balancer:

```
% kubectl get services
NAME           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
frontend       10.7.252.209   104.196.217.24   80:30563/TCP   2m
redis-master   10.7.248.117   <none>           6379/TCP       2m
redis-slave    10.7.245.58    <none>           6379/TCP       2m
```

Go to the external IP address of your load balancer (in the above example, 104.196.217.24). You should see the Guestbook application running. Typing into the submit box will show how your message is persisting to the Redis cluster.

### Switching to local development

What if you want to try out some changes to your code, without having to redeploy it each time?

We're now going to use [Telepresence](http://www.telepresence.io) to create a virtual network between your local machine and the remote Kubernetes cluster by running this command:

```
% telepresence intercept frontend --port 8080:80
```

This tells Telepresence to send remote traffic to your local service instead of the service in the remote Kubernetes cluster.

Now, change to the `examples/guestbook` directory, and start the frontend application as follows. We'll need to know the directory where PHP can load its dependencies, e.g., Predis. You can figure this out by typing:

```
% pear config-get php_dir
```

Now, in the `examples/guestbook` directory, start PHP, and pass in the pear shared directory:

```
% php -d include_path="PATH_TO_PEAR_DIR" -S 0.0.0.0:8080
```

### Editing your code

Now, open `index.html` from your shell and try renaming the Submit button to Go. Save, hit reload. BEHOLD! You'll immediately see your changes reflected live on the external IP address.

Terminate the PHP process, and type `exit` to terminate the Telepresence proxy and swap back to the original deployed code.

### Behind the scenes

What's going on behind the scenes? Your incoming request goes to the load balancer. The load balancer, as mentioned above, looks for the Telepresence proxy based on the `app:guestbook` and `tier:frontend` labels. The proxy, which is running in the Google Kubernetes Engine environment, then sends those requests to the local Telepresence client, which passes the request to the PHP application.

## Additional resources

* [Setting up a Python development environment for Docker](http://matthewminer.com/2015/01/25/docker-dev-environment-for-web-app.html) covers how to configure your Docker image for hot reload
* The [Microservices Architecture Guide](https://www.datawire.io/guide) covers design patterns and HOWTOs in setting up an end-to-end microservices infrastructure
* The [Kubernetes tutorial](https://kubernetes.io/docs/tutorials/kubernetes-basics/) gives a good walk-through of using Kubernetes, or visit the [Google Kubernetes Engine quickstart](https://cloud.google.com/kubernetes-engine/docs/quickstart)

## Conclusion

Microservices, or service-oriented development, is a paradigm that is here to stay for cloud applications. The fledgling nature of microservices means that the tooling around developing, testing, and deploying microservices is still immature. This tutorial shows a practical way to set up fast, local development of a microservice while being able to utilize cloud resources running in a Kubernetes environment.
