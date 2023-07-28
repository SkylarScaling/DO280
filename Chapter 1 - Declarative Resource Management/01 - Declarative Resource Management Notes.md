# Chapter 1 - Declarative Resource Management

## Goal: 

Deploy and update applications from resource manifests that are parameterized for different target environments

## Objectives

* Deploy and update applications from resource manifests stored as YAML files
* Deploy and update applications from resource manifests augmented by Kustomize

## Sections
* Resource Manifests
* Kustomize Overlays

## Summary
* Imperative commands perform actions, such as creating a deployment, by specifying all necessary parameters as command-line arguments.
* In the declarative workflow, you create manifests that describe resources in the YAML or JSON formats, and use commands 
such as kubectl apply to deploy the resources to a cluster.
* Kubernetes provides tools, such as the kubectl diff command, to review your changes before applying them.


* You can use Kustomize to create multiple deployments from a single base code with different customizations.
* The kubectl command integrates Kustomize into the apply subcommand and others.
* Kustomize organizes content around bases and overlays.
* Bases and overlays can create and modify existing resources from other bases and overlays.

# Kubernetes Manifests

## Creating Resources

To create a starter deployment manifest, use the kubectl create deployment command to generate a manifest by using the 
--dry-run=client option:

```
$ kubectl create deployment hello-openshift -o yaml \  
  --image registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:v1.0 \  
  --save-config \  
  --dry-run=client \  
  > ~/my-app/example-deployment.yaml
```

Example resource:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: resource-manifests
  labels:
    app: hello-openshift
  name: hello-openshift
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-openshift
  template:
    metadata:
      labels:
        app: hello-openshift
    spec:
      containers:
        - image: quay.io/redhattraining/hello-world-nginx:v1.0
          name: hello-world-nginx
          ports:
            - containerPort: 8080
              protocol: TCP
```

To create a resource, use the kubectl create -f resource.yaml command.

Instead of a file name, you can pass a directory to the command to process all the resource files in a directory.   
Add the --recursive=true or -R option to recursively process resource files that are provided in multiple subdirectories.

```
[user@host ~]$ tree my-app
my-app
├── example_deployment.yaml
└── service
    └── example_service.yaml

[user@host ~]$ kubectl create -R -f ~/my-app
deployment.apps/hello-openshift created
service/hello-openshift created
```

## Updating Resources

The kubectl apply command can also create resources with the same -f option that is illustrated with the kubectl create command. 
However, the kubectl apply command can also update a resource.

Although the kubectl create -f command can create resources from a manifest, the command is imperative and thus does not 
account for the current state of a live resource. Executing kubectl create -f against a manifest for a live resource gives an error.

In contrast, the kubectl apply -f command is declarative, and considers the difference between the current resource state 
in the cluster and the intended resource state that is expressed in the manifest.

## YAML Validation

Before applying the changes to the resource, use the --dry-run=server and the --validate=true flags to inspect the file for errors.

* The --dry-run=server option submits a server-side request without persisting the resource.
* The --validate=true option uses a schema to validate the input and fails the request if it is invalid.

Any syntax errors in the YAML are included in the output. Most importantly, the --dry-run=server option prevents applying 
any changes to the Kubernetes runtime.

```
[user@host ~]$ kubectl apply -f ~/my-app/example-deployment.yaml \
  --dry-run=server --validate=true

...output omitted...
deployment.apps/hello-openshift created (server dry-run) 
```

## Comparing Resources

Use the kubectl diff command to review differences between live objects and manifests.

```
[user@host ~]$ kubectl diff -f example-deployment.yaml
[student@workstation ~]$ kubectl diff -f my-app/example-deployment.yaml
...output omitted...
diff -u -N /tmp/LIVE-2647853521/apps.v1.Deployment.resource...
--- /tmp/LIVE-2647853521/apps.v1.Deployment.resource-manife...
+++ /tmp/MERGED-2640652736/apps.v1.Deployment.resource-mani...
@@ -6,7 +6,7 @@
     kubectl.kubernetes.io/last-applied-configuration: |
       ...output omitted...
   creationTimestamp: "2023-04-27T16:07:47Z"
-  generation: 1
+  generation: 2
   labels:
     app: hello-openshift
   name: hello-openshift
@@ -32,7 +32,7 @@
         app: hello-openshift
     spec:
       containers:
-      - image: registry.ocp4.example.com:8443/.../hello-world-nginx:v1.0
+      - image: registry.ocp4.example.com:8443/.../hello-world-nginx:latest
         imagePullPolicy: IfNotPresent
         name: hello-openshift
         ports:
```

## Update Considerations
When using the oc diff command, recognize when applying a manifest change does not generate new pods. 
For example, if an updated manifest changes only values in secret or a configuration map, then applying the updated manifest 
does not generate new pods that use those values. 

As a solution, use the oc rollout restart command to force a restart of the pods that are associated with the deployment: 

```
oc rollout restart deployment deployment-name 
```

In deployments with a single replica, you can also resolve the problem by deleting the pod. However, for multiple replicas, 
using the oc rollout command to restart the pods is preferred, because the pods are stopped and replaced in a smart manner 
that minimizes downtime.