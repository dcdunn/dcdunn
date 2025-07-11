# Some Internet-of-Things projects
I have worked on Internet-of-Things projects for the lion's share of my career
since 2012. Here is a very brief tour of the projects I have worked on. 

## electric imp: Platform-as-a-Service
I worked at [Electric Imp](https://www.electricimp.com/) from 2012 to 2017,
joining it as the third embedded engineer while it was a start-up.  Electric Imp
is now part of [Kore](http://www.electricimp.com), and is an IoT platform. It
provides cloud connectity to product developers, so that they can focus on their
core business. The hardware consists of an ARM Cortex based System-on-Chip
(SoC)[^1] and a wireless chip. We developed [various
boards](https://developer.electricimp.com/hardware/imp/datasheets) that could be
used within customer products. Initially we ran them under eCos, but later
developed a proprietary real-time OS, ImpOs.  On this OS we managed the wireless
connection and ran a virtual machine for the Squirrel programming language. We
provided [an API](https://developer.electricimp.com/api/device) allowing the
Squirrel application to control the SoC's peripherals, including non-volatile
flash memory. We also provided a propriety protocol than enable the Imp to
communicate with an "agent" running in our cloud[^2]. We worked closely with a
backend team, who created services running in AWS, written in Erlang[^3].

I worked mostly, but not exclusively, on the embedded firmware running on the
device. The firmware was written in C++, and perhaps unusually for an embedded
platform we used dynamic memory allocation. The SoCs have many peripherals, and
most customer applications make use of a small number of them, say a few GPIOs
and the SPI. To statically allocate memory to drive all of them is wasteful,
particulary as it eats memory that can be used for the customer application. One
early project I worked on was the implementation of a fast and efficient heap
memory arena for the imp.

Another memory optimisation was to do with error reporting. When providing a
programming API on the device, the developer wants good feedback when code
fails. The error reports had to be transmitted to the developer through the
cloud backend and thence to their IDE. I worked on a project to reduce the size
of the imp's ROM code by standardising and reusing these strings.

The only communication between an imp and the cloud was between itself and its
cloud agent. The agent was able to communicate with the rest of the web.  I
implementation of some TCP/IP based client protocols to run on the imp using the
Squirrel API.
 * An HTTP API, allowing the agent to make synchronous or asynchronous HTTP
   requests and receive HTTP responses. It included the ability to create
   streaming HTTP requests, i.e. where it was not known ahead of time how much
   data would be transferred. 
 * An MQTT client API. This allows the agent to act as an MQTT client, and to
   connect to an MQTT broker. It was based on the [paho
   library](https://github.com/eclipse-paho/paho.mqtt.c), and was the first of
   several times that I have implemented MQTT clients for IoT devices.
 * An FTP client API, which is now, rightly, deprecated.

Some other projects I worked on include:
 * Drivers and Squirrel bindings for USB, UART, fixed-frequency DAC for audio playback, encrypted storage in SPI Flash.
 * Reimplementation of parts of the Squirrel language targeted at an embedded VM.
 * A mechanism for retrieving impOS updates incrementally if network conditions are flaky, using HTTP range requests.
 * Encryption and decryption of the OS (AES-RSA or AES-CBC) for over-the-air updates and securely boot of the device out of external flash memory. 
    • A regular expression API based on regexp2.

IoT Security was a key part of our offering including:
 * Secure boot of the device, from encrypted storage of the OS;
 * Secure over-the-air (OTA) upgrade of the OS for timely fixes of security vulnerabilities.
 * Unique-per-device identities for each device;
 * A strong TLS mechanism based on mutual certificates and perfect-forward secrecy;

## DisplayLink: smart and connected docking stations

I joined DisplayLink in 2017, but not initially engaged in anything IoT.
However, in autumn 2018, one of our large OEM partners had an enterprise
customer that wanted to detect when a DisplayLink dock became connected to a
computer on their corporate network, using the MAC address as the identifier of
the dock. Now, the dock may have a couple of MAC addresses - one in One-Time
Programmable (OTP) memory, and perhaps one in flash memory, added by the
manufacturer on their factory line. To make this work, we had to pass one of
these addresses from the device to the host computer and then allow IT software
systems to discover it. We designed a solution for that particular case and
unlocked sales of a large fleet of docks at the enterprise.

There were some other related features, such as provisioning docks with a fixed
desktop layout. When an IT manager placed a dock, and two monitors, he would
typically have to use the Windows UI to layout the desktop as desired. We
implemented a way for the dock to be provisioned with a preferred layout, and
have it do that automatically when first connected to a Windows laptop. Again
the dock communicated to the host and the host called the Win32 API. There were
also requests to be able to remotely upgrade dock firmware. In short, it seemed
that there was a market for remote management of docks by IT engineers. We had a
new chip in under design, and it was decided that we would make that chip
IoT-capable. In the meantime we would embark on discovering how to make IoT
docking work. That effort fell into three distinct phases, or projects.

## Proof of Concept of IoT docks
The existing docks were not able to operate autonomously - they are USB devices
that power down when not connected to a host. Without a serious redesign, this
presents a challenge for internet connectivity. We designed a "proxy IoT dock":
in essence it was a C++ Windows service that embeds an MQTT client based on the
[paho](https://github.com/eclipse-paho/paho.mqtt.c) library. At the other end we
created an MQTT broker in Azure - actually it was/is an MQTT front-end to a
RabbitMQ message broker, meaning that we could integrate the MQTT messaging into
a messaging system for the cloud services. We made a rudimentary cloud app that
could show which docks were online and gather information such as what monitors
were connected to them and took it to CES 2019 for feedback. The feedback was
good; we learned that there were potential markets in desk booking systems,
facilities management and remote device management.  

## A Full IoT System Using Proxy Client Software

The next stage was to build something more robust. We created a scalable and
secure backend, running a couple of microservices written in Python using Flask.
One deals with account management using an OAuth approach. Another deals with
servicing a dock management API. We used MongoDB for data persistence and NGINX
as a reverse proxy, and create a REST API for dock management. I helped to build
a team with new capabilities for the business including cloud DevOps. The
services are deployed in Docker containers by Kubernetes, and managed by the
infrastructure-as-code tool, Pulumi. Finally we engaged a front-end team to
build a dock management web application. 

During this phase, we also got to build internet-connectivity into some wireless
docking prototypes we were making. We developed REST API endpoints to allow us
to send messages to the device that would appear on the connected display, and
some other basic functions to support desk booking.  These docks ran Linux on
MCUs adjacent to the DisplayLink chip. We ported our MQTT client code from the
proxy dock to these wireless prototypes, and took them to tradeshows for
feedback.

## An IoT Dock
It became clear that there was no future in developing our own front end. We
removed it and exposed the REST API instead. We now had the DL7450 chip, which
has three cores that run Linux. We created an embedded Linux application with an
MQTT client, but also running an instance of python (based on
[Micropython](https://micropython.org/). Customers can securelydeploy
applications on the dock, and have access to the dock peripherals (GPIO, i2c
etc), real time access to connected displays and cloud connectivity via HTTP or
MQTT. We called this the [DL7450 SDK}(
https://displaylink.github.io/dl-7450/library/index.html) 

   
[^1]: The imp devices that I worked on were
[imp001](https://developer.electricimp.com/sites/default/files/attachments/hardware/datasheets/imp001_specification.pdf),
[imp002](https://developer.electricimp.com/sites/default/files/attachments/hardware/datasheets/imp002_specification.pdf),
[imp003](https://developer.electricimp.com/sites/default/files/2017-07/imp003_SPZV1CDJ_20161024.pdf),
[imp004](https://developer.electricimp.com/sites/default/files/2017-12/imp004m_spzz1mdh_20171211.pdf),
[imp005](https://developer.electricimp.com/sites/default/files/2017-07/imp005_SPUZ1GC_20170105.pdf),
consisting of ARM Cortex M3, M4, R2 processors in for example an STM32F4 SoC.

[^2]: This pre-dated the idea of a [Device Twin](https://learn.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-device-twins), used by Azure IoT Hub.

[^3]: This is a natural choice for point-to-point interprocess communication:
Erlang was developed for telephone systems and has a mechanism where if one open
conversation fails it does not bring down all of the others.  It can also
recover from broken connections.