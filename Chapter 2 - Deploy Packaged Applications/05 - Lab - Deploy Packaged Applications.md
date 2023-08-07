# Lab: Deploy Packaged Applications

* Deploy and update applications from resource manifests that are packaged for sharing and distribution.

## Outcomes

* Deploy an application and its dependencies from resource manifests that are packaged as a Helm chart.
* Update the application to a later version by using the Helm chart.
* Use a container image in a private container registry instead of a public registry.
* Customize the deployment to add resource requests and limits.

## Instructions

### (1) Log in to the cluster as the developer user with the developer password. Create the packaged-review and packaged-review-prod projects.

Log in to the cluster as the developer user with the developer password:
```
$ oc login -u developer -p developer https://api.ocp4.example.com:6443
```

Create the packaged-review project:
```
$ oc new-project packaged-review

Now using project "packaged-review" on server ...
...output omitted...
```

Create the packaged-review-prod project:
```
$ oc new-project packaged-review-prod

Now using project "packaged-review-prod" on server ...
...output omitted...
```

### (2) Add the classroom Helm repository at the http://helm.ocp4.example.com/charts URL and examine its contents.

Use the helm repo list command to list the repositories that are configured for the student user:
```
$ helm repo list

NAME        URL
do280-repo  http://helm.ocp4.example.com/charts
```

If the do280-repo repository is present, then continue to the next step. Otherwise, add the repository.
```
$ helm repo add do280-repo http://helm.ocp4.example.com/charts

"do280-repo" has been added to your repositories
```

Use the helm search command to list all the chart versions in the repository.

The etherpad chart has versions 0.0.6 and 0.0.7. This chart is a copy of a chart from the https://github.com/redhat-cop/helm-charts repository.
```
$ helm search repo --versions

NAME                     CHART VERSION  APP VERSION  DESCRIPTION
do280-repo/etherpad      0.0.7          latest       ...
do280-repo/etherpad      0.0.6          latest       ...
...output omitted...
```

### (3) Install the 0.0.6 version of the etherpad chart on the packaged-review namespace, with the test release name. Use the registry.ocp4.example.com:8443/etherpad/etherpad:1.8.17 image in the offline classroom registry.

Create a values-test.yaml file with the image repository, name, and tag:


|Field	          |Value                                   |
|---------------- |-------                                 |
|image.repository |registry.ocp4.example.com:8443/etherpad |
|image.name       |etherpad                                |
|image.tag        |1.8.17                                  |


Switch to the packaged-review project
```
$ oc project packaged-review

Now using project "packaged-review" on server ...
```

Examine the values of the chart.

You can configure the image, the deployment resources, and other values. By default, the chart creates a route.

```
$ helm show values do280-repo/etherpad --version 0.0.6

# Default values for etherpad.
replicaCount: 1

defaultTitle: "Labs Etherpad"
defaultText: "Assign yourself a user and share your ideas!"

image:
  repository: etherpad 
  name: 
  tag: 
  pullPolicy: IfNotPresent

...output omitted...

route:
  enabled: true
  host: null 
  targetPort: http

...output omitted...

resources: {}
...output omitted...
```

With the default configuration, the chart uses the docker.io/etherpad/etherpad:latest container image.

This image is not suitable for the classroom environment. Use the registry.ocp4.example.com:8443/etherpad/etherpad:1.8.17 container image instead.

Create a values-test.yaml file with the following content:

```
image:
  repository: registry.ocp4.example.com:8443/etherpad
  name: etherpad
  tag: 1.8.17
```

Install the etherpad chart in the packaged-review namespace.

* Use the values-test.yaml file that you created in the previous step.
* Use test as the release name.
```
$ helm install test do280-repo/etherpad -f values-test.yaml --version 0.0.6

... would violate PodSecurity "restricted:v1.24": ...output omitted...
NAME: test
LAST DEPLOYED: Fri Jun 30 01:03:42 2023
NAMESPACE: packaged-review
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Use the helm list command to verify the installed version of the etherpad chart:
```
$ helm list

NAME  NAMESPACE        REVISION  ...  STATUS    CHART           APP VERSION
test  packaged-review  1         ...  deployed  etherpad-0.0.6  latest
```

Verify that the pod is running and that the deployment is ready:

```
$ oc get deployments,pods

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/test-etherpad   1/1     1            1           27s

NAME                                READY   STATUS    RESTARTS   AGE
pod/test-etherpad-c6657b556-4jh8z   1/1     Running   0          27s
```

Verify that the pod executes the specified container image:
```
$ oc describe pods -n packaged-review | egrep '^Name:|Image:'

Name:             test-etherpad-c6657b556-4jh8z
    Image:          registry.ocp4.example.com:8443/etherpad/etherpad:1.8.17
```

Get the route to obtain the application URL.
```
$ oc get routes

NAME           HOST/PORT                                            ...
test-etherpad  test-etherpad-packaged-review.apps.ocp4.example.com  ...
```

Open a web browser and navigate to the following URL to view the application page:
```
https://test-etherpad-packaged-review.apps.ocp4.example.com
```

### (4) Upgrade the etherpad application in the packaged-review namespace to the 0.0.7 version of the chart. Set the image tag for the deployment in the values-test.yaml file.

|Field	    |Value |
|---------- |----- |
|image.tag	|1.8.18 |

Edit the values-test.yaml file and update the image tag value:

```
image:
  repository: registry.ocp4.example.com:8443/etherpad
  name: etherpad
  tag: 1.8.18
```

Use the helm search command to verify that the repository contains a more recent version of the etherpad chart:
```
$ helm search repo --versions etherpad

NAME                 CHART VERSION  APP VERSION  DESCRIPTION
do280-repo/etherpad  0.0.7          latest       ...
do280-repo/etherpad  0.0.6          latest       ...
```

Use the helm upgrade command to upgrade to the latest version of the chart:
```
$ helm upgrade test do280-repo/etherpad -f values-test.yaml --version 0.0.7

... would violate PodSecurity "restricted:v1.24": ...output omitted...
Release "test" has been upgraded. Happy Helming!
NAME: test
LAST DEPLOYED: Fri Jun 30 01:05:07 2023
NAMESPACE: packaged-review
STATUS: deployed
REVISION: 2
TEST SUITE: None
```

Use the helm list command to verify the installed version of the etherpad chart:
```
$ helm list

NAME  NAMESPACE        REVISION  ...  STATUS    CHART           APP VERSION
test  packaged-review  2         ...  deployed  etherpad-0.0.7  latest
```

Verify that the pod is running and that the deployment is ready:
```
$ oc get deployments,pods

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/test-etherpad   1/1     1            1           3m31s

NAME                                 READY   STATUS    RESTARTS   AGE
pod/test-etherpad-59d775b78f-ftmsz   1/1     Running   0          64s
```

Verify that the pod executes the updated container image:
```
$ oc describe pods -n packaged-review | egrep '^Name:|Image:'

Name:             test-etherpad-59d775b78f-ftmsz
    Image:          registry.ocp4.example.com:8443/etherpad/etherpad:1.8.18
```

Reload the test-etherpad application welcome page in the web browser.

### (5) Create a second deployment of the chart in the packaged-review-prod namespace, with the prod release name. Copy the values-test.yaml file to the values-prod.yaml file, and set the route host.

|Field	|Value |
|---------- | ---------------------------- |
|route.host	|etherpad.apps.ocp4.example.com |

Access the application in the route URL to verify that it is working correctly:
```
https://etherpad.apps.ocp4.example.com
```

Switch to the packaged-review-prod project:
```
$ oc project packaged-review-prod
Now using project "packaged-review-prod" on server ...
```

Copy the values-test.yaml file to values-prod.yaml:
```
$ cp values-test.yaml values-prod.yaml
```

Set the route host in the values-prod.yaml file.
```
image:
  repository: registry.ocp4.example.com:8443/etherpad
  name: etherpad
  tag: 1.8.18
route:
  host: etherpad.apps.ocp4.example.com
```

Install the 0.0.6 version of the etherpad chart on the packaged-review-prod namespace.

* Use the values-prod.yaml file that you edited in the previous step.
* Use prod as the release name.

```
$ helm install prod do280-repo/etherpad -f values-prod.yaml --version 0.0.6

... would violate PodSecurity "restricted:v1.24": ...output omitted...
NAME: prod
LAST DEPLOYED: Fri Jun 30 01:07:29 2023
NAMESPACE: packaged-review-prod
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Use the helm list command to verify the installed version of the etherpad chart:
```
$ helm list

NAME  NAMESPACE             REVISION  ...  STATUS    CHART           APP VERSION
prod  packaged-review-prod  1         ...  deployed  etherpad-0.0.6  latest
```

Verify that the pod is running and that the deployment is ready:
```
$ oc get deployments,pods

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prod-etherpad   1/1     1            1           65s

NAME                                 READY   STATUS    RESTARTS   AGE
pod/prod-etherpad-5947dfb987-9dclr   1/1     Running   0          65s
```

Verify that the pod executes the specified container image:
```
$ oc describe pods -n packaged-review-prod | egrep '^Name:|Image:'

Name:             pod/prod-etherpad-5947dfb987-9dclr
    Image:          registry.ocp4.example.com:8443/etherpad/etherpad:1.8.18
```

Verify the deployment by opening a web browser and navigating to the application URL. This URL corresponds to the host that you specified in the values-prod.yaml file. The application welcome page appears in the production URL.
```
https://etherpad.apps.ocp4.example.com
```

### (6) Add limits to the etherpad instance in the packaged-review-prod namespace. The chart values example contains comments that show the required format for this change. Set limits and requests for the deployment in the values-prod.yaml file.

|Field	                   |Value |
|------------------------- |-------------- |
|resources.limits.memory   |128Mi |
|resources.requests.memory |128Mi |

Edit the values-prod.yaml file. Configure the deployment to request 128 MiB of RAM, and limit RAM usage to 128 MiB:
```
image:
  repository: registry.ocp4.example.com:8443/etherpad
  name: etherpad
  tag: 1.8.18
route:
  host: etherpad.apps.ocp4.example.com
resources:
  limits:
     memory: 128Mi
  requests:
     memory: 128Mi
```

Use the helm upgrade command to upgrade to the latest version of the chart:
```
$ helm upgrade prod do280-repo/etherpad -f values-prod.yaml --version 0.0.7

... would violate PodSecurity "restricted:v1.24": ...output omitted...
Release "prod" has been upgraded. Happy Helming!
NAME: prod
LAST DEPLOYED: Fri Jun 30 01:09:04 2023
NAMESPACE: packaged-review-prod
STATUS: deployed
REVISION: 2
TEST SUITE: None
```

Verify that the pod is running and that the deployment is ready:
```
$ oc get deployments,pods

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prod-etherpad   1/1     1            1           3m14s

NAME                                 READY   STATUS    RESTARTS   AGE
pod/prod-etherpad-6b7d9dffbc-f7cng   1/1     Running   0          36s
```

Examine the application pod from the production instance of the application to verify the configuration change:
```
$ oc describe pods -n packaged-review-prod | egrep -A1 '^Name:|Limits|Requests'

Name:             prod-etherpad-6b7d9dffbc-f7cng
Namespace:        packaged-review-prod
​--
    Limits:
      memory:  128Mi
    Requests:
      memory:   128Mi
```

Examine the pod of the test instance of the application in the packaged-review namespace. This deployment uses the values 
from the values-test.yaml file that did not specify resource limits or requests. The pod in the packaged-review namespace 
does not have a custom resource allocation.
```
$ oc describe pods -n packaged-review | egrep -A1 '^Name:|Limits|Requests'

Name:             test-etherpad-59d775b78f-ftmsz
Namespace:        packaged-review
```

Use the helm list command to verify the installed version of the etherpad chart:
```
$ helm list

NAME  NAMESPACE             REVISION  ...  STATUS    CHART           APP VERSION
prod  packaged-review-prod  2         ...  deployed  etherpad-0.0.7  latest
```

Reload the application welcome page in the web browser. The deployment continues working after you add the limits.

Remove the values-test.yaml and values-prod.yaml files:
```
$ rm values-test.yaml values-prod.yaml
```

