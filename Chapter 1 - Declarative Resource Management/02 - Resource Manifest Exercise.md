# Guided Exercise: Resource Manifests

Deploy and update an application from resource manifests from YAML files that are stored in a Git server.

## Outcomes

* Deploy applications from resource manifests from YAML files that are stored in a GitLab repository.
* Inspect new manifests for potential update issues.
* Update application deployments from new YAML manifests.
* Force the redeployment of pods when necessary.

## Instructions

(1) Log in to the OpenShift cluster and create the declarative-manifests project:

```
$ oc login -u developer -p developer https://api.ocp4.example.com:6443

Login successful.
```

(2) Create the declarative-manifests project:

```
$ oc new-project declarative-manifests

Now using project "declarative-manifests" on server ...
```

(3) Deploy the resource manifests of the first application version.

Validate the YAML resource manifest for the application:

```
$ oc apply -f . --validate=true --dry-run=server
```

Deploy the exoplanets application:

```
$ oc apply -f .
```

List the deployments and pods. The exoplanets pod can go into a temporary crash loop backoff state if it attempts to access 
the database before it becomes available. Wait for the pods to be ready:   
Press Ctrl+C to exit the watch command.

```
$ watch oc get deployments,pods

Every 2.0s: oc get deployments,pods ...

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/database   1/1     1            1           32s
deployment.apps/exoplanets 1/1     1            1           32s

NAME                            READY   STATUS    RESTARTS   AGE
pod/database-6fddbbf94f-2pghj   1/1     Running   0          32s
pod/exoplanets-64c87f5796-bw8tm 1/1     Running   0          32s
List the route for the exoplanets application.
```

```
$ oc get routes -l app=exoplanets

NAME         HOST/PORT                                                ...
exoplanets   exoplanets-declarative-manifests.apps.ocp4.example.com   ...
Open the route URL in the web browser. The application version is v1.1.0.

http://exoplanets-declarative-manifests.apps.ocp4.example.com/
```

(4) Deploy the second version of the exoplanets application:

```
$ oc diff -f .

...output omitted...
         - secretRef:
             name: exoplanets
-        image: registry.ocp4.example.com:8443/redhattraining/exoplanets:v1.1.0
+        image: registry.ocp4.example.com:8443/redhattraining/exoplanets:v1.1.1
         imagePullPolicy: Always
         livenessProbe:
           failureThreshold: 3
```

The new version changes the image that is deployed to the cluster. Because the change is in the deployment, the new manifest 
produces new pods for the application.

Apply the changes from the new manifests:

```
$ oc apply -f .

configmap/database unchanged
secret/database configured
deployment.apps/database configured
service/database configured
configmap/exoplanets unchanged
secret/exoplanets configured
deployment.apps/exoplanets configured
service/exoplanets unchanged
route.route.openshift.io/exoplanets configured
```

List the deployments and pods. Wait for the application pod to be ready:   
Press Ctrl+C to exit the watch command.

```
$ watch oc get deployments,pods

Every 2.0s: oc get deployments,pods ...

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/database   1/1     1            1           6m32s
deployment.apps/exoplanets 1/1     1            1           6m33s

NAME                            READY   STATUS    RESTARTS   AGE
pod/database-6fddbbf94f-2pghj   1/1     Running   0          6m33s
pod/exoplanets-74c85f5796-tw8tf 1/1     Running   0          32s
```

List the route for the exoplanets application:

```
$ oc get routes -l app=exoplanets

NAME         HOST/PORT                                                ...
exoplanets   exoplanets-declarative-manifests.apps.ocp4.example.com   ...
```

(5) Deploy the third version of the exoplanets application.

View the differences between the currently deployed version of the application and the updated resource manifests:

```
$ oc diff -f .

...output omitted...
 kind: Secret
 metadata:
   annotations:
...output omitted...
-  DB_USER: '*** (before)'
+  DB_USER: '*** (after)'
 kind: Secret
 metadata:
   annotations:
```

Inspect the current application pods:

```
$ oc get pods

NAME                          READY   STATUS    RESTARTS   AGE
database-6fddbbf94f-brlj6     1/1     Running   0          44m
exoplanets-674cc57b5d-mv8kd   1/1     Running   0          18m
```

Deploy the new version of the application:

```
[student@workstation declarative-manifests]$ oc apply -f .

configmap/database unchanged
secret/database configured
deployment.apps/database configured
service/database configured
configmap/exoplanets unchanged
secret/exoplanets configured
deployment.apps/exoplanets unchanged
service/exoplanets unchanged
route.route.openshift.io/exoplanets configured
```

Inspect the current application pods again:

```
$ oc get pods

NAME                          READY   STATUS    RESTARTS   AGE
database-6fddbbf94f-brlj6     1/1     Running   0          23m
exoplanets-674cc57b5d-mv8kd   1/1     Running   0          22m
```

Although the secret is updated, the deployed application pods are not changed. These non-updated pods are a problem, because 
the pods load secrets and configuration maps at startup. Currently, the pods have stale values from the previous configuration, 
and therefore could crash.

Force the exoplanets application to restart, to flush out any stale configuration data.

Use the oc get deployments command to confirm the deployments:

```
$ oc get deployments

NAME         READY   UP-TO-DATE   AVAILABLE   AGE
database     1/1     1            1           32m
exoplanets   1/1     1            1           32m
```

Use the oc rollout command to restart the database deployment:

```
$ oc rollout restart deployment/database

deployment.apps/database restarted
```

Use the oc rollout command to restart the exoplanets deployment:

```
$ oc rollout restart deployment/exoplanets

deployment.apps/exoplanets restarted
```

List the pods. The exoplanets pod can go into a temporary crash loop backoff state if it attempts to access the database 
before it becomes available. Wait for the application pod to be ready:  
Press Ctrl+C to exit the watch command.

```
$ watch oc get pods

Every 2.0s: oc get deployments,pods ...

NAME                          READY   STATUS    RESTARTS   AGEE
database-7c767c4bd7-m72nk     1/1     Running   0          32s
exoplanets-64c87f5796-bw8tm   1/1     Running   0          32s
```

Use the oc get deployment command with the -o yaml option to view the last-applied-configuration annotation:

```
$ oc get deployment exoplanets -o yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "3"
    description: Defines how to deploy the application server
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":...
    template.alpha.openshift.io/wait-for-ready: "true"
...output omitted...
```
