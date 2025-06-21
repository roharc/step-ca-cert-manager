# How To: setup step CA and integrate into kubernetes with cert-manager  

> :exclamation: This is a How-To for demo purpose only. It summarizes the basic steps to set up an acme provider with step-ca and integrate this into kubernetes with cert-manager.


## 1. Setup CA and ACME provider with step CA  

#### Download and install packages  
https://smallstep.com/docs/step-ca/installation/  

```
wget https://dl.smallstep.com/cli/docs-ca-install/latest/step-cli_amd64.deb
sudo dpkg -i step-cli_amd64.deb

wget https://dl.smallstep.com/certificates/docs-ca-install/latest/step-ca_amd64.deb
sudo dpkg -i step-ca_amd64.deb
```

#### Setup service user   

https://smallstep.com/docs/step-ca/certificate-authority-server-production/#create-a-service-user-to-run-step-ca

```
sudo useradd --system --home /etc/step-ca --shell /bin/false step
```  

#### Grant capability to bind portnumbers less than 1024 to step-ca binary

```
sudo setcap CAP_NET_BIND_SERVICE=+eip $(which step-ca)
```  

#### Initialize CA

```
step ca init
```

```code
✔ Deployment Type: Standalone
What would you like to name your new PKI?
✔ (e.g. Smallstep): pki-demo
What DNS names or IP addresses will clients use to reach your CA?
✔ (e.g. ca.example.com[,10.1.2.3,etc.]): pki-demo,pki-demo.home.arpa,127.0.0.1
What IP and port will your new CA bind to? (:443 will bind to 0.0.0.0:443)
✔ (e.g. :443 or 127.0.0.1:443): :443
What would you like to name the CA's first provisioner?
✔ (e.g. you@smallstep.com): first
Choose a password for your CA keys and first provisioner.
✔ [leave empty and we'll generate one]:
```

#### Move step-ca to `/etc/step-ca` and change ownership

```
sudo mkdir /etc/step-ca
sudo mv ~/.step/* /etc/step-ca
sudo chown -R step:step /etc/step-ca
```

#### Modify paths in config files

```
sudo sed 's/home\/xxx\/.step/etc\/step-ca/g' /etc/step-ca/config/ca.json
sudo sed 's/home\/xxx\/.step/etc\/step-ca/g' /etc/step-ca/config/defaults.json
```
(use `sed -i` to actually write the files after checking)

#### Create passwordfile and change ownership

```
sudo vi /etc/step-ca/password.txt
```
```
sudo chown step:step /etc/step-ca/password.txt
```

#### Create unitfile, start and check service

https://smallstep.com/docs/step-ca/certificate-authority-server-production/#running-step-ca-as-a-daemon
```
sudo vi  /etc/systemd/system/step-ca.service
```
```
sudo systemctl enable --now step-ca && sudo systemctl status step-ca
```

#### Add ACME provisioner and restart service

```
sudo STEPPATH=/etc/step-ca step ca provisioner add acme --type acme --ca-url https://127.0.0.1 --root /etc/step-ca/certs/root_ca.crt
```
```
sudo systemctl restart step-ca
```
```
sudo STEPPATH=/etc/step-ca step ca provisioner list --ca-url https://127.0.0.1
```

#### fetch CA Bundle for k8s integration

```
sudo cat /etc/step-ca/certs/intermediate_ca.crt /etc/step-ca/certs/root_ca.crt > ca_bundle.crt && chown xxx:xxx ca_bundle.crt
```

## 2. Integrate PKI into kubernetes with cert-manager
### 2.1 adding CA to k8s node system trust store and request test certificate

> :warning: **Prerequisites:**: Kubernetes Node with helm installed

#### scp CA Bundle from pki node

```
scp pki-demo:ca_bundle.crt .
```

#### try to curl CA

```
curl https://pki-demo.home.arpa
```

#### add CA to system trust store

```
sudo cp ca_bundle.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates
```

#### curl CA once more to confirm trust

```
curl https://pki-demo.home.arpa
```

#### install step-cli and request cert as a test

```
wget https://dl.smallstep.com/cli/docs-ca-install/latest/step-cli_amd64.deb
sudo dpkg -i step-cli_amd64.deb
```
```
sudo step ca certificate k3s-demo.home.arpa k3s-demo.crt k3s-demo.key --acme https://pki-demo.home.arpa/acme/acme/directory --san k3s-demo.home.arpa --san k3s-demo
```

### 2.2 deploy app for testing

#### apply manifests
```
k apply -f ~/step-ca-cert-manager/application_manifests/nginx-pod.yaml
k apply -f ~/step-ca-cert-manager/application_manifests/nginx-service.yaml
```

#### retrieve CLUSTER-IP 
```
CLUSTER_IP=$(k get svc nginx -o jsonpath='{.spec.clusterIP}')
```

#### test application by curling the service
```
curl http://$CLUSTER_IP:8088
```

### 2.3 install cert-manager

https://cert-manager.io/docs/installation/helm/

default values from here:  
https://artifacthub.io/packages/helm/cert-manager/cert-manager

depending on environment it may be necessary to configure `podDnsPolicy: "Default"` in values which is NOT the default:
https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy

#### add helm repo

```
helm repo add jetstack https://charts.jetstack.io --force-update
```
#### install cert-manager

```
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.2 \
  --values=certmgr-values.yaml \
  --set crds.enabled=true
```

### 2.4 setup with ingress controller

> :warning: **setup with ingress controller (2.4) and gateway-api (2.5) are exclusive**:  
with a default config you cannot have both at the same time as they're binding same ports!

#### install ingress-nginx

https://artifacthub.io/packages/helm/ingress-nginx/ingress-nginx?modal=install

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```
```
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace --version 4.12.1
```

#### deploy and inspect a clusterissuer

create value for clusterissuers `.spec.acme.caBundle`:

```
base64 ca_bundle.crt | tr -d "\n"
```

then edit manifest and apply file:

```
k apply -f ~/step-ca-cert-manager/ingress_manifests/clusterissuer.yaml
```
```
k get clusterissuer
```
```
k apply -f ~/step-ca-cert-manager/ingress_manifests/ingress.yaml
```
```
k get ingress nginx
```

inspect the respective kubernetes resources: 

```
clusterissuers
orders
challenges
certificates
secrets
ingress
```

the generated secret is of type `Opaque` first, changes to type `kubernetes.io/tls` after the http01 challenge has succeded.  
delete the secret to check auto-renewal and initiate a new challenge

#### inspect the actual certificate retrieved from the secret

```
k get secret nginx-k3s-demo-home-arpa -ojson | jq '.data."tls.crt"' | tr -d '"' | base64 -d | openssl x509 --text --noout
```

### 2.5 setup with gateway-api

> :warning: **setup with ingress controller (2.4) and gateway-api (2.5) are exclusive**:
with a default config you cannot have both at the same time as they're binding same ports!
 
if you deployed an ingress controller as in step 2.4 you should clean up everything before continuing

cleanup:
```
k delete clusterissuer acme-pki-demo
k delete ingress nginx 
k delete secret nginx-k3s-demo-home-arpa
helm delete ingress-nginx --namespace ingress-nginx
```

#### 2.51 enable gateway-api for certmanager
https://cert-manager.io/docs/usage/gateway/

```
helm upgrade --install cert-manager jetstack/cert-manager --namespace cert-manager \
  --version v1.17.2 \
  --set config.apiVersion="controller.config.cert-manager.io/v1alpha1" \
  --set config.kind="ControllerConfiguration" \
  --set config.enableGatewayAPI=true
```

#### install crds and nginx-gateway-fabric:

```
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.6.2" | kubectl apply -f -
```
```
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric --create-namespace -n nginx-gateway
```

#### 2.52 apply manifests
##### don't forget to label default namespace:
```
gateway-access: "true"
```

first edit `~/step-ca-cert-manager/gateway_manifests/clusterissuer.yaml` and insert `.spec.acme.caBundle` value with the output of `base64 ca_bundle.crt | tr -d "\n"`


```
k apply -f  ~/step-ca-cert-manager/gateway_manifests/
```
