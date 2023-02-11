# Getting started
This repository initializes an existing helm chart that can be managed by a kubernetes operator. 
For testing purposes, it is deployed on a local minikube cluster. 
The helm chart for initializing is a simple chart template, which has a deployment and a service. 

## Prerequisites
In order to run this example, you need to have following tools installed: 
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/)
- [minikube](https://minikube.sigs.k8s.io/docs/) or 
- [docker desktop](https://www.docker.com/products/docker-desktop/) with [kubernetes enabled](https://docs.docker.com/desktop/kubernetes/) and [dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) installed
- [operator-sdk](https://sdk.operatorframework.io/docs/installation/#install-from-github-release)


## Init operator project
Open a terminal, create a folder where the operator should be set up and run the following commands:
````sh
# scaffold operator structure
$ operator-sdk init --plugins=helm --domain=example.com 
Writing kustomize manifests for you to edit...
Next: define a resource with:
operator-sdk create api
# create api for helm chart
$ operator-sdk create api --group operators --version v1 --kind Example-platform
Writing kustomize manifests for you to edit...
Created helm-charts/example-platform
WARN[0000] Using default RBAC rules: failed to get Kubernetes config: invalid configuration: no configuration has been provided, try setting KUBERNETES_MASTER environment variable
````
A folder structure should now be visible with boilerplate code that has to be adjusted. 
We will adjust the necessary values for minikube deployment in a later step. 


## Deploy on Minikube
Now we have to start the local minikube instance. 
It´s also very useful to open the dashboard to track deployments and status of the operator installation: 
````sh
# start minikube
$ minikube start
# check if minikube is running
$ minikube status
# open minikube dashboard
$ minikube dashboard
````

The dashboard should now open:
![Dashboard](/figures/kubernetes_dashboard.png)

## Use local docker cache
In order for minikube to make use of the local pulled docker images, an eval command has to be executed in the terminal to map the docker engine to minikube. 
Otherwise a proxy to an image registry needs to be configured. 

````
# point minikube docker engine to terminal currently used
eval $(minikube -p minikube docker-env)
````

## Build local controller image
In order to grant minikube access to the controller docker image, cd to the root of the project and run a docker build in the terminal the eval command was executed before. In the dockerfile, the sources like helm charts and watches are copied which will be available as a custom resource definition as soon as the operator is deployed. 
````sh
docker build . -t controller:latest
````

# Running the operator
You have two choices of developing operators. 
1. Deploy the operator on the minikube cluster <br>
This will create a separate namespace where the operator image contianing the helm chart is build. The Custom Resource definitions are then available in all namespaces.
2. Run the operator locally <br>
This will run the operator locally in the terminal and expose the Custom Resource Definitions to minikube. 

## 1. Deploy the operator on minikube cluster
This is the more tortious way because all the docker images have to be available for minikube. You can either set up a proxy to an image registry or run a manual `docker pull` to have the images downloaded in the minikube docker cache. 

### Pull required images into minikube
Take the terminal you executed the `eval` command before to have access to minikubes docker cache and pull required images for the operator: 
````sh
# login to docker registry. If you don´t have a redhat account, crate one via developer.redhead.com
docker login registry.redhat.io
Username: XXX
Password: ******
Login Succeeded

# pull docker image from redhat registry 
docker pull registry.redhat.io/openshift4/ose-kube-rbac-proxy:v4.11
v4.11: Pulling from openshift4/ose-kube-rbac-proxy
97da74cc6d8f: Pull complete 
d8190195889e: Pull complete 
ad44d7582bbb: Pull complete 
1e85b2b37152: Pull complete 
6e5d4e83cb3e: Pull complete 
Digest: sha256:ac54cb8ff880a935ea3b4b1efc96d35bbf973342c450400d6417d06e59050027
Status: Downloaded newer image for registry.redhat.io/openshift4/ose-kube-rbac-proxy:v4.11
registry.redhat.io/openshift4/ose-kube-rbac-proxy:v4.11
````

### Adjust project
in makefile, change IMG value to dockerfile build and tag 
````sh
# Image URL to use all building/pushing image targets
IMG ?= example-platform:latest
````
Adjust `config/manager/manager.yaml` to be compatible with minikube:
````yaml
        runAsNonRoot: true
        imagePullPolicy: Never
````
Execute following command to deploy the operator image to a separate namespace:
````
$ make deploy
cd config/manager && /home/user/repos/example-operator/bin/kustomize edit set image controller=controller:latest
/home/user/repos/example-operator/bin/kustomize build config/default | kubectl apply -f -
namespace/example-operator-system created
customresourcedefinition.apiextensions.k8s.io/example-platforms.operators.example.com created
serviceaccount/example-operator-controller-manager created
role.rbac.authorization.k8s.io/example-operator-leader-election-role created
clusterrole.rbac.authorization.k8s.io/example-operator-manager-role created
clusterrole.rbac.authorization.k8s.io/example-operator-metrics-reader created
clusterrole.rbac.authorization.k8s.io/example-operator-proxy-role created
rolebinding.rbac.authorization.k8s.io/example-operator-leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/example-operator-manager-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/example-operator-proxy-rolebinding created
service/example-operator-controller-manager-metrics-service created
deployment.apps/example-operator-controller-manager created
````

Now the Deployment should be visible in the minikube dashboard: 

![Dashboard](/figures/operator_deployment.png)

Also the custom resource definition should be visible in the project: 

![Dashboard](/figures/operator_crd_visible.png)


## 2. Run the operator locally
This is the easier way to develop operators and explore the lifecycle behavior as there are good log outputs from the operator and also no docker image or image registry dependencies. <br>

Execute following command to run the operator locally and just expose the created CRD to minikube. In the terminal, log output should be visible describing the kubectl apply commands and a message watching resources 
````sh
make install run
/home/user/repos/example-operator/bin/kustomize build config/crd | kubectl apply -f -
customresourcedefinition.apiextensions.k8s.io/example-platforms.operators.example.com created
/home/user/repos/example-operator/bin/helm-operator run
{"level":"info","ts":1675928925.1594994,"logger":"cmd","msg":"Version","Go Version":"go1.19.5","GOOS":"linux","GOARCH":"amd64","helm-operator":"v1.27.0","commit":"5cbdad9209332043b7c730856b6302edc8996faf"}
{"level":"info","ts":1675928925.1611917,"logger":"cmd","msg":"Watch namespaces not configured by environment variable WATCH_NAMESPACE or file. Watching all namespaces.","Namespace":""}
{"level":"info","ts":1675928925.176321,"logger":"controller-runtime.metrics","msg":"Metrics server is starting to listen","addr":":8080"}
{"level":"info","ts":1675928925.177788,"logger":"helm.controller","msg":"Watching resource","apiVersion":"operators.example.com/v1","kind":"Example-platform","namespace":"","reconcilePeriod":"1m0s"}
{"level":"info","ts":1675928925.1786354,"msg":"Starting server","path":"/metrics","kind":"metrics","addr":"[::]:8080"}
{"level":"info","ts":1675928925.178664,"msg":"Starting server","kind":"health probe","addr":"[::]:8081"}
{"level":"info","ts":1675928925.1790054,"msg":"Starting EventSource","controller":"example-platform-controller","source":"kind source: *unstructured.Unstructured"}
{"level":"info","ts":1675928925.1790779,"msg":"Starting Controller","controller":"example-platform-controller"}
{"level":"info","ts":1675928925.280455,"msg":"Starting workers","controller":"example-platform-controller","worker count":16}
````

A Custom Resource definition as shown above should also be visible in the default namespace.

## Install Operator and check installed helm chart
Right now our operator is deployed and ready to use. if you want to make use of the underlying helm chart, you have to crate and apply a custom resource definition. As soon as the custom resource definition is applied, the operator recognizes it and installs the helm chart in the namespace.

An example yaml definition is provided by the operator-sdk and stored in `config/samples/operators_v1_example-platform.yaml`. 

All values from your chart `values.yaml` are already included in the sample and can be customized: 

````yaml
apiVersion: operators.example.com/v2
kind: Example-platform
metadata:
  name: example-platform-sample
spec:
  # Default values copied from <project_dir>/helm-charts/example-platform/values.yaml
  affinity: {}
  autoscaling:
    enabled: false
    maxReplicas: 100
    minReplicas: 1
    ...
````

Let´s apply it to the default namespace in minikube:

````sh
kubectl apply -f operators_v1_example-platform.yaml
````

The deployment should now be visible in the default namespace.

![Dashboard](/figures/crd_deployment.png)

If you check the log output from the running operator, it recognizes the apply of the Custom Resource definition and reconciles the release:
````sh
I0209 08:51:33.570619    6570 request.go:682] Waited for 1.033958682s due to client-side throttling, not priority and fairness, request: GET:https://kubernetes.docker.internal:6443/apis/batch/v1?timeout=32s
{"level":"info","ts":1675929093.73841,"logger":"helm.controller","msg":"Reconciled release","namespace":"default","name":"example-platform-sample","apiVersion":"operators.example.com/v1","kind":"Example-platform","release":"example-platform-sample"}
````

Also the helm chart with revision should exist. 
````sh
$ helm list 
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS       CHART                    APP VERSION
example-platform-sample default         1               2023-02-09 09:15:56.6603448 +0100 CET   deployed     example-platform-0.1.0   1.17.0
````

## Apply changes to the resources

So the operator successfully installed the helm chart wrapped in the Custom Resource Definition of kind `example-platform` in revision 1 in our default namespace. 
Lets make some changes to the custom resource. <br>
from the `config/samples/operators_v1_example-platform.yaml`, change the `.spec.service.port` from 80 to 8080 and apply it: 
````sh
$ kubectl apply -f operators_v1_example-platform.yaml
example-platform.operators.example.com/example-platform-sample configured
````
As you can see, the CRD gets reconfigured. The output from the running operator recognizes the change and applies it to the deployed objects

![Dashboard](/figures/crd_apply_changes.png)


# Upgrade helm chart 
Let´s test if the content in `watches.yaml` is listening for changes, i.e. when upgrading the helm chart. To do this, we have to run the operator locally using the install run command: 

````sh
$ make install run
/usr/local/bin/kustomize build config/crd | kubectl apply -f -
customresourcedefinition.apiextensions.k8s.io/example-platforms.operators.example created
/home/smolauserlo/repos/test/bin/helm-operator run
{"level":"info","ts":1675751292.9460497,"logger":"cmd","msg":"Version","Go Version":"go1.18.3","GOOS":"linux","GOARCH":"amd64","helm-operator":"v1.22.0","commit":"9e95050a94577d1f4ecbaeb6c2755a9d2c231289"}
````

The operator is now running locally with the currently configured minikube cluster and listening for changes in the `helm-charts/example-platform` directory.

There should already be an installed helm chart when running helm list. We modify the content of the field `appVersion` in  `Chart.yaml` to simulate an upgrade of the helm chart and check the log output from the operator after saving. (Note: it can take up to one minute for the watcher to check for updates) 

````sh
/config/samples$ helm list
NAME                            NAMESPACE       REVISION        UPDATED                              STATUS   CHART                   APP VERSION
example-platform-sample        default         1               2023-02-07 07:29:34.0835967 +0100 CETdeployed example-platform-0.1.0 1.16.0
````
- Update `appVersion` field in `Chart.yaml` 
- Check the log output in the terminal the operator is running in:

````sh
{"level":"info","ts":1675929822.8833482,"logger":"helm.controller","msg":"Upgraded release","namespace":"default","name":"example-platform-sample","apiVersion":"operators.example.com/v1","kind":"Example-platform","release":"example-platform-sample","force":false}
I0209 09:03:43.957269    6570 request.go:682] Waited for 1.028116888s due to client-side throttling, not priority and fairness, request: GET:https://kubernetes.docker.internal:6443/apis/flowcontrol.apiserver.k8s.io/v1beta2?timeout=32s
{"level":"info","ts":1675929824.118914,"logger":"helm.controller","msg":"Reconciled release","namespace":"default","name":"example-platform-sample","apiVersion":"operators.example.com/v1","kind":"Example-platform","release":"example-platform-sample"}
````

As we can see from the log output above, the watcher is recognizing the changes and runs a helm upgrade from the changed helm resources. If we check the installed helm chart via a `helm list` command, we can see the revision was increased to `2` and version was updated to the version we specified in the `appVersion` field of the `Chart.yaml` file.

````sh
/config/samples$ helm list
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
example-platform-sample default         2               2023-02-09 09:03:42.81794404 +0100 CET  deployed        example-platform-0.1.0  1.17.0
````

## Dealing with breaking changes
Our complete application lifecycle for helm upgrades is handled now. But what about breaking changes which shouldn´t applied automatically on deployed instances? <br>
In order to do this, we have to add another version of our Custom Resource Definition which has to be applied manually. 
This can be done in `config/bases/operators.example_example-platforms.yaml` file. In `.spec.versions[]` we can copy the content from `v1` and paste it to `v2` to create a new API version. <br>

````yaml
  - name: v2
    schema:
      openAPIV3Schema:
        description: Example-platform is the Schema for the example-platforms API
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: Spec defines the desired state of Example-platform
            type: object
            x-kubernetes-preserve-unknown-fields: true
          status:
            description: Status defines the observed state of Example-platform
            type: object
            x-kubernetes-preserve-unknown-fields: true
        type: object
    served: true
    storage: false
    subresources:
      status: {}
````

After doing this, you have to restart the local operator because the file content is not defined in watches.yaml: 

````sh
$ CTRL + c
{"level":"info","ts":1675753456.3530438,"msg":"Stopping and waiting for leader election runnables"}
{"level":"info","ts":1675753456.353102,"msg":"Shutdown signal received, waiting for all workers to finish","controller":"example-platform-controller"}
{"level":"info","ts":1675753456.353133,"msg":"All workers finished","controller":"example-platform-controller"}
{"level":"info","ts":1675753456.3531446,"msg":"Stopping and waiting for caches"}
{"level":"info","ts":1675753456.3533165,"msg":"Stopping and waiting for webhooks"}
{"level":"info","ts":1675753456.3533404,"msg":"Wait completed, proceeding to shutdown the manager"}

$ make install run
/usr/local/bin/kustomize build config/crd | kubectl apply -f -
customresourcedefinition.apiextensions.k8s.io/example-platforms.operators.example configured
/home/smolauserlo/repos/test/bin/helm-operator run
{"level":"info","ts":1675753458.0622938,"logger":"cmd","msg":"Version","Go Version":"go1.18.3","GOOS":"linux","GOARCH":"amd64","helm-operator":"v1.22.0","commit":"9e95050a94577d1f4ecbaeb6c2755a9d2c231289"}
````

The new version should now be visible under Custom Resource Definitions on the minikube dashboard:
![Dashboard](/figures/crd_v2.png)


To use the breaking changes, we also have to apply a new Custom Resource definition in the cluster. To do this, copy the file `/config/samples/operators_v1_example-platform.yaml` and name it to `operators_v2_example-platform.yaml`. 

Also, the content of the field `apiVersion` has to be updated from `apiextensions.k8s.io/v1` to `apiextensions.k8s.io/v2`.
After doint his, we can apply the file to our minikube cluster. <br>
Note: We have to delete the `v1` CRD first because its not possible to deploy two helm chart with the same release name in a namespace.  

````sh
/config/samples$ kubectl delete -f operators_v1_example-platform.yaml
example-platform.operators.example "example-platform-sample" deleted

/config/samples$ kubectl apply -f operators_v2_example-platform.yaml
example-platform.operators.example/example-platform-sample configured
````

The helm chart with the Custom Resource Definition of kind `example-platform` in version `v2` is now available in the namespace with a fresh Revision: 

````sh
$ helm list
NAME                            NAMESPACE       REVISION        UPDATED                              STATUS   CHART                   APP VERSION
example-platform-sample        default         1               2023-02-07 08:10:17.041081 +0100 CET deployed example-platform-0.1.0 1.17.0
````