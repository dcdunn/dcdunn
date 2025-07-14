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
least once (QoS 1) or exactly once (QoS 2). QoS 0 is 'fire-and-forget', and
should be used if it is not very important that the message reaches its intended
audience. Unlike TCP, QoS 1 and 2 guarantee that the application layer receives
the message.

Under certain conditions, this guarantee can be met across network outages, the
broker going offline, or even device power failures. A QoS 0 message is
discarded as soon as it is sent.  However, QoS 1 and 2 messages are stored
locally on the client, pending acknowledgement and may be resent to satisfy the
QoS semantics. Depending on implementations, the persisted messages may be store
in memory or in non-volatile storage. In IoT devices, resource constraints will
dictate the amount of available storage of each type.  Messages sent or
unacknowledged while the device is disconnected can be delivered if the
connection resumes.

As part of the connection protocol, clients can also request a _persistent
session_. In this case, the broker stores information about the session,
including the client's subscriptions, all unacknowledged QoS 1 and 2 messages,
and any QoS 1 and 2 messages sent while the client is disconnected. If the
client requests a persistent session (by setting the *cleanSession* flag to
false in the connect options), the client must also set up the means to save
session information. Finally, it is possible to set a flag for a 'retained
message', in which case the broker will store the most recent message sent to a
client, and will immediately send it when the client connects.

The MQTT protocol also provides a 'heartbeat' mechanism. In the connection
handshake the client negotiates a _keep alive_ time. This is the maximum time
allowed between messages sent by the client. If there is no message to send in
the keep alive interval, then the client sends a _ping request_, and the broker
sends a _ping response_ back. If the broker does not receive a ping request, it
will close the  connection. This heartbeat overcomes a problem with TCP, which
is that neither end of a TCP connection can detect if the other end has gone,
leading to 'half-open' TCP connections. 

A client may also create a "last will and testament" as part of the connection
protocol.  If a broker detects that the client has disconnected, it will send
the will message on the will topic. This helps IoT applications to keep an
accurate view of what is happening in the their ecosystem.

What about security? One of the precepts of IoT security is that communication
between device and cloud is over a secure transport. The first precept of IoT
device security is that each device has a _unique_ and _unforgeable_ identity.
How that is acheived is not the subject of this essay, but MQTT is designed
around this feature. Every MQTT client has a unique identity, and two clients
with the same ID cannot connect to the same broker. The client ID is used for
session management, managing QoS guarantees and for resuming sessions.  

MQTT supports Transport Layer Security (TLS) handshakes and encrypted
connections. During the handshake, the broker can ask the client for an X.509
certificate, that is trusted by the broker. The broker can refuse to connect
with a client whose identity cannot be authenticated in this way. In addition to
X.509 authentication, MQTT provides the ability for a client to specify a
username and password in the connect options.

It is a good idea to insist on secure cipher suites for MQTT connections, and to
use [_Perfect Forward
Secrecy_](https://csrc.nist.gov/glossary/term/perfect_forward_secrecy) for
connections. Under such schemes, ephemeral session keys are generated using
algorithms such as [Elliptic Curve Diffie-Helman](). This means that if a
session secret is lost, previous and future sessions are not compromised.

I have presented some reasons why MQTT is a good protocol for IoT device-cloud
communication:
 * Secure connections;
 * Long lived connections;
 * Full duplex messaging;
 * Asynchronous messaging;
 * Quality-of-Service semantics;
 * Good connection management;
 * Last-Will and Testament.


[^1]: The Synaptics Vision cloud uses RabbitMQ for internal messaging between
services. RabbitMQ has an [MQTT plugin](https://www.rabbitmq.com/docs/mqtt),
which is a suitable adapter between the device messaging and the inter-service
messaging.

[^2]: Duplex means data can flow in both directions; half-duplex means that it
can flow in one direction at a time, while full-duplex means it can flow in both
directions simultaneously.

[^3]: The [paho]( https://github.com/eclipse-paho/paho.mqtt.c) MQTT client
library instantiates two threads, one each for sending and receiving.