# Skupper Hello World

## Overview

This example is a very simple multi-service HTTP application deployed across Kubernetes clusters using Skupper.

It contains two services:

* A backend service that exposes an `/api/hello` endpoint.  It returns greetings of the form
  
   `Hi, <your-name>.  I am <my-name>(<pod-name>)`.

* A frontend service that sends greetings to the backend and fetches new greetings in response.

With Skupper, you can place the backend in one cluster and the frontend in another and maintain connectivity between the two
services without exposing the backend to the public internet.

<img src="images/entities.svg" width="640"/>

## Prerequisites

* The `oc` command-line tool (https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/)
* 2 OpenShift Clusters
* OpenShfit access account with create project permission

## Step 1: Install the Skupper command-line tool

On Linux or Mac, you can use the install script (inspect it [here][install-script]) to download and extract the command:

~~~ shell
curl https://skupper.io/install.sh | sh
~~~

The script installs the command under your home directory.  It prompts you to add the command to your path if necessary.

Or place `skupper` binary to /usr/local/bin directory

Please do not use 1.5.0 for now

## Step 2: Access your clusters

Login cluster 1 from one terminal

```
oc login https://api.hub.jonkey.cc:6443 -u admin
```

Login cluster 2 from another terminal

```
oc login https://api.ocp11.jonkey.cc:6443 -u admin
```

## Step 3: Create your west & east projects

_**Console for west:**_

```
oc new-project west
```

_**Console for east:**_

```
oc new-project east
```

## Step 4: Install Skupper in your projects

The `skupper init` command installs the Skupper router and service controller in the current namespace.  Run the `skupper init` command in each project.

_**Console for west:**_

~~~ shell
skupper init
~~~

_**Console for east:**_

~~~ shell
skupper init
~~~

_Sample output:_

~~~ console
$ skupper init
Waiting for LoadBalancer IP or hostname...
Skupper is now installed in namespace '<namespace>'.  Use 'skupper status' to get more information.
~~~

## Step 5: Check the status of your projects

Use `skupper status` in each console to check that Skupper is installed.

_**Console for west:**_

~~~ shell
skupper status
~~~

_**Console for east:**_

~~~ shell
skupper status
~~~

_Sample output:_

~~~ console
Skupper is enabled for namespace "<namespace>" in interior mode. It is connected to 1 other site. It has 1 exposed service.
The site console url is: <console-url>
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'
~~~

As you move through the steps below, you can use `skupper status` at any time to check your progress.

## Step 6: Link your projects

Creating a link requires use of two `skupper` commands in conjunction, `skupper token create` and `skupper link create`.

The `skupper token create` command generates a secret token that signifies permission to create a link.  The token also carries the link details.  Then, in a remote project, The `skupper link create` command uses the token to create a link to the project that generated it.

**Note:** The link token is truly a *secret*.  Anyone who has the token can link to your project.  Make sure that only those you
trust have access to it.

First, use `skupper token create` in one project to generate the token.  Then, use `skupper link create` in the other to create a link.

_**Console for west:**_

~~~ shell
skupper token create ~/secret.token
~~~

_Sample output:_

~~~ console
$ skupper token create ~/secret.token
Token written to ~/secret.token
~~~

_**Console for east:**_

~~~ shell
skupper link create ~/secret.token
~~~

_Sample output:_

~~~ console
$ skupper link create ~/secret.token
Site configured to link to https://10.105.193.154:8081/ed9c37f6-d78a-11ec-a8c7-04421a4c5042 (name=link1)
Check the status of the link using 'skupper link status'.
~~~

If your console sessions are on different machines, you may need to use `sftp` or a similar tool to transfer the token securely.
By default, tokens expire after a single use or **15** **minutes** after creation.

## Step 7: Deploy the frontend and backend services

Use `oc create deployment` to deploy the frontend service in `west` and the backend service in `east`.

_**Console for west:**_

~~~ shell
oc create deployment frontend --image quay.io/skupper/hello-world-frontend
~~~

_Sample output:_

~~~ console
$ oc create deployment frontend --image quay.io/skupper/hello-world-frontend
deployment.apps/frontend created
~~~

_**Console for east:**_

~~~ shell
oc create deployment backend --image quay.io/skupper/hello-world-backend --replicas 3
~~~

_Sample output:_

~~~ console
$ oc create deployment backend --image quay.io/skupper/hello-world-backend --replicas 3
deployment.apps/backend created
~~~

## Step 8: Expose the backend service

We now have two projects linked to form a Skupper network, but no services are exposed on it.  Skupper uses the `skupper
expose` command to select a service from one project for exposure on all the linked projects.

**Note:** You can expose services that are not in the same project where you installed Skupper as described in the
[Exposing services from a different namespace][different-ns] documentation.

[different-ns]: https://skupper.io/docs/cli/index.html#exposing-services-from-different-ns

Use `skupper expose` to expose the backend service to thefrontend service.

_**Console for east:**_

~~~ shell
skupper expose deployment/backend --port 8080
~~~

_Sample output:_

~~~ console
$ skupper expose deployment/backend --port 8080
deployment backend exposed as backend
~~~

## Step 9: Expose the frontend service

We have established connectivity between the two projects and made the backend in `east` available to the frontend in `west`.
Before we can test the application, we need external access to the frontend.

Use `kubectl expose` with `--type LoadBalancer` to open network access to the frontend service.

_**Console for west:**_

~~~ shell
oc expose deployment/frontend --port 8080
oc expose svc/frontend
~~~

_Sample output:_

~~~ console
$ oc expose deployment/frontend --port 8080
service/frontend exposed
$ oc expose svc/frontend
route/frontend exposed
~~~

## Step 10: Test the application

Now we're ready to try it out.  Use `oc get route/frontend` to look up the external IP of the frontend service.  Then use
`curl` or a similar tool to request the `/api/health` endpoint at that address.

**Note:** The `<external-ip>` field in the following commands is a placeholder.  The actual value is an IP address.

_**Console for west:**_

~~~ shell
kubectl get service/frontend
curl http://<external-ip>/api/health
~~~

If everything is in order, you can now access the web interface by navigating to `http://<route url>` in your browser.

## Accessing the web console

Skupper includes a web console you can use to view the application network.  To access it, use `skupper status` to look up the URL of the web console.  Then use `oc get secret/skupper-console-users` to look up the console admin password.

**Note:** The `<console-url>` and `<password>` fields in the following output are placeholders.  The actual values are specific
to your environment.

_**Console for west:**_

~~~ shell
skupper status
oc get secret/skupper-console-users -o jsonpath={.data.admin} | base64 -d
~~~

_Sample output:_

~~~ console
$ skupper status
Skupper is enabled for namespace "west" in interior mode. It is connected to 1 other site. It has 1 exposed service.
The site console url is: <console-url>
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'

$ oc get secret/skupper-console-users -o jsonpath={.data.admin} | base64 -d
<password>
~~~

Navigate to `<console-url>` in your browser.  When prompted, log in as user `admin` and enter the password.

## Cleaning up

To remove Skupper and the other resources from this exercise, use the following commands.

_**Console for west:**_

~~~ shell
oc delete project west
~~~

_**Console for east:**_

~~~ shell
oc delete project east
~~~

## Summary

This example locates the frontend and backend services in different projects, on different clusters.  Ordinarily, this means that they have no way to communicate unless they are exposed to the public internet.

Introducing Skupper into each project allows us to create a virtual application network that can connect services in different clusters.
Any service exposed on the application network is represented as a local service in all of the linked projects.

The backend service is located in `east`, but the frontend service in `west` can "see" it as if it were local.  When the frontend
sends a request to the backend, Skupper forwards the request to the namespace where the backend is running and routes the response back to the frontend.

<img src="images/sequence.svg" width="640"/>

