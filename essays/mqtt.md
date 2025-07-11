# Why MQTT is a Good IoT Protocol

I have become very familiar with MQTT, particulary through implementing MQTT
client software several times. In this essay I'll explain why it is a good
protocol for IoT devices in comparison to HTTP. Please note, I do not intend to
say that this is _always_ the case, and of course there may be reasons why HTTP,
or another application-layer protocol, is suitable. For the projects I have
worked on, at least the following give context:
 * Devices may be remote, under poor network conditions;
 * Power optimisation may be important, such as the device being battery powered,
   or in low-power states for long periods;
 * The management or control of devices is centralised in a cloud application
   and devices are not required to communicate directly with each other;
 * As part of this, the control centre should know when devices are online or
   otherwise.
 * Latency should be low, and in particular control messages sent from the
   cloud should be handled as quickly as possible by the device.

Two MQTT clients I have worked on. First, the [DL7450
SDK](https://displaylink.github.io/dl-7450/library/mqtt.html) provides an MQTT
client API so that a DisplayLink based docking station can communicate with
customer's cloud applications via an MQTT broker. I have also designed an MQTT
broker[^1]. Second, the [MQTT
client](https://developer.electricimp.com/api/mqtt), which provides electric imp
cloud agents with MQTT connectivity.

Developers may be tempted to implement a proprietary device-cloud protocol. This
can be surprisingly painful, particularly managing the connection state, and
ensuring messages are delivered. I suggest to avoid [Not Invented Here
Syndrome](https://en.wikipedia.org/wiki/Not_invented_here), and use a stable,
well-supported protocol.

One option is to use HTTP and a REST API in the cloud. "Why MQTT not REST?" is a
question that I have fielded often when building IoT infrastructure. Given the
context outlined above, there are a few reasons why HTTP wasn't suitable. In an
HTTP session, the client initiates the connection and a socket is established.
The client makes a request and receives a response.
 * In traditional HTTP, the socket is closed after the request-response cycle is
   complete, unless the *keep-alive* header is included. In that case, further
   request/response transfers are possible without closing the socket, but each
   transfer must follow the HTTP request-response pattern. In other words, the
   messaging mode is synchronous: the client waits for a response after sending
   a request.
 * Consier a system with a centralised control application in a cloud server.
   Due to the request/response structure of HTTP, the device would have
   repeatedly to send a request to see if the server has any new messages - HTTP
   _polling_. This makes for poor latency, due to the delay between the message
   being ready for consumption, and the device fetching it. This HTTP polling is
   inefficient, not least if power is an issue on the device. It also burdens
   the server with redundant requests, which is not scalable. 
 * The socket may be kept open, and further request-responses sent. Two-way chat
   is difficult. In HTTP/1 it is not possible. In HTTP/2 it is possible by
   opening two streams, each half-duplex[^2]. Websockets is an extension to HTTP
   that has full-duplex bidirectional communication. 
 * The quality of service is the same as TCP (guaranteed delivery, guaranteed
   ordering of packets, no duplicates). While TCP is considered a reliable
   transport, the delivery guarantee is compromised in many situations, such as
   network outage, timeouts, connection closed (RST) by the server; i.e.
   _in-flight_ transfers can be lost. Moreover, there is no guarantee that the
   packets are passed to the application layer. 
 * HTTP has no built-in mechanism for acknowledging receipt of a message, other
   than the response sent by the server.

The foregoing is not entirely fair. There are application layer mitigations,
such as using long polling, HTTP/2 or WebSockets. Nevertheless, long-lived,
bidirectional, good quality-of-service communication is not straightforward with
HTTP. For long-lived connections and frequent, small data transfers such as IoT
telemetry, MQTT is a better option.

MQTT is an application-layer protocol based on TCP/IP. It decouples producers
and consumers of IoT device data: the producers (publishers) of data do not need
to know about the consumers (subscribers). It does so by being a client-server
protocol. MQTT is a _topic_ based messaging protocol. Clients _subscribe_ to
topics they are interested in. Clients _publish_ messages on a _topic_, and the
server arranges for the subscribers of that topic to recieve the message. For
this reason the server is usually called the _MQTT broker_. 

For example, suppose a device has ID _fred42_, once it is connected to a broker,
it might subscribe to a topic _fred42/control/#_, where the _#_ is a wildcard -
that is, it is interested in all control messages intended for it. On the other
hand it might publish messages on topics such as _fred42/sensor/temperature_. A
remote air-conditioning application, also connected to the broker, can send
control messages back to the device in response to the temperature readings, on
a topic like _fred42/control/fan_. Unlike HTTP, the client and server can both
initiate messaging. This means there is no need for the client to poll the
broker and the control messaging has less latency than HTTP. MQTT clients and
brokers can use the topic system to broadcast to large numbers of subscribers.
For example, a broker can instruct sets of devices to update their firmware.

Once established, the connection is persistent - it remains open until the
client closes it. The broker may close the connection in a variety of
situations. For example, if the connection handshake fails due to authentication
or authorisation failure, the server will close the socket. It can also close
the connection if the client makes protocol violations, such as badly formed
packets.

An MQTT connection is also full-duplex, allowing messages to flow in either
direction simultaneously[^3]. The messaging mode asynchronous, so that when a
message is sent, the client (or broker) does not need to wait for it to be
received before continuing its operation. The acknowledgements are received
asynchronously.

MQTT has strong Quality-of-Service (QoS) semantics. Each message is sent with a
QoS level, and messages are guaranteed to be delivered at most once (QoS 0), at
least once (QoS 1) or exactly once (QoS 2).

 * QoS 1 and 2 guarantee that the application layer has received the message.
 * The client has a _unique_ identity. This is crucial for IoT security. Two
   clients with the same ID cannot connect to the same broker. The ID is also
   used for session management, managing QoS guarantees and for resuming
   sessions.  
 * Heartbeat
 * Last will and testament
 * Unreliable networks.
 
 offline messaging support.
 
One last point. MQTT can be made secure. First, since it is based on TCP/IP, the
socket can be wrapped with Transport Layer Security (TLS). 

  * Websockets
  * Port blocking on enterprise networks

The main IoT platform services, such as Azure IoT Hub, allow devices to connect via MQTT. 

[^1]: The Synaptics Vision cloud uses RabbitMQ for internal messaging between
services. RabbitMQ has an [MQTT plugin](https://www.rabbitmq.com/docs/mqtt),
which is a suitable adapter between the device messaging and the inter-service
messaging.

[^2]: Duplex means data can flow in both directions; half-duplex means that it
can flow in one direction at a time, while full-duplex means it can flow in both
directions simultaneously.

[^3]: The [paho]( https://github.com/eclipse-paho/paho.mqtt.c) MQTT client
library instantiates two threads, one each for sending and receiving.