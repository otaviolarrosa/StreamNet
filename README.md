# StreamNet
A Kafka Client to easily connect with Kafka

# Pre-requisites
- The .NET 6 SDK should be installed before continuing.
- To use Apache Kafka, I recommend to use [docker](https://docs.docker.com/engine/install/) with [docker-compose](https://docs.docker.com/compose/).

# docker-compose and kafka_server_jaas.conf
To run a docker container with kafka, just use the docker-compose up command in the same folder of the .yml and .conf files.

| Branch | Status                                                                                                                                                                                         |                                                                            
|--------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| main   | [![main](https://github.com/otaviolarrosa/StreamNet/actions/workflows/publish.yml/badge.svg?branch=main&event=push)](https://github.com/otaviolarrosa/MassTransit/actions/workflows/publish.yml) | 


``` sh
(sudo) docker-compose up -d
```

``` yml
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    networks: 
      - broker-kafka
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
  
  kafka:
    image: confluentinc/cp-kafka:5.1.0
    networks: 
      - broker-kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_LISTENERS: SASL_PLAINTEXT://:9092
      KAFKA_ADVERTISED_LISTENERS: SASL_PLAINTEXT://localhost:9092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      ZOOKEEPER_SASL_ENABLED: "false"
      KAFKA_OPTS: "-Djava.security.auth.login.config=/etc/kafka/kafka_server_jaas.conf"
      KAFKA_INTER_BROKER_LISTENER_NAME: SASL_PLAINTEXT
      KAFKA_SASL_ENABLED_MECHANISMS: PLAIN
      KAFKA_SASL_MECHANISM_INTER_BROKER_PROTOCOL: PLAIN
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./kafka_server_jaas.conf:/etc/kafka/kafka_server_jaas.conf
      - ./data:/var/lib/kafka/data
    links:
      - zookeeper
  
  magic:
    image: digitsy/kafka-magic
    ports:
      - "8080:80"
    environment:
      KMAGIC_ALLOW_TOPIC_DELETE: "true"
      KMAGIC_ALLOW_SCHEMA_DELETE: "true" 
  
networks: 
  broker-kafka:
    driver: bridge
```
``` js
KafkaServer {
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="your_user"
  password="your_password"
  user_admin="your_password";
};
Client{};
```
# Initial setup
In your ASP.Net Core app, install the nuget package in [nuget.org](https://www.nuget.org/packages/StreamNet/)
and add the following configurations:

### appsettings.json
``` json
"Kafka": {
    "BootstrapServers": "localhost:9092",
    "SaslMechanism": "Plain",
    "SecurityProtocol": "SaslPlaintext",
    "Username": "your_user",
    "Password": "your_password"
}
```
### appsettings configuration fields

1. BootstrapServers: server where your kafka is running, with the port.<br>
2. SaslMechanism: Authorization mechanism used on server-side, this field accepts the following values:
   1. "GssApi"
   2. "Plain"
   3. "ScramSha256"
   4. "ScramSha512"
   5. "OAuthBearer"
3. SecurityProtocol: The security protocol used to authorize your user at the kafka broker-side, this field accepts the following values:
   1. Plaintext
   2. Ssl
   3. SaslPlaintext
   4. SaslSsl
4. Username: your username.
5. Pasword: your password.


## Producer

### Configuring application startup
At Startup.cs or Program.cs(in case of .net 6 or later), add the following line:
``` cssharp 
builder.Services.AddProducer();
```
### Using a producer through dependency Injection
Just inject the IProducer interface, and send message of any type in ProduceAsync() method.
``` cssharp 
public class UseCaseTestImplementation : IUseCaseTestImplementation
{
    private readonly IPublisher _publisher;

    public UseCaseTestImplementation(IPublisher publisher)
    {
        _publisher = publisher;
    }
    public async Task ExecuteAsync(MessageSampleEvent message)
    {
        await _publisher.ProduceAsync(message);
    }
}

public interface IUseCaseTestImplementation
{
    Task ExecuteAsync(MessageSampleEvent message);
}
```

# Consumer
### Configuring application startup
At Startup.cs or Program.cs(in case of .net 6 or later), add the following lines:
``` cssharp 
builder.Services.AddTransient<IUseCaseTestImplementation, UseCaseTestImplementation>();
builder.Services.AddHostedService<MessageSampleEventConsumer>();
```

``` cssharp
public class MessageSampleEventConsumer : Consumer<MessageSampleEvent>
{
    private readonly IUseCaseTestImplementation _useCase;

    public MessageSampleEventConsumer(IUseCaseTestImplementation useCase) : base(consumerGroupId: "myGroupId")
    {
        _useCase = useCase;
    }

    protected override async Task HandleAsync()
    {
        await _useCase.ExecuteAsync(Message);
        return;
    }
}
```
Parameters: 
- consumerGroupId: a subscriber to one or more Kafka topics

It accepts any dependency injection parameter you'll need to use.
