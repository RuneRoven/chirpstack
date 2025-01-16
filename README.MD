# Helm chart for chirpstack 4
to add repo use: https://runeroven.github.io/chirpstack/

Complete settings with default values
The current Values file consists of a minimal setup.  
Add the relevant settings to the values for your setup.  
If not using the settings the config file will deploy to default.

Chart will deploy with minimal settings but all parents in the config section for chirpstack and gb needs to exists.

Use the region section in chirpstack to select the regions you want to enable. The frequency settings will be derived from preconfigured json files.
For a complete list of avalible regions check configurations folder or visit [chirpstack](https://www.chirpstack.io/) where you also find a description of all settings.

Refer to the README_VALUES file in this repo to see all avalible settings.

If using United Manufacturing Hub and their postgresql, there is a small issue with the CA-certificates that needs to be fixed.  
This is due to the fact that Chirpstack > v4.6 uses rustls which has a really strict use of ssl.  
This does now allow the end certificate and root certificate to be the same.  
To solve this we need to reissue a new certificate adding the hostnames for postgresql and replace the certificate for postgresql.

```bash
# exec in to the timescaledb shell
#
kubectl exec -i -t -n united-manufacturing-hub united-manufacturing-hub-timescaledb-0 -c timescaledb -- sh -c "clear; (bash || ash || sh)"

# copy the current certificates and key.
cp /etc/certificate/tls.crt /home/postgres/tls.crt
cp /etc/certificate/tls.key /home/postgres/tls.key

# create a config file to the new certificate
# use any common name and ensure that the commonName_default is not the same as the root ca
cat <<EOT >> openssl-san.cnf
[ req ]
distinguished_name = req_distinguished_name
req_extensions = v3_req

[ req_distinguished_name ]
# Common Name can be left blank if you don't need it
commonName = united-manufacturing-hub
commonName_default = united-manufacturing-hub-leaf

[ v3_req ]
# Extensions for the certificate request
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
basicConstraints = CA:FALSE
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = *.united-manufacturing-hub.svc.cluster.local
DNS.2 = *.svc.cluster.local
DNS.3 = *.cluster.local
IP.1 = 192.168.1.123
EOT

# Issue a new certificate, just press enter when it promts for a name
openssl req -new -key tls.key -out end-entity.csr -config openssl-san.cnf 
openssl x509 -req -in end-entity.csr -CA tls.crt -CAkey tls.key -CAcreateserial -out pgTls.crt -days 9999 -sha256 -extfile openssl-san.cnf -extensions v3_req

# Verify the certificate chain
openssl verify -CAfile tls.crt pgTls.crt

# Display the new and root certificate
cat pgTls.crt
cat tls.crt
```yaml

Copy the pgTls.crt and replace the tls.crt in the united-manufacturing-hub-certificate secret.
Restart the timescale pod.
Use the tls.crt as the certificate for postgres in chirpstack.


