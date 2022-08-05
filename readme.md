# How to use Wireguard client as a sidecar in Kubernetes

This example show case how to use Wireguard VPN as a sidecar in Kubernetes to allow any application to use it.

On a local container engine , the Wireguard server is run and expose it's port publicly.
On the same engine a Nginx is deployed without port exposition.

On a Kind cluster, the pod containing the Wireguard sidecar is run, this pod can access the Nginx that is not exposed using the VPN connection

Why Wireguard : 

TLDR : Step to run the example
1. Run the server with './wireguard-server.sh'
2. Run the local nginx 'docker run  -d nginx'
3. Run the kind cluster 'kind create cluster --config kind-conf.yaml --name wireguard'
4. 
 

## Setup Wireguard server and local Nginx

### Wireguard
The script './wireguard-server.sh' run the Wireguard server using the configuration in `config/wg0.conf`

The configuration of the server looks like : 

```
[Interface]
Address = 172.16.16.0/20
ListenPort = 51820
PrivateKey = OIviMX9BPHk1w/bvsXW0Qc2/mY3+HS3iS31aEtsn+Uc=
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostUp = sysctl -w -q net.ipv4.ip_forward=1
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
PostDown = sysctl -w -q net.ipv4.ip_forward=0

[Peer]
# Example Peer 1
PublicKey = AOIzLd2C71DtY8DWgUfuMllRNa0iR1O3tO2WbFO7ICU=
AllowedIPs = 172.16.16.10
```

**Depending of your device, the interface may not be `eth0`** 

This configure Wireguard to  :
- use the `172.16.16.0/20` subnet
- listen to the `51820`
- configure the private key
- allow the Peer with IP `172.16.16.10`

For more detail, go to Wireguard documentation -> I did not :) 

### Nginx

Run the local Nginx `docker run --name nginx  -d nginx `
Get the IP of the running Nginx instance `docker inspect nginx | grep 'IPAddress'`


Connect to Wireguard server and check the access to Nginx  : 

```
docker exec -it wireguard curl http://<nginx_ip> 
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Setup Wireguard client as Sidecar

### Start the Kind cluster
Start the cluster using the provided configuration:  
```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 31820
    hostPort: 31820
    protocol: UDP
  - containerPort: 30443
    hostPort: 8443
    protocol: TCP
- role: worker
- role: worker
```

Note : this config was used with Wireguard server as the Sidecar and exposed it using a NodePort.

`kind create cluster --config kind-conf.yaml --name wireguard`

### Configure the secret
Edit the secret to configure the endpoint with the address of the host gateway. 
In the case of Kind cluster it's `172.18.0.1`

```
cat kube-client-mode/wiregard-secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: wireguard
  namespace: wireguard
type: Opaque
stringData:
  wg0.conf.template: |
    [Interface]
    PrivateKey = oG2sa0u9qJGWGC8+vtXRsLtI0IxXKtaYzGlpPzqD91k=
    Address = 172.16.16.10/20
    DNS = 1.1.1.1

    [Peer]
    PublicKey = CSB59ZuD/YVwKWRpVfRpzhirVxfAr36E5770/JDqDx4=
    AllowedIPs = 0.0.0.0/0
    Endpoint = 172.18.0.1:51820
    PersistentKeepalive = 25
```
The PrivateKey in the [Interface] section corresponds to the PublicKey that has been configured in the server configuration file as peer, as well as the Address.
[Peer] section
By setting the AllowedIPs to 0.0.0.0/0 we force the complete traffic from the clients computer to be routed through the VPN server

### Run the Pod

Deploy the secret and the pod :
```
k create ns wireguard
k apply -f wireguard-sercret.yaml
k apply -f wireguard-deployment.yaml
```

### Validate the connection
Connect to the Wireguard server and check the client connection
```
docker exec -it wireguard wg
interface: wg0
  public key: CSB59ZuD/YVwKWRpVfRpzhirVxfAr36E5770/JDqDx4=
  private key: (hidden)
  listening port: 51820

peer: AOIzLd2C71DtY8DWgUfuMllRNa0iR1O3tO2WbFO7ICU=
  endpoint: 172.17.0.1:33090
  allowed ips: 172.16.16.10/32
  latest handshake: 14 seconds ago
  transfer: 4.85 KiB received, 2.50 KiB sent
```

### Test access to private Nginx

From the Wireguard client , access the Nginx deployed previously : 

```
k exec -it -n wireguard deployment/wireguard -- curl 172.17.0.5
Defaulted container "wireguard" out of: wireguard, nginx, wireguard-template-replacement (init)
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

From the Sidecar application of Wireguard client : 
```
k exec -it -n wireguard deployment/wireguard --container nginx -- curl 172.17.0.5
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Edit the secret with a dummy IP as the Wireguard endpoint : 

```
---
apiVersion: v1
kind: Secret
metadata:
  name: wireguard
  namespace: wireguard
type: Opaque
stringData:
  wg0.conf.template: |
    [Interface]
    PrivateKey = oG2sa0u9qJGWGC8+vtXRsLtI0IxXKtaYzGlpPzqD91k=
    Address = 172.16.16.10/20
    DNS = 1.1.1.1

    [Peer]
    PublicKey = CSB59ZuD/YVwKWRpVfRpzhirVxfAr36E5770/JDqDx4=
    AllowedIPs = 0.0.0.0/0
    Endpoint = 12.12.12.12:51820
    PersistentKeepalive = 25
```

Restart the deployment of the pod `k rollout restart deployment -n wireguard wireguard'`

Now the access does not work : 

```
k exec -it -n wireguard deployment/wireguard --container nginx  -- curl 172.17.0.5

curl: (28) Failed to connect to 172.17.0.5 port 80: Connection timed out
command terminated with exit code 28
```
