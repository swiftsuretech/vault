# Setting up Hashicorp Vault in DKP

## Get the Helm charts:
We'll install vault and consul using helm charts. Consul will run a simple cluster as the back end. First we'll install the repos and ensure we get the latest version:

Add the hashicorp repo:

```
helm repo add hashicorp https://helm.releases.hashicorp.com
```
List the versions. Not which version you wish to install:

```
helm search repo hashicorp/consul --versions
helm search repo hashicorp/vault --versions
```


## Generate a keypair:

Make a cert directory:

```
mkdir certs && cd certs
```
## Generate TLS Certs:
Set some env variables:
```
# Set some env variables:
export SERVICE=vault-server-tls
export NAMESPACE=vault-namespace
export SECRET_NAME=vault-server-tls
```
Generate our private kay:
```
openssl genrsa -out vault.key 2048
```
Set our certificate parameters:
```
cat <<EOF >csr.conf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = ${SERVICE}
DNS.2 = ${SERVICE}.${NAMESPACE}
DNS.3 = ${SERVICE}.${NAMESPACE}.svc
DNS.4 = ${SERVICE}.${NAMESPACE}.svc.cluster.local
IP.1 = 127.0.0.1
EOF
```
Create the CSR:
```
openssl req -new -key vault.key \
    -subj "/O=system:nodes/CN=system:node:${SERVICE}.${NAMESPACE}.svc" \
    -out ${TMPDIR}/server.csr \
    -config ${TMPDIR}/csr.conf
```
Create a CSR kubernetes manifest:
```
cat <<EOF >csr.yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: ${CSR_NAME}
spec:
  groups:
  - system:authenticated
  request: $(cat server.csr | base64 | tr -d '\r\n')
  signerName: kubernetes.io/kubelet-serving
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
```
Apply the manifest:
```
kubectl create -f csr.yaml
```
Now check that the CSR is there and pending:
```
kubectl get csr
```
Approve the CSR:
```
kubectl certificate approve ${CSR_NAME}
```
Now check that the CSR is approved:
```
kubectl get csr
```
If it's been approved we can extract the certificate and place it into a file:
```
kubectl get csr ${CSR_NAME} -o jsonpath='{.status.certificate}' | base64 -d > vault.crt   
```
Create our Namespace:
```
kubectl create namespace ${NAMESPACE}
```
Generate our secret based upon certificates:
```
kubectl create secret generic ${SECRET_NAME} \
    --namespace ${NAMESPACE} \
    --from-file=vault.key=vault.key \
    --from-file=vault.crt=vault.crt \
    --from-file=vault.ca=vault.ca

```