# Load balancing an API using Skupper

## Overview

This example shows how to deploy a 3scale instance and two instances of an API.
WE will use 3 OCP namespaces, one for the API client and 2 for the API's, which will be replicated.
We will then create a Skupper VAN in the following way:

![Alt text](/images/img1.png?raw=true "Title")


![Alt text](/images/img2.png?raw=true "Title")


## Install the Skupper command-line tool

The `skupper` command-line tool is the entrypoint for installing
and configuring Skupper.  You need to install the `skupper`
command only once for each development environment.

See [Installing Skupper][install-docs].

[install-docs]: https://access.redhat.com/documentation/en-us/red_hat_application_interconnect/1.0/html/creating_a_service_network_with_openshift/installing-cli


## Configure separate console sessions

Skupper is designed for use with multiple namespaces, usually on
different clusters.  The `skupper` command uses your
[kubeconfig][kubeconfig] and current context to select the
namespace where it operates.

[kubeconfig]: https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/

Your kubeconfig is stored in a file in your home directory.  The
`skupper` and `kubectl` commands use the `KUBECONFIG` environment
variable to locate it.

A single kubeconfig supports only one active context per user.
Since you will be using multiple contexts at once in this
exercise, you need to create distinct kubeconfigs.

Start a console session for each of your namespaces.  Set the
`KUBECONFIG` environment variable to a different path in each
session.

_**Console for north:**_

~~~ shell
export KUBECONFIG=~/.kube/config-north
~~~

_**Console for west:**_

~~~ shell
export KUBECONFIG=~/.kube/config-west
~~~

_**Console for east:**_

~~~ shell
export KUBECONFIG=~/.kube/config-east
~~~

## Access your clusters

The procedure for accessing a Kubernetes cluster varies by
provider. [Find the instructions for your chosen
provider][kube-providers] and use them to authenticate and
configure access for each console session.

[kube-providers]: https://skupper.io/start/kubernetes.html

## Set up your namespaces

Use `kubectl create namespace` to create the namespaces you wish
to use (or use existing namespaces).  Use `kubectl config
set-context` to set the current namespace for each session.

_**Console for north:**_

~~~ shell
kubectl create namespace north
kubectl config set-context --current --namespace north
~~~

_**Console for west:**_

~~~ shell
kubectl create namespace west
kubectl config set-context --current --namespace west
~~~

_**Console for east:**_

~~~ shell
kubectl create namespace east
kubectl config set-context --current --namespace east
~~~

## Install Skupper in your namespaces

The `skupper init` command installs the Skupper router and service
controller in the current namespace.  Run the `skupper init` command
in each namespace.

_**Console for north:**_

~~~ shell
skupper init
~~~

_Sample output:_

~~~ console
$ skupper init
Skupper is now installed in namespace 'north'.  Use 'skupper status' to get more information.
~~~

_**Console for west:**_

~~~ shell
skupper init
~~~

_Sample output:_

~~~ console
$ skupper init
Skupper is now installed in namespace 'west'.  Use 'skupper status' to get more information.
~~~

_**Console for east:**_

~~~ shell
skupper init
~~~

_Sample output:_

~~~ console
$ skupper init
Skupper is now installed in namespace 'east'.  Use 'skupper status' to get more information.
~~~

##  Link your namespaces

_**Console for west:**_

~~~ shell
skupper token create ~/west.token
~~~

_Sample output:_

~~~ console
$ skupper token create ~/west.token
Token written to ~/west.token
~~~

_**Console for east:**_

~~~ shell
skupper token create ~/east.token
~~~

_Sample output:_

~~~ console
$ skupper token create ~/east.token
Token written to ~/east.token
~~~

_**Console for north:**_

~~~ shell
skupper link create ~/west.token
~~~

_Sample output:_

~~~ console
$ skupper link create ~/west.token
Site configured to link to https://10.105.193.154:8081/ed9c37f6-d78a-11ec-a8c7-04421a4c5042 (name=link1)
Check the status of the link using 'skupper link status'.
~~~

~~~ shell
skupper link create ~/east.token
~~~

_Sample output:_

~~~ console
$ skupper link create ~/east.token
Site configured to link to https://10.105.193.154:8081/ed9c37f6-d78a-11ec-a8c7-04421a4c5042 (name=link2)
Check the status of the link using 'skupper link status'.
~~~

## Deploy the API's

_**Console for west:**_

~~~ shell
oc new-app quay.io/skupper/hello-world-backend
~~~

_Sample output:_

~~~ console
....deployment.apps "hello-world-backend" created
~~~

_**Console for east:**_

~~~ shell
oc new-app quay.io/skupper/hello-world-backend
~~~

_Sample output:_

~~~ console
....deployment.apps "hello-world-backend" created
~~~

## Expose the API's

_**Console for west:**_

~~~ shell
skupper expose deployment/hello-world-backend --address hello-world-api --port 8080
~~~

_Sample output:_

~~~ console
$ skupper expose deployment/hello-world-backend --address hello-world-api --port 8080
deployment hello-world-backend exposed as hello-world-api
~~~

_**Console for east:**_

~~~ shell
skupper expose deployment/hello-world-backend --address hello-world-api --port 8080
~~~

_Sample output:_

~~~ console
$ skupper expose deployment/hello-world-backend --address hello-world-api --port 8080
deployment hello-world-backend exposed as hello-world-api
~~~


## Check logs

Start listening to the logs in both east and west namespaces. When moltiple request are sent together, the router will load balance them to both the APIs

_**Console for west and east:**_

~~~ shell
oc logs -f $(oc get pods -l deployment=hello-world-backend -o=jsonpath='{.items[0].metadata.name}')
~~~

## Test the API

_**Console for north:**_
~~~ shell
oc run curl-test --image=curlimages/curl -i --tty --rm=true -- sh
curl hello-world-api:8080/api/hello& \
curl hello-world-api:8080/api/hello&
~~~

_Sample output:_

~~~ console
$ oc run curl-test --image=curlimages/curl -i --tty --rm=true -- sh
~ $ curl hello-world-api:8080/api/hello& \
curl hello-world-api:8080/api/hello&
Hello, stranger.  I am Handy Hardware (hello-world-backend-6d8898687b-h64xs).
Hello, stranger.  I am Magnificent Mechanism (hello-world-backend-6d8898687b-zrz5v).
~~~
