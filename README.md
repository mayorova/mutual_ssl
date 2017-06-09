
# Securing traffic to upstream servers with client certificates

Info: https://www.nginx.com/resources/admin-guide/nginx-https-upstreams/

## Creating and Signing Your Certs

Source: http://nategood.com/client-side-certificate-authentication-in-ngi

This is SSL, so you'll need an cert-key pair for you/the server, the api users/the client and a CA pair. You will be the CA in this case (usually a role played by VeriSign, thawte, GoDaddy, etc.), signing your client's certs. There are plenty of tutorials out there on creating and signing certificates, so I'll leave the details on this to someone else and just quickly show a sample here to give a complete tutorial. NOTE: This is just a quick sample of creating certs and not intended for production.

### Create the CA Key and Certificate for signing Client Certs
```
openssl genrsa -des3 -out ca.key 4096
openssl req -new -x509 -days 365 -key ca.key -out ca.crt
```

### Create the Server Key, CSR, and Certificate
```
openssl genrsa -des3 -out server.key 1024
openssl req -new -key server.key -out server.csr
```

### Self-sign the certificate with our CA cert
```
openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out server.crt
```

### Create the Client Key and CSR
```
openssl genrsa -des3 -out client.key 1024
openssl req -new -key client.key -out client.csr
```

### Sign the client certificate with our CA cert
```
openssl x509 -req -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out client.crt
```

## Testing

### Making calls to the API backend directly

```
curl -v -k "https://127.0.0.1:8443/hello/world"
```

```
curl -v -s -k --key client.key --cert client.crt "https://127.0.0.1:8443/hello/world"
```

#### On macOS

Issue:

```
curl -v -s -k --key client.key --cert client.crt "https://127.0.0.1:8443/hello/world"
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to 127.0.0.1 (127.0.0.1) port 8443 (#0)
* WARNING: SSL: CURLOPT_SSLKEY is ignored by Secure Transport. The private key must be in the Keychain.
* WARNING: SSL: Certificate type not set, assuming PKCS#12 format.
* SSL: Can't load the certificate "client.crt" and its private key: OSStatus -25299
* Curl_http_done: called premature == 0
* Closing connection 0
```

To solve this bundle the keys into pkcs12:

```
openssl pkcs12 -export -in client.crt -inkey client.key -out client.p12 -name "client"
```

Then you can make the curl like this:

```
curl -v -s -k --cert client.p12:pass "https://127.0.0.1:8443/hello/world"
```

### Verifying that proxy works the same way

```
curl -v -s "http://127.0.0.1:8080/hello/world"
```
