---
layout: tutorials
title: Persistence with Queues
summary: Demonstrates persistent messages for guaranteed delivery.
icon: persistence-with-queues-icon.png
---

This tutorial builds on the basic concepts introduced in the [publish/subscribe tutorial]({{ site.baseurl }}/publish-subscribe){:target="_blank"}, and will show you how to send and receive persistent messages with JMS 2.0 API client using AMQP and Solace Message Router.

![Sample Image Text]({{ site.baseurl }}/images/persistence-with-queues-icon.png)

This tutorial is available in [GitHub]({{ site.repository }}){:target="_blank"} along with the other [Solace Getting Started AMQP Tutorials]({{ site.links-get-started-amqp }}){:target="_top"}.

At the end, this tutorial walks through downloading and running the sample from source.

This tutorial focuses on using a non-Solace JMS API implementation. For using the Solace JMS API see [Solace Getting Started JMS Tutorials]({{ site.links-get-started-jms }}){:target="_blank"}.

## Assumptions

This tutorial assumes the following:

* You are familiar with Solace [core concepts]({{ site.docs-core-concepts }}){:target="_top"}.
* You have access to a running Solace message router with the following configuration:
    * Enabled `default` message VPN
    * Enabled `default` client username
    * Enabled `default` client profile with guaranteed messaging permissions.
    * A durable queue with the name `amqp/tutorial/queue` exists on the `default` message VPN.
         * See [Configuring Queues]({{ site.docs-confugure-queues }}){:target="_blank"} for details on how to configure durable queues on Solace Message Routers with Solace CLI.
         * See [Management Tools]({{ site.docs-management-tools }}){:target="_top"} for other tools for configure durable queues.

One simple way to get access to a Solace message router is to start a Solace VMR load [as outlined here]({{ site.docs-vmr-setup }}){:target="_top"}. By default the Solace VMR will run with the `default` message VPN configured and ready for messaging. Going forward, this tutorial assumes that you are using the Solace VMR. If you are using a different Solace message router configuration, adapt the instructions to match your configuration.

## Goals

The goal of this tutorial is to demonstrate how to use JMS 2.0 API over AMQP using the Solace Message Router. This tutorial will show you:

1.  How to send a persistent message to a durable queue that exists on the Solace message router
2.  How to bind to this queue and receive a persistent message

## Solace message router properties

In order to send or receive messages to a Solace message router, you need to know a few details of how to connect to the Solace message router. Specifically you need to know the following:

<table>
<tbody>
<tr>
<th>Resource</th>
<th>Value</th>
<th>Description</th>
</tr>
<tr>
<td>Host</td>
<td>String of the form <code>DNS name</code> or <code>IP:Port</code></td>
<td>This is the address client’s use when connecting to the Solace Message Router to send and receive messages. For a Solace VMR this there is only a single interface so the IP is the same as the management IP address. For Solace message router appliances this is the host address of the message-backbone. The port number must match the port number for the plain text AMQP service on the router.</td>
</tr>
<tr>
<td>Message VPN</td>
<td>String</td>
<td>The `default` Solace message router Message VPN that this client will connect to.</td>
</tr>
<tr>
<td>Client Username</td>
<td>String</td>
<td>The `default` client username.</td>
</tr>
</tbody>
</table>

## Java Messaging Service (JMS) Introduction

JMS is a standard API for sending and receiving messages. As such, in addition to information provided on the Solace developer portal, you may also look at some external sources for more details about JMS. The following are good places to start

1. [http://java.sun.com/products/jms/docs.html](http://java.sun.com/products/jms/docs.html){:target="_blank"}.
2. [https://en.wikipedia.org/wiki/Java_Message_Service](https://en.wikipedia.org/wiki/Java_Message_Service){:target="_blank"}
3. [https://docs.oracle.com/javaee/7/tutorial/partmessaging.htm#GFIRP3](https://docs.oracle.com/javaee/7/tutorial/partmessaging.htm#GFIRP3){:target="_blank"}

The last (Oracle docs) link points you to the JEE official tutorials which provide a good introduction to JMS.

This tutorial focuses on using [JMS 2.0 (May 21, 2013)]({{ site.links-jms2-specification }}){:target="_blank"}, for [JMS 1.1 (April 12, 2002)]({{ site.links-jms1-specification }}){:target="_blank"} see [Solace Getting Started AMQP JMS 1.1 Tutorials]({{ site.links-get-started-amqp-jms1 }}){:target="_blank"}.

## Obtaining JMS 2.0 API

This tutorial depends on you having the [Apache Qpid JMS client](https://qpid.apache.org/components/jms/index.html) downloaded and installed for your project, and the instructions in this tutorial assume you successfully done it. If your environment differs then adjust the build instructions appropriately.

The easiest way to do it through Maven. See the project's *pom.xml* file for details.

## Connecting to the Solace Message Router

In order to send or receive messages, an application must start a JMS connection.

There is only one required parameter for establishing the JMS connection: the Solace Message Router host name with the AMQP service port number. The value of this parameter is loaded in the examples by the `javax.naming.InitialContext.InitialContext()` from the *jndi.properties* project's file, but of course it could be assigned directly in the application by assigning the corresponding environment variable.

*jndi.properties*
~~~
java.naming.factory.initial = org.apache.qpid.jms.jndi.JmsInitialContextFactory
connectionfactory.solaceConnectionLookup = amqp://192.168.123.45:8555
~~~

Because the message/reply pattern uses the point-to-point messaging model, the specialized `QueueConnectionFactory` and `QueueConnection` are used.

*QueueSender.java/QueueReceiver.java*
~~~java
Context initialContext = new InitialContext();
QueueConnectionFactory factory = (QueueConnectionFactory) initialContext.lookup("solaceConnectionLookup");

try (QueueConnection connection = factory.createQueueConnection()) {
    connection.setExceptionListener(new QueueConnectionExceptionListener());
    connection.start();
...
~~~

The target for  messages will be a JMS Queue, therefore a session of the `javax.jms.QueueSession` type needs to be created. The session will be non-transacted using the acknowledge mode that automatically acknowledges a client's receipt of a message.

*QueueSender.java/QueueReceiver.java*
~~~java
try (QueueSession session = connection.createQueueSession(false, Session.AUTO_ACKNOWLEDGE)) {
...
~~~

At this point the application is connected to the Solace Message Router and ready to send and receive messages.

## Sending a persistent message to a queue

In order to send a message to a queue a JMS queue sender (a specialization of the JMS *Producer*) needs to be created.

![sending-message-to-queue]({{ site.baseurl }}/images/persistence-with-queues-details-2.png)

There is no difference in the actual method calls to the JMS `QueueSender` when sending a JMS `persistent` message as compared to a JMS `non-persistent` message shown in the [publish/subscribe tutorial]({{ site.baseurl }}/publish-subscribe){:target="_blank"}. The difference in the JMS `persistent` message is that the Solace Message Router will acknowledge the message once it is successfully stored on the message router and the `QueueSender.send()` call will not return until it has successfully received this acknowledgement. This means that in JMS, all calls to the `QueueSender.send()` are blocking calls and they wait for message confirmation from the Solace message router before proceeding. This is outlined in the JMS 2.0 specification and Solace JMS adheres to this requirement.

The name of the queue for sending messages is loaded by `javax.naming.InitialContext.InitialContext()` from the *jndi.properties* project's file. It must exist on the Solace Message Router as a `durable queue`.

See [Configuring Queues]({{ site.docs-confugure-queues }}){:target="_blank"} for details on how to configure durable queues on Solace Message Routers with Solace CLI.

See [Management Tools]({{ site.docs-management-tools }}){:target="_top"} for other tools for configure durable queues.

*jndi.properties*
~~~
queue.queueLookup = amqp/tutorial/queue
~~~

*QueueSender.java*
~~~java
Queue target = (Queue) initialContext.lookup("queueLookup");
try (QueueSender messageSender = session.createSender(target)) {
    messageSender.setDeliveryMode(DeliveryMode.PERSISTENT);
...
~~~

Now send the message:

*QueueSender.java*
~~~java
messageSender.send(message);
~~~

## Receiving a persistent message from a queue

In order to receive a persistent message from a queue a JMS queue receiver (a specialization of the JMS *Consumer*) needs to be created.

![]({{ site.baseurl }}/images/persistence-with-queues-details-1.png)

The name of the queue is loaded by the `javax.naming.InitialContext.InitialContext()` from the *jndi.properties* project's file, and it is the same as the one to which we send messages.

*jndi.properties*
~~~
queue.queueLookup = amqp/tutorial/queue
~~~

*QueueReceiver.java*
~~~java
Queue source = (Queue) initialContext.lookup("queueLookup");
try (QueueReceiver messageConsumer = session.createReceiver(source)) {
...
~~~

This is how we receive messages sent to the queue.

*QueueReceiver.java*
~~~java
Message message = messageConsumer.receive();
~~~

## Summarizing

Combining the example source code shown above results in the following source code files:

*   [QueueSender.java]({{ site.repository }}/blob/master/src/main/java/com/solace/samples/QueueSender.java){:target="_blank"}
*   [QueueReceiver.java]({{ site.repository }}/blob/master/src/main/java/com/solace/samples/QueueReceiver.java){:target="_blank"}

### Getting the Source

Clone the GitHub repository containing the Solace samples.

```
git clone {{ site.repository }}
cd {{ site.baseurl | remove: '/'}}
```

### Building

Modify the *jndi.properties* file to reflect your Solace Message Router host and port number for the AMQP service.

You can build and run both example files directly from Eclipse.

To build a jar file that includes all dependencies execute the following:

~~~sh
mvn assembly:single
~~~

Then the examples can be executed as:

~~~sh
java -cp ./target/solace-samples-amqp-jms1-1.0.1-SNAPSHOT-jar-with-dependencies.jar  com.solace.samples.QueueReceiver
java -cp ./target/solace-samples-amqp-jms1-1.0.1-SNAPSHOT-jar-with-dependencies.jar  com.solace.samples.QueueSender
~~~

### Sample Output

First start the `QueueReceiver` so that it is up and waiting for messages.

~~~sh
$ java -cp ./target/solace-samples-amqp-jms1-1.0.1-SNAPSHOT-jar-with-dependencies.jar  com.solace.samples.QueueReceiver
2017-07-04T16:45:19,705 INFO sasl.SaslMechanismFinder - Best match for SASL auth was: SASL-ANONYMOUS
2017-07-04T16:45:19,736 INFO jms.JmsConnection - Connection ID:41cb3895-9bb8-44eb-aa5a-7be68c246ef2:1 connected to remote Broker: amqp://192.168.123.45:8555
2017-07-04T16:45:19,783 INFO samples.QueueReceiver - Waiting for a persistent message...
~~~

Then you can start the `QueueSender` to send the message.

~~~sh
$ java -cp ./target/solace-samples-amqp-jms1-1.0.1-SNAPSHOT-jar-with-dependencies.jar  com.solace.samples.QueueSender
2017-07-04T16:46:04,146 INFO sasl.SaslMechanismFinder - Best match for SASL auth was: SASL-ANONYMOUS
2017-07-04T16:46:04,177 INFO jms.JmsConnection - Connection ID:d3510904-7181-436f-8a0f-c1f23d713dcd:1 connected to remote Broker: amqp://192.168.123.45:8555
2017-07-04T16:46:04,239 INFO samples.QueueSender - Message message sent successfully.
~~~

Notice how the message is received by the `QueueReceiver`.

~~~sh
...
2017-07-04T16:45:19,783 INFO samples.QueueReceiver - Waiting for a persistent message...
2017-07-04T16:46:04,255 INFO samples.QueueReceiver - Received message with string data: "Message with String Data"
~~~

Now you know how to use JMS 2.0 API over AMQP using the Solace Message Router to send and receive persistent messages from a queue.

If you have any issues sending and receiving message or reply, check the [Solace community]({{ site.links-community }}){:target="_top"} for answers to common issues seen.