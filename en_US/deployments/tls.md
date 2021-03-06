# TLS/SSL Configuration

## Introduction to TLS/SSL
As a security protocol based on the public key algorithm of modern cryptography, TLS/SSL can ensure transmission security on computer communication networks.

Security benefits from TLS/SSL

- Strong authentication. When TLS is used to establish a connection, both sides of the communication can check each other's identity. In practice, a very common way of identity check is to check the X.509 digital certificate held by the other party. Such digital certificates are usually issued by a trusted organization and cannot be forged.
- Confidentiality. Each session of TLS communication is encrypted by session key, which is generated by negotiation between two parties. No third party can be aware of the content of the communication. Even if the key of one session is  leaked, the security of other sessions will not be affected.
- Integrity. The data in encrypted communication is difficult to be tampered without being discovered.

EMQ X Cloud supports two kinds of authentication: one-way authentication and two-way authentication<br>
One-way authentication requires:

- Server certificate
- Certificate chain
- Private key

Two-way authentication requires:
- Server certificate
- Certificate chain
- Private key
- Client CA certificate

## Certificate restrictions
- The certificate must specify a cryptographic algorithm and a key size. The following algorithms are supported by EMQ X Cloud:
  - 1024-bit RSA (RSA_1024)
  - 2048-bit RSA (RSA_2048)
- The certificate must be an SSL/TLS X.509 version 3 certificate. It must contain a public key, the fully qualified domain name (FQDN) or IP address for your website, and information about the issuer. The certificate can be self-signed by your private key or by the private key of an issuing CA. If your certificate is signed by a CA, you must include the certificate chain when you import your certificate.
- The certificate must be valid. and it cannot be imported before its validity period begins or within `180` days before its validity period begins.
- The certificate, private key and certificate chain must be PEM-encoded.
- The private key must be unencrypted.
- The cryptographic algorithm of an imported certificate must match the algorithm of the signing CA. For example, if the signing CA key type is RSA, then the certificate key type must also be RSA.

### Format description
#### Certificate format
```
-----BEGIN CERTIFICATE-----
Base64–encoded certificate
-----END CERTIFICATE-----
```

#### Certificate chain format
> Do not copy the certificate into the certificate chain

```
-----BEGIN CERTIFICATE-----
Base64–encoded certificate
-----END CERTIFICATE-----
```

#### Private key format
```
-----BEGIN RSA PRIVATE KEY-----
Base64–encoded private key
-----END RSA PRIVATE KEY-----
```

### Self-signed certificate
Please make sure you have installed [OpenSSL](https://www.openssl.org/)
#### Create a server CA certificate
```bash
openssl req \
    -new \
    -newkey rsa:2048 \
    -days 3650 \
    -nodes \
    -x509 \
    -subj "/C=CN/ST=XX/L=XX/O=EMQ X Cloud/CN=CA" \
    -keyout cloud-ca.key \
    -out cloud-ca.crt
```

The generated `cloud-ca.crt` is the server CA certificate, which is used to sign and issue the server certificate and verify the server certificate when connecting.

#### Create a server certificate

Generate private key
```bash
openssl genrsa -out cloud.key 2048
```
Create the `openssl.cnf` file and modify `CONNECT_ADDRESS` of `alt_names` to the connection address of the deployment details page. `req_distinguished_name` can be modified as required.

```
[req]
default_bits  = 2048
distinguished_name = req_distinguished_name
req_extensions = req_ext
x509_extensions = v3_req
prompt = no
[req_distinguished_name]
countryName = CN
stateOrProvinceName = XX
localityName = XX
organizationName = XX
commonName = XX
[req_ext]
subjectAltName = @alt_names
[v3_req]
subjectAltName = @alt_names
[alt_names]
DNS.1 = CONNECT_ADDRESS
```

Generate certificate request file `cloud.csr`
```bash
openssl req -new -key cloud.key -config openssl.cnf -out cloud.csr 
```

Sign the server with a CA certificate
```bash
openssl x509 -req -days 3650 -in cloud.csr -CA cloud-ca.crt -CAkey cloud-ca.key -CAcreateserial -out cloud.crt  -extensions v3_req -extfile openssl.cnf 
```

The above steps mainly generated the following files:
- cloud.crt: server certificate
- cloud.key: private key

#### Create a client certificate (two-way authentication)
```bash
# Create a client CA certificate
openssl req \
    -new \
    -newkey rsa:2048 \
    -days 3650 \
    -nodes \
    -x509 \
    -subj "/C=CN/ST=XX/L=XX/O=EMQ X Cloud/CN=Client CA" \
    -keyout client-ca.key \
    -out client-ca.crt

# Create client key
openssl genrsa -out client.key 2048
# Create client request file
openssl req -new -key client.key -out client.csr -subj "/C=CN/ST=XX/L=XX/O=EMQ X Cloud/CN=Client"
# Sign the client with a CA certificate
openssl x509 -req -days 3650 -in client.csr -CA client-ca.crt -CAkey client-ca.key -CAcreateserial -out client.crt
```
The above steps mainly generated the following files:
- client-ca.crt: client CA certificate
- client.key: client private key
- client.crt: client certificate

## Create a certificate
1. Create a deployment and wait until the `running` state.
2. Log in  [EMQ X Cloud console](<https://cloud.emqx.io/console>).
3. Enter the deployment details page, click the `+TLS/SSL configuration` button to configure the certificate content, you can upload the file or directly fill in the certificate content
   - Authentication type:
     - One-way authentication: only the client verify the server certificate
     - Two-way authentication: the client and server mutually verify the certificate
   - Certificate: server certificate
   - Certificate chain: certificate chain is usually provided by a third-party organization when issuing a certificate
   - Private key: private key
   - CA certificate: when selecting two-way authentication, you need to provide the client's CA certificate
4. After filling in, click "Confirm".

![tls](../_assets/deployments/tls.png)

## Test connection
Before testing, please make sure to create the authentication information, refer to [authentication](./users_and_acl.md).<br>
You can use [MQTTX](<https://mqttx.app/>) for connecting and testing.<br>
We use MQTTX for testing:

- Create a new connection, enter the Name, and the Client ID is randomly generated.
- Select Host, fill in the connection address and port of the deployment
  - If you choose SSL connection, select `mqtts://` and `8883` ports
  - If you choose Websocket with SSL, select `wss://` and `8084` ports
- Enter the created authentication information: username and password
- Select true for SSL/TLS
- Select certificate
  - For certificates from third-party organizations, there is no need to provide CA certificates
  - For self-signed certificate, server CA certificate is required. If two-way authentication is selected,  client certificate and private key are required.
- Turn on strict mode
- Click "connect"

![mqttx_tls](../_assets/deployments/mqttx_tls.png)

## Delete a certificate
Deletion of the certificate will disconnect the client's connection to `8883` and `8084`. Please make sure this will not affect your business.

1. Log in [EMQ X Cloud console](<https://cloud.emqx.io/console>).
2. Enter the deployment details and click the delete button of the certificate in the `TLS/SSL configuration` section.
3. Click "Confirm" in the dialog box to complete the deletion.

## Common problem

1. The certificate content contains multiple certificates
   
   The purchased certificate contains intermediate certificate. When the certificate is opened in text form, multiple certificates are in the order of user certificate-intermediate certificate -root certificate. Generally speaking, the certificate contains user certificate and multiple intermediate certificates. You need to separate the user certificate from the intermediate certificate and fill the intermediate certificate into the certificate chain.
   
   ``` 
   -----BEGIN CERTIFICATE-----
   
   user certificate
   
   -----END CERTIFICATE-----
   
   -----BEGIN CERTIFICATE-----
   
   intermediate certificate
   
   -----END CERTIFICATE-----
   
   -----BEGIN CERTIFICATE-----
   
   root certificate
   
   -----END CERTIFICATE-----
   ```
   
2. Missing certificate chain
  
    Certificate chain completion: https://myssl.com/chain_download.html