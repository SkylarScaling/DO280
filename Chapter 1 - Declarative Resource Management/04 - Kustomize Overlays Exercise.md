# Guided Exercise: Kustomize Overlays

* Deploy and update an application by applying different Kustomize overlays that are stored in a Git server.

## Outcomes

* Deploy an application by using Kustomize from provided files.
* Apply an application update that changes a deployment.
* Deploy an overlay of the application that increases the number of replicas.
* As the student user on the workstation machine, use the lab command to prepare your system for this exercise.

## Instructions

(1) Use the tree command to review the structure of the repository:

```
$ tree
.
├── base
│   ├── database 
│   │   ├── configmap.yaml
│   │   ├── deployment.yaml
│   │   ├── kustomization.yaml
│   │   └── service.yaml
│   ├── exoplanets 
│   │   ├── deployment.yaml
│   │   ├── kustomization.yaml
│   │   ├── route.yaml
│   │   └── service.yaml
│   └── kustomization.yaml 
└── README.md
```

Examine the base/kustomization.yaml file:

```
$ cat base/kustomization.yaml
kind: Kustomization
bases: 
- database
- exoplanets
secretGenerator: 
- name: db-secrets
  literals:
    - DB_ADMIN_PASSWORD=postgres
    - DB_NAME=database
    - DB_PASSWORD=password
    - DB_USER=user
configMapGenerator: 
- name: db-config
  literals:
    - DB_HOST=database
    - DB_PORT=5432
```

(2) Deploy the base directory of the repository to a new declarative-kustomize project. Verify that the v1.1.0 version of 
the application is available at http://exoplanets-declarative-kustomize.apps.ocp4.example.com.

Log in to the OpenShift cluster as the developer user with the developer password:

```
$ oc login -u developer -p developer https://api.ocp4.example.com:6443

Login successful.
```

Create the declarative-kustomize project:

```
$ oc new-project declarative-kustomize

...output omitted...
```

Use the oc apply -k command to deploy the application with Kustomize:

```
$ oc apply -k base

configmap/database created
configmap/db-config-2d7thbcgkc created
secret/db-secrets-55cbgc8c6m created
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

NAME
  HOST/PORT                                                ...
route.route.openshift.io/exoplanets
  exoplanets-declarative-kustomize.apps.ocp4.example.com   ...
```

Press Ctrl+C to exit the watch command.

Open a web browser and navigate to http://exoplanets-declarative-kustomize.apps.ocp4.example.com.

(3) Deploy the updated application and verify that the URL now displays the v1.1.1 version.

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

NAME
  HOST/PORT                                                ...
route.route.openshift.io/exoplanets
  exoplanets-declarative-kustomize.apps.ocp4.example.com   ...
```

Press Ctrl+C to exit the watch command.

Open a web browser and navigate to http://exoplanets-declarative-kustomize.apps.ocp4.example.com.

(4) The v1.1.2 version updates the base kustomization. This update changes the password that the database uses. This change 
is possible because the sample application re-creates the database on startup.

List the secrets in the namespace:

```
$ oc get secret

NAME                       TYPE                                  DATA   AGE
builder-dockercfg-qwn4v    kubernetes.io/dockercfg               1      4m31s
builder-token-z754n        kubernetes.io/service-account-token   4      4m31s
db-secrets-55cbgc8c6m      Opaque                                4      4m28s
default-dockercfg-w4v89    kubernetes.io/dockercfg               1      4m31s
default-token-zw89c        kubernetes.io/service-account-token   4      4m31s
deployer-dockercfg-l8sct   kubernetes.io/dockercfg               1      4m31s
deployer-token-knvhb       kubernetes.io/service-account-token   4      4m31s
```

When creating a secret, Kustomize appends a hash to the secret name.

Extract the contents of the secret. The name of the secret can change in your environment. Use the output from a previous 
step to learn the name of the secret:

```
$ oc extract secret/db-secrets-55cbgc8c6m --to=-

# DB_PASSWORD
password
# DB_USER
user
# DB_ADMIN_PASSWORD
postgres
# DB_NAME
database
```

(5) Deploy the updated application.

Use the oc apply -k command to execute the changes:

```
$ oc apply -k base

configmap/database unchanged
configmap/db-config-2d7thbcgkc unchanged
secret/db-secrets-6h668tk789 created
service/database unchanged
service/exoplanets unchanged
deployment.apps/database configured
deployment.apps/exoplanets configured
route.route.openshift.io/exoplanets configured
```

Because the password is different, Kustomize creates another secret. Kustomize also updates the two deployments that use 
the secret to use the new secret.

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

NAME
  HOST/PORT                                                ...
route.route.openshift.io/exoplanets
  exoplanets-declarative-kustomize.apps.ocp4.example.com   ...
```

Press Ctrl+C to exit the watch command.

Open a web browser and navigate to http://exoplanets-declarative-kustomize.apps.ocp4.example.com.

The browser continues showing the v1.1.1 version of the application.

Examine the deployment:

```
$ oc get deployment exoplanets -o jsonpath='{.spec.template.spec.containers[0].envFrom}'

[{"configMapRef":{"name":"db-config-2d7thbcgkc"}},{"secretRef":{"name":"db-secrets-6h668tk789"}}]
```

The deployment uses the new secret.

Examine the secret. Use the name of the secret from a previous step:

```
$ oc extract secret/db-secrets-6h668tk789 --to=-

# DB_ADMIN_PASSWORD
postgres
# DB_NAME
database
# DB_PASSWORD
newpassword
# DB_USER
user
```

The deployment uses the changed password.

(6) The v1.1.3 version adds a production overlay that increases the number of replicas.

Deploy the updated application and verify the number of replicas.

Use the oc apply -k command to execute the changes:

```
$ oc apply -k overlays/production

...output omitted...
```

Use the watch command to wait until the application redeploys:

```
$ watch oc get all

NAME                             READY   STATUS    RESTARTS      AGE
pod/database-7dfb559cf7-rvxhx    1/1     Running   0             11m
pod/exoplanets-957bb5b48-5xl2d   1/1     Running   2 (11m ago)   11m
pod/exoplanets-957bb5b48-mgbrx   1/1     Running   0             19s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/database     ClusterIP   172.30.87.214   <none>        5432/TCP   19m
service/exoplanets   ClusterIP   172.30.25.65    <none>        8080/TCP   19m

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/database     1/1     1            1           19m
deployment.apps/exoplanets   2/2     2            2           19m

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/database-7dfb559cf7     1         1         1       11m
replicaset.apps/database-d4cd8dcc       0         0         0       19m
replicaset.apps/exoplanets-6c7b4bb44c   0         0         0       19m
replicaset.apps/exoplanets-7ccb754c8b   0         0         0       18m
replicaset.apps/exoplanets-957bb5b48    2         2         2       11m

NAME
  HOST/PORT                                                ...
route.route.openshift.io/exoplanets
  exoplanets-declarative-kustomize.apps.ocp4.example.com   ...
```

Press Ctrl+C to exit the watch command. After you run the command, the application has two replicas.

(7) Delete the application.

Use the oc delete -k command to delete the resources that Kustomize manages:

```
]$ oc delete -k base

configmap "database" deleted
configmap "db-config-2d7thbcgkc" deleted
secret "db-secrets-h9hdmt2g79" deleted
service "database" deleted
service "exoplanets" deleted
deployment.apps "database" deleted
deployment.apps "exoplanets" deleted
route.route.openshift.io "exoplanets" deleted
```
