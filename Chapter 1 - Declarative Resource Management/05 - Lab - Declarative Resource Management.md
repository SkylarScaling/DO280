# Lab: Declarative Resource Management

* Deploy and update applications from resource manifests that are parameterized for different target environments.

## Outcomes

* Deploy an application by using Kustomize from provided files.
* Apply an application update that changes a deployment.
* Deploy an overlay of the application that increases the number of replicas.

## Instructions

(1) Clone the v1.1.0 version of the application from the https://git.ocp4.example.com/developer/declarative-review.git URL:

```
$ git clone https://git.ocp4.example.com/developer/declarative-review.git --branch v1.1.0

Cloning into 'declarative-review'...
...output omitted...
```

(2) Deploy the base directory of the repository to a new declarative-review project. Verify that the v1.1.0 version of the 
application is available at http://exoplanets-declarative-review.apps.ocp4.example.com:

Log in to the OpenShift cluster as the developer user with the developer password:

```
$ oc login -u developer -p developer https://api.ocp4.example.com:6443

Login successful.

...output omitted...
```

Create the declarative-review project:

```
$ oc new-project declarative-review

...output omitted...
```

Use the oc apply -k command to deploy the application with Kustomize:

```
$ oc apply -k base

configmap/database created
configmap/exoplanets created
secret/database created
secret/exoplanets created
service/database created
service/exoplanets created
deployment.apps/database created
deployment.apps/exoplanets created
route.route.openshift.io/exoplanets created
```

Use the watch command to wait until the workloads are running:

```
$ watch oc get all

NAME                             READY   STATUS    RESTARTS      AGE
pod/database-55d6c77787-47649    1/1     Running   0             57s
pod/exoplanets-d6f57869d-jhkhc   1/1     Running   2 (54s ago)   57s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/database     ClusterIP   172.30.236.123   <none>        5432/TCP   57s
service/exoplanets   ClusterIP   172.30.248.130   <none>        8080/TCP   57s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/database     1/1     1            1           57s
deployment.apps/exoplanets   1/1     1            1           57s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/database-55d6c77787    1         1         1       57s
replicaset.apps/exoplanets-d6f57869d   1         1         1       57s

NAME                  HOST/PORT                                             ...
route.../exoplanets   exoplanets-declarative-review.apps.ocp4.example.com   ...
```

Press Ctrl+C to exit the watch command.

Open a web browser and navigate to http://exoplanets-declarative-review.apps.ocp4.example.com.

```
The browser displays version v1.1.0 of the application.
```

(3) Deploy the updated application and verify that the URL now displays the v1.1.1 version.

Change to the v1.1.1 branch:

```
$ git checkout v1.1.1

Branch 'v1.1.1' set up to track remote branch 'v1.1.1' from 'origin'.
Switched to a new branch 'v1.1.1'
```

Deploy the updated application and verify that the URL now displays the v1.1.1 version.

Use the oc apply -k command to execute the changes:

```
$ oc apply -k base

...output omitted...
```

Use the watch command to wait until the application redeploys:

```
$ watch oc get all

NAME                             READY   STATUS    RESTARTS      AGE
pod/database-55d6c77787-47649    1/1     Running   0             57s
pod/exoplanets-d6f57869d-jhkhc   1/1     Running   2 (54s ago)   57s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/database     ClusterIP   172.30.236.123   <none>        5432/TCP   57s
service/exoplanets   ClusterIP   172.30.248.130   <none>        8080/TCP   57s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/database     1/1     1            1           57s
deployment.apps/exoplanets   1/1     1            1           57s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/database-55d6c77787    1         1         1       57s
replicaset.apps/exoplanets-d6f57869d   1         1         1       57s

NAME                  HOST/PORT                                             ...
route.../exoplanets   exoplanets-declarative-review.apps.ocp4.example.com   ...
```

Press Ctrl+C to exit the watch command.

Open a web browser and navigate to http://exoplanets-declarative-review.apps.ocp4.example.com:

```
The browser displays version v1.1.1 of the application.
```

(4) Examine the overlay in the overlays/production path.

Examine the overlays/production/kustomization.yaml file:

```
$ cat overlays/production/kustomization.yaml

kind: Kustomization
bases:
- ../../base/
patches:
- path: patch-replicas.yaml
  target:
    kind: Deployment
    name: exoplanets
```

This overlay applies a patch over the base.

Examine the overlays/production/patch-replicas.yaml file:

```
$ cat overlays/production/patch-replicas.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: exoplanets
spec:
  replicas: 2
```

This patch increases the number of replicas of the deployment, so that the production deployment can handle more users.

(5) Deploy the production overlay to a new declarative-review-production project. Verify that the v1.1.1 version of the 
application is available at http://exoplanets-declarative-review-production.apps.ocp4.example.com with two replicas.

Create the declarative-review-production project:

```
$ oc new-project declarative-review-production

Now using project "declarative-review-production" on server "https://api.ocp4.example.com:6443".
...output omitted...
```

Use the oc apply -k command to deploy the overlay:

```
$ oc apply -k overlays/production

configmap/database created
configmap/exoplanets created
secret/database created
secret/exoplanets created
service/database created
service/exoplanets created
deployment.apps/database created
deployment.apps/exoplanets created
route.route.openshift.io/exoplanets created
```

Use the watch command to wait until the workloads are running:

```
$ watch oc get all

NAME                              READY   STATUS    RESTARTS       AGE
pod/database-55d6c77787-b5x4n     1/1     Running   0              5m11s
pod/exoplanets-55666f556f-ndwkz   1/1     Running   2 (5m8s ago)   5m11s
pod/exoplanets-55666f556f-q7s7j   1/1     Running   2 (5m7s ago)   5m11s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/database     ClusterIP   172.30.24.165   <none>        5432/TCP   5m11s
service/exoplanets   ClusterIP   172.30.90.176   <none>        8080/TCP   5m11s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/database     1/1     1            1           5m11s
deployment.apps/exoplanets   2/2     2            2           5m11s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/database-55d6c77787     1         1         1       5m11s
replicaset.apps/exoplanets-55666f556f   2         2         2       5m11s

NAME                HOST/PORT
route.../exoplanets exoplanets-declarative-review-production.apps.ocp4.example.com
```

The exoplanets deployment has two replicas.

Open a web browser and navigate to http://exoplanets-declarative-review-production.apps.ocp4.example.com:

```
The browser displays version v1.1.1 of the application.
```