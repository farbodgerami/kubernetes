# Install NGINX Ingress Controller
### If you are running an old version of Kubernetes (1.18 or earlier), please read this paragraph for specific instructions.
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.47.0/deploy/static/provider/cloud/deploy.yaml
```
### Pre-flight check
A few pods should start in the ingress-nginx namespace:

```bash
# check pod
kubectl get pods --namespace=ingress-nginx

# check pod and service
kubectl get all -n ingress-nginx
```
### After a while, they should all be running. The following command will wait for the ingress controller pod to be up, running, and ready:

```
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

# Install cert-manager
### Below you will find details on various scenarios we aim to support and that are compatible with the documentation on this website. Furthermore, the most applicable install methods are listed below for each of the situations.

### Default static install
You don’t require any tweaking of the cert-manager install parameters.
The default static configuration can be installed as follows:
```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.yaml
```
### Pre-flight check
A few pods should start in the cert-manager namespace:

```bash
# check pod
kubectl get pods -n cert-manager

# check pod and service
kubectl get all -n cert-manager
```


## Issuer Configuration
The first thing you’ll need to configure after you’ve installed cert-manager is an issuer which you can then use to issue certificates.

This section documents how the different issuer types can be configured. You might want to read more about Issuer and ClusterIssuer resources here.

cert-manager comes with a number of built-in certificate issuers which are denoted by being in the cert-manager.io group. You can also install external issuers in addition to the built-in types. Both built-in and external issuers are treated the same and are configured similarly.

When using ClusterIssuer resource types, ensure you understand the purpose of the Cluster Resource Namespace; this can be a common source of issues for people getting started with cert-manager.

### Create clusterissuer

before creating ingress change `YOUR_EMAIL_ADDRESS` on this manifest.

```bash
cat > clusterissuer.yml <<- EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: <YOUR_EMAIL_ADDRESS>
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx
          serviceType: ClusterIP
EOF
kubectl create -f ./clusterissuer.yml
kubectl get clusterissuer
```

# Sample ingress manifest
### Create https ingress for nginx service

before creating ingress change `SUB.DOMAIN.TLD` on this manifest.
```bash
cat > ngnix-ingress.yml <<- EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ngnixing
  annotations:
    kubernetes.io/ingress.class: "nginx"    
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    certmanager.k8s.io/acme-http01-edit-in-place: "true"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    kubernetes.io/tls-acme: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: <SUB.DOMAIN.TLD>
      http:
        paths:
          - pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 8080
            path: /
  tls:
    - hosts:
      - <SUB.DOMAIN.TLD>
      secretName: kubetls
EOF

kubectl create -f ./weave-ingress.yml
kubectl -n weave get ingress
curl -I http://SUB.DOMAIN.TLD
```

## **NOTE:** Be sure to make the DNS record before applying.

### haproxy  install and configuration

```bash

apt install -y haproxy 

echo "copy and move haproxy config"
cat <<EOT >> /etc/haproxy/haproxy.cfg
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http
listen Stats-Page
  bind *:8000
  mode http
  stats enable
  stats hide-version
  stats refresh 10s
  stats uri /
  stats show-legends
  stats show-node

frontend fe-http
   bind 0.0.0.0:80
   mode tcp
   option tcplog
   default_backend be-http

backend be-http
   mode tcp
   option tcp-check
   balance roundrobin
   default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
   server control-plane-1 5.34.207.238:30306 check

frontend fe-https
   bind 0.0.0.0:443
   mode tcp
   option tcplog
   default_backend be-https

backend be-https
   mode tcp
   option tcp-check
   balance roundrobin
   default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
   server control-plane-1 5.34.207.238:31367 check

EOT

haproxy -c -f /etc/haproxy/haproxy.cfg

echo "Enable and start haproxy service"
{
systemctl enable haproxy
systemctl restart haproxy
systemctl is-active --quiet haproxy && echo -e "\e[1m \e[96m haproxy service: \e[30;48;5;82m \e[5mRunning \e[0m" || echo -e "\e[1m \e[96m docker service: \e[30;48;5;196m \e[5mNot Running \e[0m"
}

```