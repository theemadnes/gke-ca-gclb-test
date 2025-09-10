# gke-ca-gclb-test
create cert chain and add leaf cert to GKE cluster and manage via GKE CRs

for company testco.xyz 

### create cert chain

```
cd root/ca

# following https://openssl-ca.readthedocs.io/en/latest/create-the-root-pair.html
openssl genrsa -aes256 -out private/ca.key.pem 4096
chmod 400 private/ca.key.pem

openssl req -config openssl.cnf -key private/ca.key.pem -new -x509 \
    -days 7300 -sha256 -extensions v3_ca -out certs/ca.cert.pem

# Common Name []:TestCo Root CA
# Country Name (2 letter code) [XX]:US
# State or Province Name [MyState]:Illinois
# Locality Name []:
# Organization Name [MyOrg]:TestCo
# Organizational Unit Name []:TestCo Certificate Authority
# Email Address []:

# verify cert 
openssl x509 -noout -text -in certs/ca.cert.pem # all root certificates are self-signed

# create intermediate key
openssl genrsa -aes256 -out intermediate/private/intermediate.key.pem 4096

# create CSR
openssl req -config intermediate/openssl.cnf -new -sha256 \
    -key intermediate/private/intermediate.key.pem \
    -out intermediate/csr/intermediate.csr.pem

# Common Name []:TestCo Intermediate CA
# Country Name (2 letter code) [XX]:US
# State or Province Name [MyState]:Illinois
# Locality Name []:
# Organization Name [MyOrg]:TestCo
# Organizational Unit Name []:TestCo Certificate Authority
# Email Address []:

# create intermediate cert
openssl ca -config openssl.cnf -extensions v3_intermediate_ca \
    -days 3650 -notext -md sha256 -in intermediate/csr/intermediate.csr.pem \
    -out intermediate/certs/intermediate.cert.pem

# verify intermediate
openssl x509 -noout -text -in intermediate/certs/intermediate.cert.pem
openssl verify -CAfile certs/ca.cert.pem intermediate/certs/intermediate.cert.pem

# create cert chain file
cat intermediate/certs/intermediate.cert.pem certs/ca.cert.pem > \
    intermediate/certs/ca-chain.cert.pem

chmod 444 intermediate/certs/ca-chain.cert.pem

# begin server cert creation process
openssl genrsa -out intermediate/private/testco.xyz.key.pem 2048 # omitting the -aes256 flag to avoid password
chmod 400 intermediate/private/testco.xyz.key.pem

openssl req -config intermediate/openssl.cnf \
    -key intermediate/private/testco.xyz.key.pem \
    -new -out intermediate/csr/testco.xyz.csr.pem \
    -addext "subjectAltName = DNS:testco.xyz,DNS:www.testco.xyz,DNS:whereami.testco.xyz"

openssl ca -config intermediate/openssl.cnf -extensions server_cert \
    -days 375 -notext -md sha256 -in intermediate/csr/testco.xyz.csr.pem \
    -out intermediate/certs/testco.xyz.cert.pem
chmod 444 intermediate/certs/testco.xyz.cert.pem

# verify the server cert
openssl x509 -noout -text -in intermediate/certs/testco.xyz.cert.pem

# verify a valid chain of trust
openssl verify -CAfile intermediate/certs/ca-chain.cert.pem \
    intermediate/certs/testco.xyz.cert.pem
```

### create test app and gateway resources

To simplify things, just going to create everything in the same namespace.

```
cd ~/gke-ca-gclb-test # change directory to repo base for these commands

# create ns
kubectl create ns whereami-testco-xyz

# create whereami test app (from base dir of repo)
kubectl -n whereami-testco-xyz apply -k whereami/variant

# create TLS secret
kubectl -n whereami-testco-xyz create secret tls testco-xyz --cert=root/ca/intermediate/certs/testco.xyz.cert.pem --key=root/ca/intermediate/private/testco.xyz.key.pem

# create gateway resources
kubectl -n whereami-testco-xyz apply -f gateway/

# get IP address of load balancer
WHEREAMI_IP=$(kubectl -n whereami-testco-xyz get gateway external-http -o=jsonpath="{.status.addresses[0].value}")

# call the endpoint (after waiting for the load balancer to come up)
curl https://whereami.testco.xyz --resolve whereami.testco.xyz:443:$WHEREAMI_IP --cacert root/ca/intermediate/certs/ca-chain.cert.pem -s | jq
```

Example output:
```
$ curl https://whereami.testco.xyz --resolve whereami.testco.xyz:443:$WHEREAMI_IP --cacert root/ca/intermediate/certs/ca-chain.cert.pem -s | jq
{
  "cluster_name": "edge-to-mesh-01",
  "gce_instance_id": "6985173335383727699",
  "gce_service_account": "e2m-private-test-01.svc.id.goog",
  "host_header": "whereami.testco.xyz",
  "metadata": "whereami",
  "node_name": "gk3-edge-to-mesh-01-nap-1c08r7is-873c2a55-zktv",
  "pod_ip": "10.54.2.32",
  "pod_name": "whereami-5d6bdc8cf4-jj8lm",
  "pod_name_emoji": "🍴",
  "pod_namespace": "whereami-testco-xyz",
  "pod_service_account": "whereami",
  "project_id": "e2m-private-test-01",
  "timestamp": "2025-09-10T03:35:05",
  "zone": "us-central1-a"
}
```
