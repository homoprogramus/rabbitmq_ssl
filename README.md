# rabbitmq ssl #

How to Rabbitmq with SSL on Python

## 1.RabbitMQ Server ##

Once installed you have to add the plugin: rabbitmq_auth_mecanism_ssl

```rabbitmq-plugins enable rabbitmq_auth_mecanism_ssl```

### 1.RabbitMQ configuration ###

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

## 2.Python Client ##



