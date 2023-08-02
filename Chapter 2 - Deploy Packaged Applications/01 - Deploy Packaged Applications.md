# Deploy Packaged Applications

## Goal

Deploy and update applications from resource manifests that are packaged for sharing and distribution.

## Objectives

* Deploy an application and its dependencies from resource manifests that are stored in an OpenShift template.
* Deploy and update applications from resource manifests that are packaged as Helm charts.

## Sections

* OpenShift Templates
* Helm Charts

## Summary
* Use templates to deploy workloads with parameterization.
* Use the *oc create -f* command to upload a template to a project.
* Use the *oc process command* and the *oc apply -f -* command to deploy template resources to the Kubernetes cluster.
* Provide parameters to customize the template with the *-p* or *--param-file* arguments to the oc command.
* View Helm charts with the *helm show chart <chart-reference>* and *helm show values <chart-reference>* commands.
* Use the *helm install release-name <chart-reference>* command to create a release for a chart.
* Inspect releases by using the *helm list* command.
* Use the *helm history <release-name>* command to view the history of a release.
* Use the *helm repo add <repo-name> <repo-url>* command to add a Helm repository to the ~/.config/helm/repositories.yaml configuration file.
* Use the *helm search repo* command to search repositories in the ~/.config/helm/repositories.yaml configuration file.

# OpenShift Templates

## Objectives

Deploy and update applications from resource manifests that are packaged as OpenShift templates.

### Discovering Templates
The templates that the Cluster Samples Operator provides are in the openshift namespace. Use the following *oc get* command 
to view a list of these templates:

```
$ oc get templates -n openshift

NAME                      DESCRIPTION            PARAMETERS        OBJECTS
cache-service             Red Hat Data Grid...    8 (1 blank)      4
cakephp-mysql-example     An example CakePHP...  21 (4 blank)      8
cakephp-mysql-persistent  An example CakePHP...  22 (4 blank)      9
...output omitted...
```

To evaluate any template, use the *oc describe template <template-name> -n openshift* command to view more details about the 
template, including the description, the labels that the template uses, the template parameters, and the resources that 
the template generates.

The following example shows the details of the cache-service template:

```
$ oc describe template cache-service -n openshift

Name: cache-service
Namespace: openshift
Created: 2 months ago
Labels: samples.operator.openshift.io/managed=true
template=cache-service
Description: Red Hat Data Grid is an in-memory, distributed key/value store.
Annotations: iconClass=icon-datagrid
...output omitted...

Parameters:
    Name: APPLICATION_NAME
    Display Name: Application Name
    Description: Specifies a name for the application.
    Required: true
    Value: cache-service

 ...output omitted...

    Name: APPLICATION_PASSWORD
    Display Name: Client Password
    Description: Sets a password to authenticate client applications.
    Required: false
    Generated: expression
    From: [a-zA-Z0-9]{16}

Object Labels: template=cache-service

Message: <none>

Objects:
    Secret ${APPLICATION_NAME}
    Service ${APPLICATION_NAME}-ping
    Service ${APPLICATION_NAME}
    StatefulSet.apps ${APPLICATION_NAME}
```

In addition to using the *oc describe* command to view information about a template, the *oc process* command provides a --parameters 
option to view only the parameters that a template uses. 

For example, use the following command to view the parameters that 
the cache-service template uses:

```
$ oc process --parameters cache-service -n openshift

NAME                   ...   GENERATOR    VALUE
APPLICATION_NAME       ...                cache-service
IMAGE                  ...                registry.redhat.io/jboss-datagrid-7/...
NUMBER_OF_INSTANCES    ...                1
REPLICATION_FACTOR     ...                1
EVICTION_POLICY        ...                evict
TOTAL_CONTAINER_MEM    ...                512
APPLICATION_USER       ...
APPLICATION_PASSWORD   ...   expression   [a-zA-Z0-9]{16}
```

Use the -f option to view the parameters of a template that are defined in a file:

```
$ oc process --parameters -f  my-cache-service.yaml
```

Use the *oc get template <template-name> -o yaml -n namespace* command to view the manifest for the template. 

The following example retrieves the template manifest for the cache-service template:

```
$ oc get template cache-service -o yaml -n openshift

apiVersion: template.openshift.io/v1
kind: Template
labels:
  template: cache-service
metadata:
 ...output omitted...
- apiVersion: v1
  kind: Secret
  metadata:
 ...output omitted...
- apiVersion: v1
  kind: Service
  metadata:
 ...output omitted...
- apiVersion: v1
  kind: Service
  metadata:
 ...output omitted...
- apiVersion: apps/v1
  kind: StatefulSet
  metadata:
 ...output omitted...
parameters:
- description: Specifies a name for the application.
  displayName: Application Name
  name: APPLICATION_NAME
  required: true
  value: cache-service
- description: Sets an image to bootstrap the service.
  name: IMAGE
 ...output omitted...
```

### Using Templates

The *oc new-app* command has a --template option that can deploy the template resources directly from the openshift project.
 
The following example deploys the resources that are defined in the cache-service template from the openshift project:

```
$ oc new-app --template=cache-service -p APPLICATION_USER=my-user
```

The *oc new-app* command can only create new resources, not update existing resources.

You can use the *oc process* command to apply parameters to a template, to produce manifests to deploy the templates with 
a set of parameters. 

The *oc process* command can process both templates that are stored in files locally, and templates that are stored in the cluster. 

However, to process templates in a namespace, you must have write permissions on the template namespace.

### Deploying Applications from Templates

The *oc process* command uses parameter values to transform a template into a set of related Kubernetes resource manifests. 

For example, the following command creates a set of resource manifests for the my-cache-service template. When you use the 
-o yaml option, the resulting manifests are in the YAML format. The example writes the manifests to a my-cache-service-manifest.yaml file:

```
$ oc process my-cache-service -p APPLICATION_USER=user1 -o yaml > my-cache-service-manifest.yaml
```

The previous example uses the -p option to provide a parameter value to the only required parameter without a default value.

Use the -f option with the oc process command to process a template that is defined in a file:

```
$ oc process -f my-cache-service.yaml -p APPLICATION_USER=user1 -o yaml > my-cache-service-manifest.yaml
```

Use the -p option with key=value pairs with the oc process command to use parameter values that override the default values. 

The following example passes three parameter values to the my-cache-service template, and overrides the default values of 
the specified parameters:

```
$ oc process my-cache-service -o yaml \
  -p TOTAL_CONTAINER_MEM=1024 \
  -p APPLICATION_USER='cache-user' \
  -p APPLICATION_PASSWORD='my-secret-password' \
  > my-cache-service-manifest.yaml
```

Instead of specifying parameters on the command line, place the parameters in a file. This option cleans up the command 
line when many parameter values are required. Save the parameters file in a version control system to keep records of the 
parameters that are used in production deployments.

For example, instead of using the command-line options in the previous examples, place the key-value pairs in a *my-cache-service-params.env* file:

```
TOTAL_CONTAINER_MEM=1024
APPLICATION_USER='cache-user'
APPLICATION_PASSWORD='my-secret-password'
```

The corresponding oc process command uses the --param-file option to pass the parameters as follows:

```
$ oc process my-cache-service -o yaml --param-file=my-cache-service-params.env > my-cache-service-manifest.yaml
```

Generating a manifest file is not required to use templates. Instead, pipe the output of the oc process command directly 
to the input for the *oc apply -f -* command. The oc apply command creates live resources on the Kubernetes cluster.

```
$ oc process my-cache-service --param-file=my-cache-service-params.env | oc apply -f -
```

### Updating Apps from Templates

Because you use the oc apply command, after deploying a set of manifests from a template, you can process the template again 
and use oc apply for updates.

To compare the results of applying a different parameters file to a template against the live resources, pipe the manifest 
to the *oc diff -f -* command. 

For example, given a second parameter file named my-cache-service-params-2.env, use the following command:

```
$ oc process my-cache-service -o yaml --param-file=my-cache-service-params-2.env | oc diff -f -

...output omitted...
-  generation: 1
+  generation: 2
   labels:
     application: cache-service
     template: cache-service
@@ -86,10 +86,10 @@
           timeoutSeconds: 10
         resources:
           limits:
-            memory: 1Gi
+            memory: 2Gi
           requests:
             cpu: 500m
-            memory: 1Gi
+            memory: 2Gi
         terminationMessagePath: /dev/termination-log
         terminationMessagePolicy: File
         volumeMounts:
```

After verifying that the changes are what you intend, you can pipe the output of the oc process to the *oc apply -f -* command.

### Managing Templates

For production usage, make a customized copy of the template, to change the default values of the template to suitable values 
for the target project. 

To copy a template into your project, use the *oc get template* command with the -o yaml option to copy the template YAML to a file.

The following example copies the cache-service template from the openshift project to a YAML file named my-cache-service.yaml:

```
$ oc get template cache-service -o yaml -n openshift > my-cache-service.yaml
```

After creating a YAML file for a template, consider making the following changes to the template:

* Give the template a new name that is specific to the target use of the template resources.
* Apply appropriate changes to the parameter default values at the end of the file.
* Remove the namespace field of the template resource.

After you have a YAML file for a template, use the *oc create -f* command to upload the template to the current project.
 
In this case, the *oc create* command is not creating the resources that the template defines. Instead, the command is creating 
a template resource in the project. 

Using a template that is uploaded to a project clarifies which template provides the 
resource definitions of a project. 

The following example uploads a customized template that is defined in the my-cache-service.yaml file to the current project:

```
$ oc create -f my-cache-service.yaml
```

Use the -n namespace option to upload the template to a different project. The following example uploads the template that 
is defined in the my-cache-service.yaml file to the shared-templates project:

```
$ oc create -f my-cache-service.yaml -n shared-templates
```

Use the oc get templates command to view a list of available templates in the project:

```
$ oc get templates -n shared-templates

NAME                      DESCRIPTION            PARAMETERS        OBJECTS
my-cache-service          Red Hat Data Grid...    8 (1 blank)      4
```
