* Get server SSL cert chain:
```sh
openssl s_client -showcerts -connect ${HOST}:${PORT}
```
* Get server SSL cert chain WITH SNI:
```sh
openssl s_client -showcerts -connect ${HOST}:${PORT} -servername ${DOMAIN:-${HOST}}
```
* List keystore contents: 
```sh
keytool -list -v -storetype jks -keystore keystore.jks
```
* Print pem certificate content:  
```sh
openssl x509 -in cert.pem -text
```
OR
```sh
keytool -printcert -file cert.pem
```
* Print CSR details:
```sh
openssl req -in ingress.csr -noout -text
```
* Convert DER to PEM:
```sh
openssl x509 -inform der -in cert.cer -out cert.pem
```
* Convert PKCS7B to PEM:
```sh
openssl pkcs7 -print_certs -in cert.p7b -out cert.pem
```
* Check if server supports TLS version:
```sh
openssl s_client -connect ${HOST}:${PORT} -tls1_3
```
* Check a private key:
```sh
openssl rsa -check -in key.pem
```
* Test Mutal SSL with client cert/key
```sh
openssl s_client -connect ${HOST}:${PORT} -servername ${HOST} -tls1_2 -key ${KEY_FILE} -cert ${CERT_FILE} -CAfile ${CA_CHAIN_FILE}
```
* List supported ciphers:
```sh
nmap --script ssl-enum-ciphers -p ${PORT} ${HOST}
```
* Verify CA certificate trust chain:
```sh
openssl verify -CAfile rootCA.crt -untrusted pki.crt cert.pem
```
* Verify a server's CA certificate:
```sh
openssl s_client -CAfile rootCA.crt -connect ${HOST}:${PORT}
```
* Generate self-signed cert and key:
```sh
openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out cert.pem -subj "/CN=${DOMAIN}"
```
* Import certificate into truststore/keystore
```sh
keytool -importcert -noprompt -file ${CERT} -alias ${ALIAS_NAME} -keystore ${KEYSTORE} -storepass changeit
```
* Verify key/cert pair match (md5 of modulus should match)
```sh
openssl x509 -noout -modulus -in cert.pem | openssl md5 && openssl rsa -noout -modulus -in key.pem | openssl md5
```
* Create jks keystore
```sh
KEYSTORE_NAME="keystore" && \
openssl pkcs12 -export -in "${CERT_FILE}" -inkey "${KEY_FILE}" -out "${KEYSTORE_NAME}.p12" -name "${ALIAS_NAME}" -password "pass:${PASSWORD}" && \
keytool -importkeystore -noprompt -srckeystore "${KEYSTORE_NAME}.p12" -srcstoretype pkcs12 -srcstorepass "${PASSWORD}" -destkeystore "${KEYSTORE_NAME}.jks" -deststoretype jks -deststorepass "${PASSWORD}" && \
rm "${KEYSTORE_NAME}.p12"
```
* Generate private key and CSR:
  - DOMAIN
  - STATE
  - CITY
```sh
STATE=""
CITY=""
DOMAIN=""

openssl req -new -sha256 -nodes -out ingress.csr \
-newkey rsa:2048 -keyout ingress.key -config \
<(echo '[req]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn
default_days   = 1095

[ dn ]
C=US
ST='${STATE}'
L='${CITY}'
O=Organization
OU=Organization Unit
CN='${DOMAIN}'

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1='${DOMAIN}'')
```
* Fulfill a CSR using CA cert/key
  - DOMAIN
  - CA_KEY
  - CA_CERT
```sh
DIR=${DIR:-.} && mkdir -p ${DIR}/tmp/certs ${DIR}/tmp/crl && \
echo 1000 > ${DIR}/tmp/serial && \
touch ${DIR}/tmp/index.txt ${DIR}/tmp/index.txt.attr && \
openssl ca -batch -keyfile ${CA_KEY} -cert ${CA_CERT} -in ${CSR_FILE} -out ${DIR}/signed-server-cert.pem -notext \
-subj "/CN=provision.np.wce.ibm.com/C=US/ST=Massachusetts/L=Littleton/O=IBM/OU=toscana" -config \
<(echo '[ ca ]
default_ca     = CA_default

[ CA_default ]
dir             = '$DIR'
certs           = $dir/tmp/certs
crl_dir         = $dir/tmp/crl
database        = $dir/tmp/index.txt
new_certs_dir   = $dir/tmp/certs
certificate     = '$CA_CERT'
serial          = $dir/tmp/serial
crl             = $dir/tmp/crl/crl.pem
private_key     = '$CA_KEY'
RANDFILE        = $dir/tmp/.rnd
nameopt         = default_ca
certopt         = default_ca
policy          = policy_match
x509_extensions = req_ext
default_days    = 1095
default_md      = sha256

[ policy_match ]
countryName            = optional
stateOrProvinceName    = optional
organizationName       = optional
organizationalUnitName = optional
commonName             = supplied
emailAddress           = optional

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1=example.com')
```
