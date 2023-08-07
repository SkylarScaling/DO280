# Lab: Network Security

Configure firewall rules to protect microservice communication, and also configure TLS encryption between those microservices 
and for external access.

## Outcomes

* Encrypt internal traffic between pods by using TLS service secrets that OpenShift generates.
* Route external traffic to terminate TLS within the cluster.
* Restrict ingress traffic for a group of pods by using network policies.

### (1) Log in to your OpenShift cluster as the admin user with the redhatocp password.

Use the oc login command to log in to your OpenShift cluster as the admin user:
```
$ oc login -u admin -p redhatocp https://api.ocp4.example.com:6443

Login successful.

...output omitted...
```

### (2) Create the stock-service-cert secret for the OpenShift service certificate to encrypt communications between the product and the stock microservices.

Change to the network-review project:
```
[student@workstation ~]$ oc project network-review

Now using project "network-review" on server "https://api.ocp4.example.com:6443"
```

Change to the ~/DO280/labs/network-review directory to access the lab files.
```
[student@workstation ~]$ cd ~/DO280/labs/network-review
```

Edit the stock-service.yaml manifest to configure the stock service with the service.beta.openshift.io/serving-cert-secret-name: stock-service-cert annotation. This annotation creates the stock-service-cert secret with the service certificate and the key.
```
apiVersion: v1
kind: Service
metadata:
  name: stock
  namespace: network-review
  annotations:
    service.beta.openshift.io/serving-cert-secret-name: stock-service-cert
spec:
...output omitted...
```

Apply the stock service changes by using the oc apply command.
```
[student@workstation network-review]$ oc apply -f stock-service.yaml

service/stock configured
```

Verify that the stock-service-cert secret contains a valid certificate for the stock.network-review.svc hostname in the tls.crt secret key. Decode the secret output with the base64 command by using the -d option. Then, use the openssl x509 command with the -in - option to read the output from standard input, and use the -text option to print the certificate in text form.
```
[student@workstation network-review]$ oc get secret stock-service-cert \
  --output="jsonpath={.data.tls\.crt}" \
  | base64 -d \
  | openssl x509 -in - -text
...output omitted...
      X509v3 Subject Alternative Name:
          DNS:stock.network-review.svc, DNS:stock.network-review.svc.cluster.local
...output omitted...
```

### (3) Configure TLS on the stock microservice by using the stock-service-cert secret that OpenShift generates.

Use the following settings in the deployment to configure TLS:
* Set the path for the certificate and key to /etc/pki/stock/.
* Set the TLS_ENABLED environment variable to "true".
* Update the liveness and readiness probes to use TLS.
* Change the service to listen on the standard HTTPS 443 port.

Edit the stock-deployment.yaml file to mount the stock-service-cert secret on the /etc/pki/stock/ path.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stock
  namespace: network-review
spec:
...output omitted...
    spec:
      containers:
        - name: stock
...output omitted...
          env:
            - name: TLS_ENABLED
              value: "false"
          volumeMounts:
            - name: stock-service-cert
              mountPath: /etc/pki/stock/
      volumes:
        - name: stock-service-cert
          secret:
            defaultMode: 420
            secretName: stock-service-cert
```

Edit the stock deployment in the stock-deployment.yaml file to configure TLS for the application and for the liveness and readiness probes.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stock
  namespace: network-review
spec:
...output omitted...
    spec:
      containers:
      - name: stock
...output omitted...
        ports:
          - containerPort: 8085
        readinessProbe:
          httpGet:
            port: 8085
            path: /readyz
            scheme: HTTPS
        livenessProbe:
          httpGet:
            port: 8085
            path: /livez
            scheme: HTTPS
        env:
          - name: TLS_ENABLED
            value: "true"
...output omitted...
```

Apply the stock deployment updates by using the oc apply command.
```
[student@workstation network-review]$ oc apply -f stock-deployment.yaml

deployment/stock configured
```

Edit the stock-service.yaml file to configure the stock service to listen on the standard HTTPS 443 port.
```
apiVersion: v1
kind: Service
metadata:
  name: stock
  namespace: network-review
  annotations:
    service.beta.openshift.io/serving-cert-secret-name: stock-service-cert
spec:
  selector:
    app: stock
  ports:
    - port: 443
      targetPort: 8085
      name: https
```
Apply the stock service changes by using the oc apply command.
```
[student@workstation network-review]$ oc apply -f stock-service.yaml

service/stock configured
```

### (4) Configure TLS between the product and the stock microservices by using the internal Certificate Authority (CA) from OpenShift.

The product microservice requires the following settings:

* The CERT_CA environment variable that is set to /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem to access the OpenShift CA
* The STOCK_URL environment variable with the HTTPS protocol

Edit the configuration map in the service-ca-configmap.yaml file to add the service.beta.openshift.io/inject-cabundle: "true" annotation. 
This annotation injects the OpenShift internal CA into the service-ca configuration map.
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: service-ca
  namespace: network-review
  annotations:
    service.beta.openshift.io/inject-cabundle: "true"
data: {}
```

Create the service-ca configuration map by using the oc create command.
```
[student@workstation network-review]$ oc create -f service-ca-configmap.yaml

configmap/service-ca created
```

Verify that OpenShift injects the CA certificate by describing the service-ca configuration map with the oc describe command.
```
[student@workstation network-review]$ oc describe configmap service-ca

Name:         service-ca
Namespace:    network-review
Labels:       <none>
Annotations:  service.beta.openshift.io/inject-cabundle: true

Data
====
service-ca.crt:
-----
-----BEGIN CERTIFICATE-----
```

Edit the product-deployment.yaml file to configure the product deployment to use the service-ca configuration map and to 
update the STOCK_URL environment variable to use the HTTPS protocol.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product
  namespace: network-review
spec:
...output omitted...
    spec:
      containers:
      - name: product
...output omitted...
        env:
          - name: TLS_ENABLED
            value: "false"
          - name: STOCK_URL
            value: "https://stock.network-review.svc"
        volumeMounts:
          - name: trusted-ca
            mountPath: /etc/pki/ca-trust/extracted/pem

      volumes:
        - name: trusted-ca
          configMap:
            defaultMode: 420
            name: service-ca
            items:
              - key: service-ca.crt
                path: tls-ca-bundle.pem
```

Apply the product deployment updates by using the oc apply command.
```
[student@workstation network-review]$ oc apply -f product-deployment.yaml

deployment/product configured
```

Send a request to the https://stock.network-review.svc/product/1 URL from product deployment to verify that you can query the stock microservice by using HTTPS. Run the oc exec command to run the curl command to send a request to the stock microservice.
```
[student@workstation network-review]$ oc exec deployment/product 
  -- curl -s https://stock.network-review.svc/product/1

10
```

### (5) Configure TLS on the product microservice by using a signed certificate by a corporate CA to accept TLS connections from outside the cluster.

You have the CA certificate and the signed certificate for the product.apps.ocp4.example.com domain in the certs directory of the lab.

Use the following settings in the deployment to configure TLS:
* Set the path for the certificate and key to /etc/pki/product/.
* Set the TLS_ENABLED environment variable to the "true" value.
* Update the liveness and readiness probes to use TLS.

Create the passthrough-cert secret by using the product.pem certificate and the product.key key from the lab directory.
```
[student@workstation network-review]$ oc create secret tls passthrough-cert \
  --cert certs/product.pem --key certs/product.key
secret/passthrough-cert created
```

Edit the product deployment to mount the passthrough-cert secret on the /etc/pki/product/ path.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product
spec:
...output omitted...
    spec:
      containers:
      - name: product
...output omitted...
        volumeMounts:
          - name: passthrough-cert
            mountPath: /etc/pki/product/
          - name: trusted-ca
            mountPath: /etc/pki/ca-trust/extracted/pem

      volumes:
        - name: passthrough-cert
          secret:
            defaultMode: 420
            secretName: passthrough-cert
        - name: trusted-ca
          configMap:
            defaultMode: 420
            name: service-ca
            items:
              - key: service-ca.crt
                path: tls-ca-bundle.pem
```

Edit the product deployment to configure TLS for the application and for the liveness and readiness probes.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product
spec:
...output omitted...
    spec:
      containers:
      - name: product
...output omitted...
        ports:
          - containerPort: 8080
        readinessProbe:
          httpGet:
            port: 8080
            path: /readyz
            scheme: HTTPS
        livenessProbe:
          httpGet:
            port: 8080
            path: /livez
            scheme: HTTPS
        env:
          - name: TLS_ENABLED
            value: "true"
          - name: STOCK_URL
            value: "https://stock.network-review.svc"
...output omitted...
```

Apply the product deployment updates by using the oc apply command.
```
[student@workstation network-review]$ oc apply -f product-deployment.yaml

deployment.apps/product configured
```

### (6) Expose the product microservice to outer cluster access by using the FQDN in the signed certificate by the corporate CA. Use the product.apps.ocp4.example.com hostname.

Create a passthrough route for the product service by using the product.apps.ocp4.example.com hostname.
```
[student@workstation network-review]$ oc create route passthrough product-https \
  --service product --port 8080 \
  --hostname product.apps.ocp4.example.com

route.route.openshift.io/product-https created
```

Verify that you can query the product microservice from outside the cluster by using the curl command with the ca.pem CA certificate.
```
[student@workstation network-review]$ curl --cacert certs/ca.pem https://product.apps.ocp4.example.com/products

[{"id":1,"name":"rpi4_4gb","stock":10},{"id":2,"name":"rpi4_8gb","stock":5}]
```

### (7) Configure network policies to accept only ingress connections to the stock pod on the 8085 port that come from a pod with the app=product label.

Edit the stock-ingresspolicy.yaml to add the network policy specification.
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-stock-policy
spec:
  podSelector:
    matchLabels:
      app: stock
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: product
      ports:
        - protocol: TCP
          port: 8085
```

Create the network policy.
```
[student@workstation network-review]$ oc create -f stock-ingresspolicy.yaml

networkpolicy.networking.k8s.io/stock-ingress-policy created
```

### (8) Configure network policies to accept only ingress connections to the product pod on the 8080 port that come from the OpenShift router pods.

Edit the product-ingresspolicy.yaml file to accept ingress connections from router pods by adding a namespace selector with the network.openshift.io/policy-group=ingress label.
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: product-ingress-policy
spec:
  podSelector:
    matchLabels:
      app: product
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              network.openshift.io/policy-group: ingress
      ports:
        - protocol: TCP
          port: 8080
```

Create the network policy.
```
[student@workstation network-review]$ oc create -f product-ingresspolicy.yaml

networkpolicy.networking.k8s.io/product-ingress-policy created
```
