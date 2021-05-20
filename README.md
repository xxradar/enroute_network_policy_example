# enroute_network_policy_example



## Install demo app
```
git clone https://github.com/xxradar/app_routable_demo.git
cd ./app_routable_demo
./setup.sh
kubectl apply -n app-routable-demo -f ./netw_policy.yaml
watch kubectl get po -n app-routable-demo
```
Check if the app works
```
kubectl run -it -n app-routable-demo --rm -l app=siege --image xxradar/hackon mycurler -- bash
       curl -v -H "Cookie: loc=client" http://zone1/app3
```

## Installing Helm

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## Install enroute CRDs and gateway
```
helm repo add saaras https://getenroute.io
helm search repo
helm install saaras/enroute-crds --generate-name
helm ls --all-namespaces
helm install enroute-demo saaras/enroute   \
    --set enrouteService.rbac.create=true
kubectl get svc -n enroutedemo
```
Deploy Gatewayhosts ...
```
helm install httpapproutabledemo-service-policy saaras/service-policy \
        --set service.name=zone1                       \
        --set service.namespace=app-routable-demo             \
        --set service.prefix=/                    \
        --set service.port=80      \
        --set service.fqdn=app-routable-demo.dockerhack.me
```
Verify the Gatewayhost
```
kubectl get GatewayHosts -n app-routable-demo
```
Does this work? A: No
```
kubectl get svc -n enroutedemo
```
Connect to the LoadBalancer or NodeIP:port
```
$ curl -kv http://10.11.2.48:32245/app1
*   Trying 10.11.2.48...
* TCP_NODELAY set
* Connected to 10.11.2.48 (10.11.2.48) port 32245 (#0)
> GET /app1 HTTP/1.1
> Host: 10.11.2.48:32245
> User-Agent: curl/7.58.0
> Accept: */*
>
< HTTP/1.1 404 Not Found
< date: Thu, 20 May 2021 09:57:40 GMT
< server: envoy
< content-length: 0
<
* Connection #0 to host 10.11.2.48 left intact
```
This proves that the gateway is up (404 error)

```
$ curl -kv -H "Host: app-routable-demo.dockerhack.me"  http://10.11.2.48:32245/app1
*   Trying 10.11.2.48...
* TCP_NODELAY set
* Connected to 10.11.2.48 (10.11.2.48) port 32245 (#0)
> GET /app1 HTTP/1.1
> Host: app-routable-demo.dockerhack.me
> User-Agent: curl/7.58.0
> Accept: */*
>
< HTTP/1.1 503 Service Unavailable
< content-length: 91
< content-type: text/plain
< vary: Accept-Encoding
< date: Thu, 20 May 2021 09:59:48 GMT
< server: envoy
<
* Connection #0 to host 10.11.2.48 left intact
upstream connect error or disconnect/reset before headers. reset reason: connection failure
```
The network policies in place for ns app-routable-demo prevent Enroute to connect to the configure services

Create a network policy allowing enroute to connect to service in app-routable-demo namespace
```
kubectl apply -n app-routable-demo -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-control
  namespace: app-routable-demo
spec:
  podSelector:
    matchLabels:
      app: nginx-zone1
  policyTypes:
    - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: enroutedemo
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
EOF
```

Let's check if this works ...
```
 curl -kv -H "Host: app-routable-demo.dockerhack.me"  http://10.11.2.48:32245/app1
*   Trying 10.11.2.48...
* TCP_NODELAY set
* Connected to 10.11.2.48 (10.11.2.48) port 32245 (#0)
> GET /app1 HTTP/1.1
> Host: app-routable-demo.dockerhack.me
> User-Agent: curl/7.58.0
> Accept: */*
>
< HTTP/1.1 200 OK
< server: envoy
< date: Thu, 20 May 2021 10:11:23 GMT
< content-type: application/json; charset=utf-8
< content-length: 880
< x-powered-by: Express
< etag: W/"370-/QgGhn56ARvQ++aLCzdTavs+re0"
< x-envoy-upstream-service-time: 4
< vary: Accept-Encoding
<
{
  "path": "/app1",
  "headers": { ...
  ```


Let's add TLS support for Let's Encrypt
```
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm search repo
```
```
helm install cert-manager jetstack/cert-manager \
    --namespace cert-manager                    \
    --version v1.2.0                            \
    --create-namespace                          \
    --set installCRDs=true
```
```
helm ls --all-namespaces
```

(This is requiring your cluster is available through a LB and DNS ... )

Enroute needs access to validate the acme-challenge for Let's Encrypt ...

```
kubectl apply -n app-routable-demo -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cert-validation
spec:
  podSelector:
    matchLabels:
      "acme.cert-manager.io/http01-solver": "true"
  policyTypes:
    - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          "kubernetes.io/metadata.name": "enroutedemo"
    ports:
    - protocol: TCP
      port: 8089
EOF
```

```
helm upgrade httpapproutabledemo-service-policy  saaras/service-policy       \
        --set service.namespace=app-routable-demo               \
        --set service.name=zone1               \
        --set service.prefix=/                               \
        --set service.port=80                                   \
        --set service.enableTLS=true                            \
        --set autoTLS.certificateCN=app-routable-demo.dockerhack.me     \
        --set autoTLS.enableProd=true                           \
        --set autoTLS.email=xxradar@radarhack.com                \
        --set filters.lua.enable=true                           \
        --set filters.cors.enable=true                          \
         --set autoTLS.createIssuers=true                      \
        --set filters.jwt.enable=false                          \
        --set filters.ratelimit.enable=true     \
        --set service.fqdn=app-routable-demo.dockerhack.me
```
Let's check if this works ...
```
$ curl -kv -H "Host: app-routable-demo.dockerhack.me"  https://app-routable-demo.dockerhack.me/app1
*   Trying 10.11.2.48...
* TCP_NODELAY set
* Connected to app-routable-demo.dockerhack.me (10.11.2.48) port 31091 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: /etc/ssl/certs
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS Unknown, Certificate Status (22):
* TLSv1.3 (IN), TLS handshake, Unknown (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Client hello (1):
* TLSv1.3 (OUT), TLS Unknown, Certificate Status (22):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=app-routable-demo.dockerhack.me
*  start date: May 20 09:40:40 2021 GMT
*  expire date: Aug 18 09:40:40 2021 GMT
*  issuer: C=US; O=Let's Encrypt; CN=R3
```
