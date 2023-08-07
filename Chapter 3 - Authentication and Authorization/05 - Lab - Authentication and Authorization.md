# Lab: Authentication and Authorization

Configure the HTPasswd identity provider, create groups, and assign roles to users and groups.

## Outcomes

* Create users and passwords for HTPasswd authentication.
* Configure the identity provider for HTPasswd authentication.
* Assign cluster administration rights to users.
* Remove the ability to create projects at the cluster level.
* Create groups and add users to groups.
* Manage user privileges in projects by granting privileges to groups.

## Instructions

### (1) Update the existing ~/DO280/labs/auth-review/tmp_users HTPasswd authentication file to remove the analyst user. Ensure that the tester and leader users in the file use the L@bR3v!ew password. Add two entries to the file for the new_admin and new_developer users. Use the L@bR3v!ew password for each new user.

Remove the analyst user from the ~/DO280/labs/auth-review/tmp_users HTPasswd authentication file:
```
$ htpasswd -D ~/DO280/labs/auth-review/tmp_users analyst

Deleting password for user analyst
```

Update the entries for the tester and leader users to use the L@bR3v!ew password. Add entries for the new_admin and new_developer 
users with the L@bR3v!ew password:
```
$ for NAME in tester leader new_admin new_developer ; \
    do \
    htpasswd -b ~/DO280/labs/auth-review/tmp_users ${NAME} 'L@bR3v!ew' ; \
    done

Updating password for user tester
Updating password for user leader
Adding password for user new_admin
Adding password for user new_developer
```

Review the contents of the ~/DO280/labs/auth-review/tmp_users file. This file does not contain a line for the analyst user. 
The file includes two new entries with hashed passwords for the new_admin and new_developer users.
```
$ cat ~/DO280/labs/auth-review/tmp_users

tester:$apr1$EyWSDib4$uLoUMpwohNWUrU5L5ogkB/
leader:$apr1$/O8SyNdp$gjr.P7FMJbK2IebFU0QQn/
new_admin:$apr1$M5WHRPR2$GbGDkTK8QTrW2S/f2/1Kt1
new_developer:$apr1$dXdG8tWd$N8HA0SUe3TbqAhI049gOH0
```

### (2) Log in to your OpenShift cluster as the admin user with the redhatocp password. Configure your cluster to use the HTPasswd identity provider by using the defined user names and passwords in the ~/DO280/labs/auth-review/tmp_users file. For grading, use the auth-review name for the secret.

Log in to the cluster as the admin user:
```
$ oc login -u admin -p redhatocp https://api.ocp4.example.com:6443
Login successful.

...output omitted...
```

Create an auth-review secret by using the ~/DO280/labs/auth-review/tmp_users file:
```
$ oc create secret generic auth-review \
    --from-file htpasswd=/home/student/DO280/labs/auth-review/tmp_users \
    -n openshift-config
secret/auth-review created
```

Export the existing OAuth resource to ~/DO280/labs/auth-review/oauth.yaml:
```
$ oc get oauth cluster -o yaml > ~/DO280/labs/auth-review/oauth.yaml
```

Edit the ~/DO280/labs/auth-review/oauth.yaml file to add an identity provider by including the lines from the following 
example that are displayed in bold. Ensure that the htpasswd, mappingMethod, name, and type strings are at the same indentation level:
```
apiVersion: config.openshift.io/v1
kind: OAuth
...output omitted...
spec:
  identityProviders:
  - ldap:
...output omitted...
    type: LDAP
  - htpasswd:
      fileData:
        name: auth-review
    mappingMethod: claim
    name: htpasswd
    type: HTPasswd
```

Apply the customized resource that you defined in the previous step:
```
$ oc replace -f ~/DO280/labs/auth-review/oauth.yaml

oauth.config.openshift.io/cluster replaced
```

A successful update to the oauth/cluster resource re-creates the oauth-openshift pods in the openshift-authentication namespace:
```
$ watch oc get pods -n openshift-authentication
```

Wait until the new oauth-openshift pods are ready and running, and the previous pods have terminated:
```
Every 2.0s: oc get pods -n openshift-authentication            ...

NAME                               READY   STATUS    RESTARTS   AGE
oauth-openshift-68d6f666fd-z746p   1/1     Running   0          42s
```
Press Ctrl+C to exit the watch command.

NOTE  
Pods in the openshift-authentication namespace redeploy when the oc replace command succeeds.

In this exercise, changes to authentication might require a few minutes to apply.

You can examine the status of pods and deployments in the openshift-authentication namespace to monitor the authentication 
status. You can also examine the authentication cluster operator for further status information.

Provided that the previously created secret was created correctly, you can log in by using the HTPasswd identity provider.

### (3) Make the new_admin user a cluster administrator. Log in as both the new_admin and new_developer users to verify HTPasswd user configuration and cluster privileges.

Assign the new_admin user the cluster-admin role:
```
$ oc adm policy add-cluster-role-to-user cluster-admin new_admin

Warning: User 'new_admin' not found
clusterrole.rbac.authorization.k8s.io/cluster-admin added: "new_admin"
```

Log in to the cluster as the new_admin user to verify that HTPasswd authentication is configured correctly:
```
$ oc login -u new_admin -p 'L@bR3v!ew'

Login successful.

...output omitted...
```

Use the oc get nodes command to verify that the new_admin user has the cluster-admin role:
```
$ oc get nodes

NAME       STATUS   ROLES           AGE   VERSION
master01   Ready    master,worker   14d   v1.24.0+b62823b
```

Log in to the cluster as the new_developer user to verify that the HTPasswd authentication is configured correctly:
```
$ oc login -u new_developer -p 'L@bR3v!ew'

Login successful.

...output omitted...
```

Use the oc get nodes command to verify that the new_developer user does not have cluster administration privileges:
```
$ oc get nodes

Error from server (Forbidden): nodes is forbidden: User "new_developer" cannot list resource "nodes" in API group "" at the cluster scope
```

### (4) As the new_admin user, prevent users from creating projects in the cluster.

Log in to the cluster as the new_admin user:
```
$ oc login -u new_admin -p 'L@bR3v!ew'

Login successful.

...output omitted...
```

Remove the self-provisioner cluster role from the system:authenticated:oauth virtual group.
```
$ oc adm policy remove-cluster-role-from-group  self-provisioner system:authenticated:oauth

Warning: Your changes may get lost whenever a master is restarted, unless you prevent reconciliation of this rolebinding using the following command: oc annotate clusterrolebinding.rbac self-provisioners 'rbac.authorization.kubernetes.io/autoupdate=false' --overwrite
clusterrole.rbac.authorization.k8s.io/self-provisioner removed: "system:authenticated:oauth"
```

### (5) Create a managers group, and add the leader user to the group. Grant project creation privileges to the managers group. As the leader user, create the auth-review project.

Create a managers group:
```
$ oc adm groups new managers

group.user.openshift.io/managers created
```

Add the leader user to the managers group:
```
$ oc adm groups add-users managers leader

group.user.openshift.io/managers added: "leader"
```

Assign the self-provisioner cluster role to the managers group:
```
$ oc adm policy add-cluster-role-to-group  self-provisioner managers

clusterrole.rbac.authorization.k8s.io/self-provisioner added: "managers"
```

As the leader user, create the auth-review project:
```
$ oc login -u leader -p 'L@bR3v!ew'

Login successful.

...output omitted...
```

The user who creates a project is automatically assigned the admin role on the project:
```
$ oc new-project auth-review

Now using project "auth-review" on server "https://api.ocp4.example.com:6443".

...output omitted...
```

### (6) Create a developers group and grant edit privileges on the auth-review project. Add the new_developer user to the group.

Log in to the cluster as the new_admin user:
```
$ oc login -u new_admin -p 'L@bR3v!ew'

Login successful.

...output omitted...
```

Create a developers group:
```
$ oc adm groups new developers

group.user.openshift.io/developers created
```

Add the new_developer user to the developers group:
```
$ oc adm groups add-users developers new_developer

group.user.openshift.io/developers added: "new_developer"
```

Grant edit privileges to the developers group on the auth-review project:
```
$ oc policy add-role-to-group edit developers

clusterrole.rbac.authorization.k8s.io/edit added: "developers"
```

### (7) Create a qa group and grant view privileges on the auth-review project. Add the tester user to the group.

Create a qa group:
```
$ oc adm groups new qa

group.user.openshift.io/qa created
```

Add the tester user to the qa group:
```
$ oc adm groups add-users qa tester

group.user.openshift.io/qa added: "tester"
```

Grant view privileges to the qa group on the auth-review project:
```
$ oc policy add-role-to-group view qa

clusterrole.rbac.authorization.k8s.io/view added: "qa"
```

