# About me

I am a Principal Software Engineer with twenty-five years experience. My
[LinkedIn profile](https://www.linkedin.com/in/david-dunn-60b4385/) contains
job-related information. This page is more about the technologies I've
developed and some of the things I've learned along the way.

Language-wise I am most experienced in
 * Modern C++, for embedded Linux, embedded RTOSes and Windows drivers and
   applications.
 * Python 3, for cloud backend services (using Flask) and for test automation over
   most of my career.

Here are some not-very-well organised, or concise, words that explain some of
the projects I have worked on over my career.

## Test Automation
One thing I have learned is that a prerequisite for designing and delivering
high-quality software systems - of any type - is a test-automation capability,
and a good understanding of the hows and whys of such capabilities. I have
written [some musings](test_automation.md) on one aspect of the subject:
testing in the face of asynchrony. For the last fifteen years I have worked in,
and helped grow, teams that favour a CI approach to development.  Test
automation is an important part of this, but so is understanding how to build
durable architectures: good modularised systems, with well-defined interfaces,
tight responsibilities, sensible messaging protocols. I have come to believe
that developing testable systems is the best way to develop good software
designs, from very local classes, through libraries, packages and so on, and
that these principles apply to a collection of cloud services just as well as
they do to a collection of C++ or Python classes. 

## Internet-of-Things
I worked at [Electric Imp](https://www.electricimp.com/) from 2012 to 2017,
joining it as a start-up, as the third engineer in the embedded firmware team.
The idea was that we would provide cloud connectivity to product developers so
they could focus on their core business and use our hardware and cloud services
to make their device internet-connected. The hardware consisted of an ARM
Cortex System-on-Chip (SoC)[^1] and a wireless chip. We developed [various
boards](https://developer.electricimp.com/hardware/imp/datasheets). Initially
we ran them under eCos, but later developed a proprietary real-time OS, ImpOs.
On this OS we managed the wireless connection and ran a virtual machine for the
Squirrel programming language. We provided [an
API](https://developer.electricimp.com/api/device) that allowed the Squirrel
application to control the SoC's peripherals, to have access to flash memory.
We also provided a propriety protocol than enable the Imp to communicate with
an "agent" running in our cloud[^2].

We worked closely with a backend team, who created services running in AWS,
written in Erlang. This is a natural choice for point-to-point interprocess
communication: Erlang was developed for telephone systems and has a mechanism
where if one open conversation fails it does not bring down all of the others.
It can also recover from broken connections.

IoT Security was a key part of our offering including
 * Secure boot of the device, including encrypted storage of the OS;
 * Safe upgrade of the OS to fix security vulnerabilities;
 * Unique-per-device identities for each device;
 * A strong TLS mechanism based on mutual certificates and perfect-forward secrecy;

I (re)joined DisplayLink in 2017, not initially having anything to do with IoT.
However, in around autumn 2018, one of our large OEM partners had an enterprise
customer that wanted to detect when a DisplayLink docking station came online
by its MAC address. To make this work, we had to pass that address from the
device to the host computer and then allow IT software systems to discover it.
There were various other desires, such as firmware upgrades of docks and other
remote dock management features. We had a chip in development, and it was
decided that we would make that chip IoT-capable. In the meantime we would
embark on discovering how to make IoT docking work. That effort fell into three phases.

1. The existing docks were not able to operate autonomously - they are USB
   devices that power down when not connected to a host. We created a proxy
   that ran under Windows. In essence it is a C++ Windows service that embeds
   an MQTT client based on the
   [paho](https://github.com/eclipse-paho/paho.mqtt.c) library. At the other
   end we created an MQTT broker in Azure - actually it was/is an MQTT
   front-end to a RabbitMQ message broker, meaning that we could integrate the
   MQTT messaging into a messaging system for the cloud services. We made a
   rudimentary cloud app that could show which docks were online and gather
   information such as what monitors were connected to them and took it to CES
   2019 for feedback. The feedback was good; we learned that there were
   potential markets in desk booking systems, facilities management and remote
   device management.  
2. The next stage was to build something more robust. We created a scalable and
   secure backend, running a couple of microservices written in Python using
   Flask. One deals with account management using an OAuth approach. Another
   deals with servicing a dock management API. We used MongoDB for data
   persistence and NGINX as a reverse proxy, and create a REST API for dock
   management. I helped to build a team with new capabilities for the business
   including cloud DevOps. The services are deployed in Docker containers by
   Kubernetes, and managed by the infrastructure-as-code tool, Pulumi. Finally
   we engaged a front-end team to build a dock management web application. 

   During this phase, we also got to build internet-connectivity into some
   wireless docking prototypes we were making. We developed REST API endpoints
   to allow us to send messages to the device that would appear on the
   connected display, and some other basic functions to support desk booking.
   These docks ran Linux on MCUs adjacent to the DisplayLink chip. We ported
   our MQTT client code from the proxy dock to these wireless prototypes, and
   took them to tradeshows for feedback.
3. It became clear that there was no future in developing our own front end. We
   removed it and exposed the REST API instead. We now had the DL7450 chip,
   which has three cores that run Linux. We created an embedded Linux
   application with an MQTT client, but also running an instance of python
   (based on [Micropython](https://micropython.org/). Customers can
   securelydeploy applications on the dock, and have access to the dock
   peripherals (GPIO, i2c etc), real time access to connected displays and
   cloud connectivity via HTTP or MQTT. We called this the [DL7450 SDK}(
   https://displaylink.github.io/dl-7450/library/index.html) 


## Academic Life
In a different life I was an academic researcher in various areas of Fluid
Mechanics. You can find my research papers and academic contributions on [Google
Scholar](https://scholar.google.com/citations?user=aaN_h4EAAAAJ&hl=en).

Most of my academic computing was done in Fortran 90, although in my job at
Bristol I learned and used both C++ and Python. The techniques included:
 * [Contour Dynamics and Contour Surgery](https://www.sciencedirect.com/science/article/abs/pii/016779778990004X)
   but adapted for quasigeostrophic flows. I used this technique for studying
   the nonlinear interaction of vortices and waves in the ocean or atmosphere.
 * Wavelet analysis of turbulent boundary layer flows. I used the pyramid algorithm
   of Mallat for efficient computation of an orthogonal wavelet transform of the
   data. Unlike the Fourier transform, a wavelet transform maintains some
   spatial information. Hence they are suitable for examining coherent spatial
   structure at different spectral scales. 
 * I used the [Gerris](http://gerris.dalembert.upmc.fr/) solver, which includes an
   accurate model of the free surface.  I spent some time with the author,
   Stephane Popinet, when he visited Bristol University. He was very forward
   thinking for an academic programmer, and I learned about version control
   systems and object-orientation (in C, but there you go) from him. 
   
[^1]: The imp devices that I worked on were
[imp001](https://developer.electricimp.com/sites/default/files/attachments/hardware/datasheets/imp001_specification.pdf),
[imp002](https://developer.electricimp.com/sites/default/files/attachments/hardware/datasheets/imp002_specification.pdf),
[imp003](https://developer.electricimp.com/sites/default/files/2017-07/imp003_SPZV1CDJ_20161024.pdf),
[imp004](https://developer.electricimp.com/sites/default/files/2017-12/imp004m_spzz1mdh_20171211.pdf),
[imp005](https://developer.electricimp.com/sites/default/files/2017-07/imp005_SPUZ1GC_20170105.pdf),
consisting of ARM Cortex M3, M4, R2 processors in for example an STM32F4 SoC.

[^2]: This pre-dated the idea of a [Device Twin](https://learn.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-device-twins), used by Azure IoT Hub.

<!--
**dcdunn/dcdunn** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m currently working on ...
- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m looking for help with ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...
-->
