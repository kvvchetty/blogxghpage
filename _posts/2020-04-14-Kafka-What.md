---
layout: post
title: "Kafka What!"
---
#
# Apache Kafka is a distributed streaming platform

#

A synchronous messaging, enable senders and receivers to communicate without the restriction that both must be available listening to the message at the same time. As the following image shows, the sender can produce a message and whenever the receiver is available, it will read and handle it.

![](RackMultipart20200415-4-ggdfa5_html_a895202c15584564.png)

Use Case:

- Messaging
- Logging
- Stream Processing

Kafka has the following features in distributed communication

- Topics

Partitions

Brokers

Producers

Consumers

Consumer Groups

Topic

A topic is where your messages will be sent to (produced). Whoever wants to read this piece of information will read it (consume) from the topic.

![](RackMultipart20200415-4-ggdfa5_html_528e2b9e7ed7369d.png)

Partition

![](RackMultipart20200415-4-ggdfa5_html_b9224a7b74eab09.png)

Broker

![](RackMultipart20200415-4-ggdfa5_html_6822f14b8c3bc287.png)

Producers

  - Writes the messages to topic

Consumer

  - Reads the messages from topic

![](RackMultipart20200415-4-ggdfa5_html_ea2fbc160cd2cf36.png)

KafkaJs is another open source library to run with NodeJs.

Kafka runs locally in a docker container

Open-Bank use case with Kafka

[https://github.com/OpenBankProject/OBP-API/wiki/Open-Bank-Project-Architecture](https://github.com/OpenBankProject/OBP-API/wiki/Open-Bank-Project-Architecture)

# References

[https://kafka.apache.org/intro](https://kafka.apache.org/intro)

**JS Library** : [https://kafka.js.org/docs/getting-started](https://kafka.js.org/docs/getting-started)

