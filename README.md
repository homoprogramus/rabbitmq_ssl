# rabbitmq ssl #

How to Rabbitmq with SSL on Python

## 1.RabbitMQ Server ##

Once installed you have to add the plugin: rabbitmq_auth_mecanism_ssl

```rabbitmq-plugins enable rabbitmq_auth_mecanism_ssl```

### RabbitMQ configuration ###

In /etc/rabbitmq, in the file rabbitmq.conf

```
[
  {ssl, [{versions, ['tlsv1.2', 'tlsv1.1']}]},
  {rabbit, [
    {ssl_listeners, [5671]},
    {ssl_options, [
      {cacertfile,"/home/joel/testca/cacert.pem"},
      {certfile,"/home/joel/server/cert.pem"},
      {keyfile,"/home/joel/server/key.pem"},
      {password, "MySecretPassword"},
      {verify, verify_peer},
      {fail_if_no_peer_cert, false}
    ]},
    {auth_mechanisms, ['PLAIN', 'AMQPLAIN', 'EXTERNAL']},
    {ssl_cert_login_from, common_name}
  ]}
].
```

The password is the one used to create the certificates.

### Certificate generation  ###

Follow these steps:

```
mkdir testca
cd testca
mkdir certs private
chmod 700 private
echo 01 > serial
touch index.txt
```

testca/openssl.cnf
```
[ ca ]
default_ca = testca

[ testca ]
dir = .
certificate = $dir/cacert.pem
database = $dir/index.txt
new_certs_dir = $dir/certs
private_key = $dir/private/cakey.pem
serial = $dir/serial

default_crl_days = 7
default_days = 365
default_md = sha256

policy = testca_policy
x509_extensions = certificate_extensions

[ testca_policy ]
commonName = supplied
stateOrProvinceName = optional
countryName = optional
emailAddress = optional
organizationName = optional
organizationalUnitName = optional
domainComponent = optional

[ certificate_extensions ]
basicConstraints = CA:false

[ req ]
default_bits = 2048
default_keyfile = ./private/cakey.pem
default_md = sha256
prompt = yes
distinguished_name = root_ca_distinguished_name
x509_extensions = root_ca_extensions

[ root_ca_distinguished_name ]
commonName = hostname

[ root_ca_extensions ]
basicConstraints = CA:true
keyUsage = keyCertSign, cRLSign

[ client_ca_extensions ]
basicConstraints = CA:false
keyUsage = digitalSignature,keyEncipherment
extendedKeyUsage = 1.3.6.1.5.5.7.3.2

[ server_ca_extensions ]
basicConstraints = CA:false
keyUsage = digitalSignature,keyEncipherment
extendedKeyUsage = 1.3.6.1.5.5.7.3.1
```

Server:

```
cd testca
openssl req -x509 -config openssl.cnf -newkey rsa:2048 -days 365 \
    -out cacert.pem -outform PEM -subj /CN=MyTestCA/ -nodes
openssl x509 -in cacert.pem -out cacert.cer -outform DER
cd ..
ls
testca
cd server
openssl genrsa -out key.pem 2048
openssl req -new -key key.pem -out req.pem -outform PEM \
    -subj /CN=$(hostname)/O=server/ -nodes
cd ../testca
openssl ca -config openssl.cnf -in ../server/req.pem -out \
    ../server/cert.pem -notext -batch -extensions server_ca_extensions
cd ../server
openssl pkcs12 -export -out keycert.p12 -in cert.pem -inkey key.pem \
    -passout pass:MySecretPassword
```

Client:

You have to replace the $(hostname) by a rabbitmq username

```
cd ..
ls
mkdir client
cd client
openssl genrsa -out key.pem 2048
openssl req -new -key key.pem -out req.pem -outform PEM \
    -subj /CN=$(hostname)/O=client/ -nodes
cd ../testca
openssl ca -config openssl.cnf -in ../client/req.pem -out \
    ../client/cert.pem -notext -batch -extensions client_ca_extensions
cd ../client
openssl pkcs12 -export -out keycert.p12 -in cert.pem -inkey key.pem \
    -passout pass:MySecretPassword
```

You have to transfert client/key.pem, client/cert.pem and testca/cacert.pem to the client.

## 2.Python Client ##

A simple python client using authentication:

```
import ssl
import pika
import logging
from pika.credentials import ExternalCredentials

logging.basicConfig(level=logging.INFO)

cp = pika.ConnectionParameters(
    ssl=True,
    ssl_options=dict(
        ssl_version=ssl.PROTOCOL_TLSv1_2,
        ca_certs='cacert.pem',
        keyfile='key.pem',
        certfile='cert.pem',
        cert_reqs=ssl.CERT_REQUIRED
    ),
    host='128.178.117.77',
    port=5671,
    virtual_host='/',
    credentials=ExternalCredentials()
)

conn = pika.BlockingConnection(cp)
ch = conn.channel()
print(ch.queue_declare("sslq"))
ch.publish("", "sslq", "abc")
print(ch.basic_get("sslq"))
```
