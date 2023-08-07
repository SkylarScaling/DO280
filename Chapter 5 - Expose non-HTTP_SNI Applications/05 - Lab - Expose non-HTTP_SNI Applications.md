# Lab: Expose non-HTTP/SNI Applications

Expose applications to external access without using an ingress controller.

## Outcomes

* Expose a non-http application to external access by using the LoadBalancer type service.
* Configure a network attachment definition for an isolated network.
* Make an application accessible outside the cluster on an isolated network by using an existing node network interface.

## Instructions

### (1) Deploy the virtual-rtsp application to a new non-http-review-rtsp project as the developer user with the developer password, and verify that the virtual-rtsp pod is running.

The application consists of the ~/DO280/labs/non-http-review/virtual-rtsp.yaml file.

Log in to your OpenShift cluster as the developer user with the developer password.
```
[student@workstation ~]$ oc login -u developer -p developer https://api.ocp4.example.com:6443

Login successful.
...output omitted...
```

Change to the ~/DO280/labs/non-http-review directory.
```
[student@workstation ~]$ cd ~/DO280/labs/non-http-review
```

Create a non-http-review-rtsp project.
```
[student@workstation non-http-review]$ oc new-project non-http-review-rtsp

Now using project "non-http-review-rtsp" on server ...
...output omitted...
```

Use the oc create command to create the virtual-rtsp deployment by using the virtual-rtsp.yaml file.
```
[student@workstation non-http-review]$ oc create -f virtual-rtsp.yaml

Warning: would violate PodSecurity "restricted:v1.24": ...output omitted...
deployment.apps/virtual-rtsp created
```

List the deployments and pods. Wait for the virtual-rtsp pod to be ready. Press Ctrl+C to exit the watch command.
```
[student@workstation non-http-review]$ watch oc get deployments,pods

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/virtual-rtsp   1/1     1            1           21s

NAME                                READY   STATUS    RESTARTS   AGE
pod/virtual-rtsp-54d8d6b57d-6jsvm   1/1     Running   0          21s
```

### (2) Expose the virtual-rtsp deployment by using the LoadBalancer service.

Create a load balancer service for the virtual-rtsp deployment.
```
[student@workstation non-http-review]$ oc expose deployment/virtual-rtsp \
  --name=virtual-rtsp-loadbalancer --type=LoadBalancer

service/virtual-rtsp-loadbalancer exposed
```

Retrieve the external IP address of the virtual-rtsp-loadbalancer service.
```
[student@workstation non-http-review]$ oc get svc/virtual-rtsp-loadbalancer

NAME                       TYPE          ...  EXTERNAL-IP    PORT(S)
virtual-rtsp-loadbalancer  LoadBalancer  ...  192.168.50.20  8554:32570/TCP
```

The virtual-rtsp-loadbalancer has the 192.168.50.20 external IP address.

### (3) Access the virtual-rtsp application by using the URL in the media player. Run the vlc -q -V xcb_x11 rtsp://EXTERNAL-IP:8554/stream command to play the stream in the media player.

Open the URL in the media player to confirm that the video stream is working correctly.

rtsp://192.168.50.20:8554/stream
```
[student@workstation non-http-review]$ vlc -q -V xcb_x11 rtsp://192.168.50.20:8554/stream
...output omitted...
```

Close the media player window after confirming that the video stream works correctly.

### (4) Deploy the nginx deployment to a new non-http-review-nginx project as the developer user with the developer password, and verify that the nginx pod is running. The application consists of the ~/DO280/labs/non-http-review/nginx.yaml file.

Create a non-http-review-nginx project.
```
[student@workstation non-http-review]$ oc new-project non-http-review-nginx

Now using project "non-http-review-nginx" on server ...
...output omitted...
```

Use the oc create command to create the nginx deployment by using the nginx.yaml file.
```
[student@workstation non-http-review]$ oc create -f nginx.yaml

Warning: would violate PodSecurity "restricted:v1.24": ...output omitted...
deployment.apps/nginx created
```

List the deployments and pods. Wait for the nginx pod to be ready. Press Ctrl+C to exit the watch command.
```
[student@workstation non-http-review]$ watch oc get deployments,pods

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           53s

NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-649779cbd-d6sbv   1/1     Running   0          53s
```

### (5) Configure a network attachment definition for the ens4 interface, so that the isolated network can be attached to a pod.

The master01 node has two Ethernet interfaces. The ens3 interface is the main network interface of the cluster. The ens4 
interface is an additional network interface for exercises that require an additional network. The ens4 interface is attached 
to a 192.168.51.0/24 network, with the 192.168.51.10 IP address.

You can modify the ~/DO280/labs/non-http-review/network-attachment-definition.yaml file to configure a network attachment 
definition by using the following parameters:

|Parameter	|Value
|---------- |----- |
|name |custom |
|type |host-device |
|device	|ens4 |
|ipam.type |static |
|ipam.addresses	|{"address": "192.168.51.10/24"} |

Log in to your OpenShift cluster as the admin user with the redhatocp password.
```
[student@workstation non-http-review]$ oc login -u admin -p redhatocp https://api.ocp4.example.com:6443

Login successful.
...output omitted...
```

Edit the ~/DO280/labs/non-http-review/network-attachment-definition.yaml file. Use the custom name, the host-device type, and the ens4 device. Configure IP address management to use the static type, with the 192.168.51.10/24 address.
```
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: custom
spec:
  config: |-
    {
      "cniVersion": "0.3.1",
      "name": "custom",
      "type": "host-device",
      "device": "ens4",
      "ipam": {
        "type": "static",
        "addresses": [
          {"address": "192.168.51.10/24"}
        ]
      }
    }
```

Use the oc create command to create the network attachment definition.
```
[student@workstation non-http-review]$ oc create -f network-attachment-definition.yaml

networkattachmentdefinition.k8s.cni.cncf.io/custom created
```

### (6) The nginx application does not contain any services, so the application is not accessible outside the pod network.

Assign the ens4 network interface exclusively to the nginx pod, by using the custom network attachment definition. Edit the 
nginx deployment to add the k8s.v1.cni.cncf.io/networks annotation with the custom value as the developer user with the developer password.

Log in to the OpenShift cluster as the developer user with the developer password.
```
[student@workstation non-http-review]$ oc login -u developer -p developer https://api.ocp4.example.com:6443

Login successful.

...output omitted...
```

Edit the ~/DO280/labs/non-http-review/nginx.yaml file to add the k8s.v1.cni.cncf.io/networks annotation with the custom value.
```
...output omitted...
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        k8s.v1.cni.cncf.io/networks: custom
    spec:
      containers:
...output omitted...
```

Use the oc apply command to add the annotation.
```
[student@workstation non-http-review]$ oc apply -f nginx.yaml

Warning: would violate PodSecurity "restricted:v1.24": ...output omitted...
deployment.apps/nginx configured
```

Wait for the nginx pod to be ready. Press Ctrl+C to exit the watch command.
```
[student@workstation non-http-review]$ watch oc get deployments,pods

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           34m

NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-6f45d9f89-wp2gg   1/1     Running   0          53s
```

Examine the k8s.v1.cni.cncf.io/networks-status annotation in the pod.
```
[student@workstation ~]$ oc get pod nginx-6f45d9f89-wp2gg \
  -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/networks-status}'
[{
    "name": "ovn-kubernetes",
    "interface": "eth0",
    "ips": [
        "10.8.0.82"
    ],
    "mac": "0a:58:0a:08:00:52",
    "default": true,
    "dns": {}
},{
    "name": "non-http-review-nginx/custom",
    "interface": "net1",
    "ips": [
        "192.168.51.10"
    ],
    "mac": "52:54:00:01:33:0a",
    "dns": {}
}]
```

NOTE  
The period is the JSONPath field access operator. Normally, you use the period to access parts of the resource, such as in 
the .metadata.annotations JSONPath expression. To access fields that contain periods with JSONPath, you must escape the periods 
with a backslash (\\).

### (7) Verify that you can access the nginx application from the utility machine by using the following URL:

http://isolated-network-IP-address:8080

Use the ssh command to connect to the utility machine.
```
[student@workstation non-http-review]$ ssh utility

...output omitted...

```

Verify that the nginx application is accessible. Use the IP address on the isolated network to access the nginx application.
```
[student@utility ~]$ curl 'http://192.168.51.10:8080/'
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>
```

Exit the SSH session to go back to the workstation machine.

### (8) Verify that you cannot access the nginx application from the workstation machine, because the workstation machine cannot access the isolated network.

Verify that the nginx application is not accessible from the workstation machine.
```
[student@workstation non-http-review]$ curl 'http://192.168.51.10:8080/'

curl: (7) Failed to connect to 192.168.51.10 port 8080: Connection timed out
```
