# Setting up ExternalDNS for Infoblox

This tutorial describes how to setup ExternalDNS for usage with Infoblox.

Make sure to use **>=0.4.6** version of ExternalDNS for this tutorial. The only WAPI version that
has been validated is **v2.3.1**. It is assumed that the API user has rights to create objects of
the following types: `zone_auth`, `record:a`, `record:cname`, `record:txt`.

This tutorial assumes you have substituted the correct values for the following environment variables:

```
export EXTERNAL_DNS_INFOBLOX_GRID_HOST=127.0.0.1
export EXTERNAL_DNS_INFOBLOX_HTTP_POOL_CONNECTIONS=10
export EXTERNAL_DNS_INFOBLOX_HTTP_REQUEST_TIMEOUT=60
export EXTERNAL_DNS_INFOBLOX_SSL_VERIFY=1
export EXTERNAL_DNS_INFOBLOX_WAPI_PORT=443
export EXTERNAL_DNS_INFOBLOX_WAPI_VERSION=2.3.1
export EXTERNAL_DNS_INFOBLOX_WAPI_USERNAME=admin
export EXTERNAL_DNS_INFOBLOX_WAPI_PASSWORD=infoblox
```

## Creating an Infoblox DNS zone

The Infoblox provider for ExternalDNS will find suitable zones for domains it manages; it will
not automatically create zones.

Create an Infoblox DNS zone for "example.com":

```
$ curl -kl \
      -X POST \
      -d fqdn=example.com \
      -u ${EXTERNAL_DNS_INFOBLOX_WAPI_USERNAME}:${EXTERNAL_DNS_INFOBLOX_WAPI_PASSWORD} \
         https://${EXTERNAL_DNS_INFOBLOX_GRID_HOST}:${EXTERNAL_DNS_INFOBLOX_WAPI_PORT}/wapi/v${EXTERNAL_DNS_INFOBLOX_WAPI_VERSION}/zone_auth
```

Substitute a domain you own for "example.com" if desired.

## Creating an Infoblox Configuration Secret

For ExternalDNS to access the Infoblox API, create a Kubernetes secret.

To create the secret:

```
$ kubectl create secret generic external-dns \
      --from-literal=EXTERNAL_DNS_INFOBLOX_GRID_HOST=${EXTERNAL_DNS_INFOBLOX_GRID_HOST} \
      --from-literal=EXTERNAL_DNS_INFOBLOX_HTTP_POOL_CONNECTIONS=${EXTERNAL_DNS_INFOBLOX_HTTP_POOL_CONNECTIONS} \
      --from-literal=EXTERNAL_DNS_INFOBLOX_HTTP_REQUEST_TIMEOUT=${EXTERNAL_DNS_INFOBLOX_HTTP_REQUEST_TIMEOUT} \
      --from-literal=EXTERNAL_DNS_INFOBLOX_SSL_VERIFY=${EXTERNAL_DNS_INFOBLOX_SSL_VERIFY} \
      --from-literal=EXTERNAL_DNS_INFOBLOX_WAPI_PORT=${EXTERNAL_DNS_INFOBLOX_WAPI_PORT} \
      --from-literal=EXTERNAL_DNS_INFOBLOX_WAPI_VERSION=${EXTERNAL_DNS_INFOBLOX_WAPI_VERSION} \
      --from-literal=EXTERNAL_DNS_INFOBLOX_WAPI_USERNAME=${EXTERNAL_DNS_INFOBLOX_WAPI_USERNAME} \
      --from-literal=EXTERNAL_DNS_INFOBLOX_WAPI_PASSWORD=${EXTERNAL_DNS_INFOBLOX_WAPI_PASSWORD}
```

## Deploy ExternalDNS

Create a deployment file called `externaldns.yaml` with the following contents:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      containers:
      - name: external-dns
        image: registry.opensource.zalan.do/teapot/external-dns:v0.4.5
        args:
        - --source=service
        - --domain-filter=example.com # (optional) limit to only example.com domains; change to match the zone created above.
        - --provider=infoblox
        env:
        - name: EXTERNAL_DNS_INFOBLOX_GRID_HOST
          valueFrom:
            secretKeyRef:
              name: external-dns
              key: EXTERNAL_DNS_INFOBLOX_GRID_HOST
        - name: EXTERNAL_DNS_INFOBLOX_HTTP_POOL_CONNECTIONS
          valueFrom:
            secretKeyRef:
              name: external-dns
              key: EXTERNAL_DNS_INFOBLOX_HTTP_POOL_CONNECTIONS
        - name: EXTERNAL_DNS_INFOBLOX_HTTP_REQUEST_TIMEOUT
          valueFrom:
            secretKeyRef:
              name: external-dns
              key: EXTERNAL_DNS_INFOBLOX_HTTP_REQUEST_TIMEOUT
        - name: EXTERNAL_DNS_INFOBLOX_SSL_VERIFY
          valueFrom:
            secretKeyRef:
              name: external-dns
              key: EXTERNAL_DNS_INFOBLOX_SSL_VERIFY
        - name: EXTERNAL_DNS_INFOBLOX_WAPI_PORT
          valueFrom:
            secretKeyRef:
              name: external-dns
              key: EXTERNAL_DNS_INFOBLOX_WAPI_PORT
        - name: EXTERNAL_DNS_INFOBLOX_WAPI_VERSION
          valueFrom:
            secretKeyRef:
              name: external-dns
              key: EXTERNAL_DNS_INFOBLOX_WAPI_VERSION
        - name: EXTERNAL_DNS_INFOBLOX_WAPI_USERNAME
          valueFrom:
            secretKeyRef:
              name: external-dns
              key: EXTERNAL_DNS_INFOBLOX_WAPI_USERNAME
        - name: EXTERNAL_DNS_INFOBLOX_WAPI_PASSWORD
          valueFrom:
            secretKeyRef:
              name: external-dns
              key: EXTERNAL_DNS_INFOBLOX_WAPI_PASSWORD
```

Create the deployment for ExternalDNS:

```
$ kubectl create -f externaldns.yaml
```

## Deploying an Nginx Service

Create a service file called 'nginx.yaml' with the following contents:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
spec:
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: example.com
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Note the annotation on the service; use the same hostname as the Infoblox DNS zone created above. The annotation may also be a subdomain
of the DNS zone (e.g. 'www.example.com').

ExternalDNS uses this annotation to determine what services should be registered with DNS.  Removing the annotation
will cause ExternalDNS to remove the corresponding DNS records.

Create the deployment and service:

```
$ kubectl create -f nginx.yaml
```

It takes a little while for the Infoblox cloud provider to create an external IP for the service.  Check the status by running
`kubectl get services nginx`.  If the `EXTERNAL-IP` field shows an address, the service is ready to be accessed externally.

Once the service has an external IP assigned, ExternalDNS will notice the new service IP address and synchronize
the Infoblox DNS records.

## Verifying Infoblox DNS records

Run the following command to view the A records for your Infoblox DNS zone:

```
$ curl -kl \
      -X GET \
      -u ${EXTERNAL_DNS_INFOBLOX_WAPI_USERNAME}:${EXTERNAL_DNS_INFOBLOX_WAPI_PASSWORD} \
         https://${EXTERNAL_DNS_INFOBLOX_GRID_HOST}:${EXTERNAL_DNS_INFOBLOX_WAPI_PORT}/wapi/v${EXTERNAL_DNS_INFOBLOX_WAPI_VERSION}/record:a?zone=example.com
```

Substitute the zone for the one created above if a different domain was used.

This should show the external IP address of the service as the A record for your domain ('@' indicates the record is for the zone itself).

## Clean up

Now that we have verified that ExternalDNS will automatically manage Infoblox DNS records, we can delete the tutorial's
DNS zone:

```
$ curl -kl \
      -X DELETE \
      -u ${EXTERNAL_DNS_INFOBLOX_WAPI_USERNAME}:${EXTERNAL_DNS_INFOBLOX_WAPI_PASSWORD} \
         https://${EXTERNAL_DNS_INFOBLOX_GRID_HOST}:${EXTERNAL_DNS_INFOBLOX_WAPI_PORT}/wapi/v${EXTERNAL_DNS_INFOBLOX_WAPI_VERSION}/zone_auth?fqdn=example.com
```
