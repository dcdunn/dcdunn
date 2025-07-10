# TIL: Test Automation is an Essential Skill

TL;DR: if you want to become an expert system engineer, simultaneously, you
have to become an expert in testing those systems.

One afternoon, Friday 15th October 2010 to be precise, my life as a software
engineer was transformed by Steve Freeman and Nat Pryce. It was not as you might
expect by reading the now-famous _Growing Object-Oriented Software Guided by
Tests_, which I had not yet come across. Rather it was at a workshop
at the [Agile Cambridge
2010](https://agilecambridge.net/sites/default/files/resources/2010%20programme.jpg)
conference. More on this later.

At that time, I was working at DisplayLink, which was a start-up in
Cambridge.  DisplayLink is a fabless ASIC (application-specific integrated
circuit) company, and is now part of Synaptics Incorporated. DisplayLink build
chips that decode compressed video that is sent over USB, ethernet or wifi from
a host PC or laptop, and displays it on connected monitors. Internally we call
the devices _NIVOs_ (network-in, video-out). The team I was in back then
developed drivers for Windows so that the Windows desktop can be extended on to
the USB connected monitors.

The Principal Software Architect had come from an XP team at Schlumberger, and
even back in 2006 had instituted some early Continuous Integration using
subversion and CruiseControl. He insisted on unit-testing as a good practice,
and we had created a bespoke test-assertion library for C++, in the days before
gtest and gmock.

A huge challenge for us was creating an automated test harness for system or
integration testing. A simple check, such as connecting a device, and making
sure the OS enumerates it, attaches a new display to the desktop and then sends
pixels to our driver was elusive. Not least, this was because we were unable to
find ways to probe the system, or perhaps more accurately, we hadn't understood
that we should have designed the system to admit probing for testability. Even
the tests we could write were unlovable. We tried variations on the theme of
writing scripts or C++ programs that would require some manual fixture setup
step like connecting a device. The test application would then start the
DisplayLink service and then check that the NIVO was enumerated in the Windows
device tree. The latter could be done by probing the device tree via Windows
Management Interface (WMI) functions. Now, the service would take some time to
start. It would then take some time to enumerate the attached device, and
Windows would then take some time to make it usable and detectable. This
inherent asynchrony is what killed the testing. The tests were very flaky, so we
added five seconds of wait time. They were still flaky, so we added another
five. And so on. In the end, even with very large wait times they were
flickering tests. No one believes in slow, flaky tests and we couldn't run them
in our CI pipeline anyway.

Related to this were the kind and number of bugs we had. We had to inter-operate
with all of the graphics card drivers in the world, with any monitors connected
and we had to work with multiple versions of the OS. We had a very large number
of difficult-to-reproduce bugs, and each bug report listed a particular graphics
driver, a particular version of Windows graphics libraries, particular monitors
and a particular variant of NIVO. The difficulty of reproduction alongside the
complex matrix of possible inter-operating components led to some belief that the
interoperability itself was intractable. 

So, back to Agile Cambridge 2010. It was a good event. James Whittaker gave a
Keynote speech on testing at Google. Gojko Adzic presented some case studies
that became _Specification by Example_ and Steve & Nat did a workshop "Testing
at the System Scale". My colleague and I attended it. The
session started with a representative system, which was something like a
warehouse, shop fronts, a backend database. "This must be familiar to most
people here", said Steve. Well, not to us, so the guys said "this might not be
useful to you". Far be it from me to contradict our vaunted hosts, but they
couldn't have been more wrong. 

You can see [Test-Driven Development of Asynchronous
Systems](http://www.natpryce.com/articles/000755.html), on Nat's old blog, which
gives the flavour. The GOOS book covers the subject of course, particularly
Chapter 27: _Testing Aysnchronous Code_. In that workshop we learned about
_probes_ and _pollers_, and how to use them to synchronise your test with the
system-under-test. We learned about Flickering Tests, Slows Tests, False
Positives - all the things that made our existing tests less valuable.

I don't need to rehearse the GOOS approach here, but the crucial tools are the
asynchronous 'matchers', coupled with a philosophy of good testing. Like any
asynchronous process, there are two approaches to synchronisation: polling or
listening. The first leads to the concept of *assert_eventually* matchers, which
repeatedly probe some aspect of the system until it reaches a required state.
The second leads to capturing some system events or messages, in a _notification
trace_, until a required one arrives. In each case, reaching a pre-determined
timeout is considered a test failure.

The precepts that underpin good test practice are:
 * Shared, thread-safe state: modified by the system, probed by the test.
 * Look for success not failure: when the system reaches the required state, the test passes.
 * Succeed Fast.
 * Timeout is Failure.
 * Single Timeout definition.

The first one is really a deceptively simple statement that test-driven
development is a design methodology. It says that we should design systems that
are probe-able, and which deterministically enter states in response to stimuli.
This leads to architectures such as [Ports &
Adapters](https://alistair.cockburn.us/hexagonal-architecture). It also leads to
the realisation that an _observable_ system is synonymous with a _testable_
system, and in the end, an observable system is one that can, in principle, be
restored to a good state when it goes wrong. As Freeman & Pryce put it in the
workshop, to support/test a system we need to
 * know what the system is doing;
 * know when it has stopped doing it;
 * know when it has gone wrong;
 * determine why it has gone wrong;
 * restore it to a good state.

To really make use of this took a creative breakthrough by another colleague. He
invented a way of making a simulated NIVO, using a USB library and then building
a COM object around it. Ta-da. We could make Windows believe that a NIVO had
connected to the USB bus, and we could test the host software driver by both
sending messages from the simulated device and by capturing messages sent from
the host to the device[^2].

So, to create a test harness, we needed something that could talk COM, could
easily invoke Win32 API calls and had a good testing framework, including a port
of Hamcrest. Python is not inevitable, but is a good fit. We were able to write
very simple tests, similar to:

```python
from simulated_device import NivoBuilder
from os_probes import connected_usb_devices
from device_matchers import contains_device, vid, pid
#etc

def test_should_enumerate_connected_nivo():
    nivo = NivoBuilder(). \
        with_vid(DL_VID). \
        with_pid(PID). \
        build()
    
    nivo.connect()
    assert_eventually(connected_usb_devices(contains_device(all_of(vid(DL_VID), pid(PID)))))
```

We discovered that Hudson, soon-to-be Jenkins, was a good CI tool for builds
triggered by commits and reporting out xUnit style test results. Our process was
transformed.  We could use the NIVO simulator to pretend certain monitors were
connected to a certain output, and see if the monitor EDID is reflected in the
resolution list presented to a user. We could use the Win32 API to set a monitor
resolution and detect if the driver sends the correct commands to the device.
And so on. 

The tests were _still_ flickering. But now we knew why - the system itself was
flickering. In other words we had reliable feedback on system bugs. We had a lot
of concurrency issues in our C++ code, and it was a lot easier to reproduce
them, not least because we could run the tests fast and often. As with most
concurrency bugs, a reliable reproduction makes them eminently tractable. We
fixed many issues and improved the architecture to support concurrency better.
We had to grasp this nettle before we had more than just a handful of these
tests.  What do you know? The number of the intractable, rare, hard-to-reproduce
system issues dramatically declined. System stability improved. DisplayLink
developed a reputation for "just working" and being a high quality product. I
believe the lessons about testing asynchronous systems led to us better
understanding asynchronous systems, and were a real driver for system level
improvements.

There's that thing about everything looking like a nail to a person with a
hammer.  Nevertheless this approach has been useful in every project I've worked
on since. Once we had developed the capability of writing such tests, we
were able to add automated upgrade and downgrade tests for our driver. We were
able to develop frame grabber and USB switch drivers so we could bring the actual
NIVOs into the system-under-test (but see below). 

Following DisplayLink, I moved to an IoT platform developer, Electric Imp.
There, we wanted to use the same approach of testing an asynchronous system. How
could we send application code to the device, as if from our cloud servers, and
then sense that it was operating the device peripherals as intended? One day a
colleague's friend, who worked at ARM, came by to say hello. He brought an [MBED
board](https://os.mbed.com/platforms/MAX32670EVKIT/) with him. This was another
serendipitous moment. The MBED has several peripherals on it, and it is
controllable from a host machine via a well defined remote procedure call
mechanism. It was very easy to develop python drivers for the MBED peripherals,
and now we could use the MBED as a sensor for our devices. For example, to test
the i2c slave API, we would set the MBED up as an i2c master. Initially we had a
mess of bread boards and wires, but later we build printed circuit boards as
proper test harnesses[^1].

More recently, back at DisplayLink, we created an IoT backend consisting of an
API gateway, a few internal services, MongoDB persistent storage, and an MQTT
broker for communicating with DisplayLink docks.  The same approach with Python
and asynchronous testing tools works just as well here. Spin up the backend
locally and use a software MQTT client that the test harness can use to
stimulate and probe the MQTT traffic, and use python requests to stimulate and
probe via the REST API. 

What about _cost of ownership_? It is very tempting to take the ability to test
asynchronous systems, and develop lots of tests that integrate all, or nearly
all, of the actual production system into tests.  Most readers will have heard
about the software testing pyramid. I think it contains a fundamental truth: the
more of your system you integrate into a test, the more expensive the tests are
to develop and maintain. We write lots of unit tests, and we become as certain
as we can that each class and method is behaving as required. We can put tests
around libraries and modules to make sure a collection of classes integrates
well with each other and the outside world.  Testing like this encourages your
library or module to have higher level of abstraction. The temptation is to try
to cover every code path in the library tests. This is not necessary, and in
fact can make the code difficult to change or otherwise maintain. So we select
some representative tests - acceptance tests for the library - that give us
confidence that the library is operating as an entity as required.

When we get to the level of subsystem or end-to-end testing, the subject of this
article, we arrive at the most expensive to own tests. The most expensive form
of end-to-end testing is manual testing by test engineers, who did not develop
the code. Prior to release, they will test "everything", checking for
regressions and making sure new features work. This approach of leaving testing
until late, and then doing it as a gate to release can make the feedback time
between code being written and that code being deployed can be months. The
amount of new code is large and the chance of the system being broken in some
way is high. It is well-known in our business (or should be) that it is much
more expensive and risky to fix bugs late in the release cycle.

Even automating these tests is tests is expensive:
 * It is much more difficult to properly control the system under test. For example,
   if you are working on an IoT backend, it is very difficult to arrange for end-to-end
   tests to check the behaviour when a device sends poor data. 
 * They are more fragile. Tests should be deterministic: you should arrange for the system
   to be in a particular state; you should provide an stimulus; you should check
   that the expected change occurred. If you have to rely on a component to
   behave in a certain way in order to test the component you are developing,
   then tests will fail more often because of changes beyond your control.
 * The tests are complex because you have to arrange for a lot of things in a lot of
   components to be in place. If you need to arrange for a device to send a
   certain message to your backend, that is more complex than just sending the
   required message to the backend. This adds a cost of ownership, due to it
   being more expensive to add new tests or to understand failures

However it _is_ desirable to test your system in production-like scenarios. The
IoT cloud backend can be deployed into a production like environment and have
tests run on it without needing a real device. The device firmware can be
deployed on a real device and tests run on it without the need for the real
cloud backend.

Sometimes, even experienced software professionals often hit a category error
here: the belief that the only valuable tests are those test the whole system.
For example, see [The Test
Pyramid](https://www.leapwork.com/blog/testing-pyramid#), which would turn it
into a test hourglass. Consider a DisplayLink example. When a collection of
monitors is connected to a NIVO, the monitors may support video timings that the
NIVO cannot. For example the DL6900 series can raster 4K video resolutions at
30Hz, where many monitors support 60Hz or 120Hz refresh rates. We can also infer
that if a monitor can handle video timing X, then it can also handle video
timing Y, even if it does not report that capability.  These _video timing
rules_ take a set of monitor data
([EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data)) as
input, and output the same or a different set of monitor data. These are
business rules that are expressed very locally in C++ code. That code can be
tested to Kingdom Come with literally thousands of EDIDs, in tests that run so
fast they can be part of the commit stage of the CD pipeline. Reproducing the
test coverage at system level would require thousands of device and monitor
connects, disconnects and reconnects, even simulated ones. Each of these
operations takes seconds to complete by informing the OS of the new
capabilities. Now you have a slow suite of tests that really adds little value
to the focussed tests of the business rules.

The art is identifying boundaries of the system (or subsystem) that you develop
and control. Then find out how to probe at these boundaries, which may require
some careful thinking and creative breakthroughs.  Finally, discover a good set
of tests that do not simply re-test what was covered in lower level testing:
i.e. test at the right level of abstraction - some kind of acceptance tests for
your part of the system. Test the code that runs on the device on a host PC,
using a hardware abstraction layer. Run a local MQTT broker on the test PC. You
need very, very few genuine end-to-end tests, so introduce each new one with
care. 

If you got this far, here's [a link](https://github.com/dcdunn/asyncmatch#) to a
port of the asynchronous test tools of Freeman & Pryce, written in Python.

[^1]: [How Electric Imp Tests Software Part I: Physical Test
Harness](docs/HowElectricImpTestsSoftware.pdf) was published on the Electric Imp
website at the time. 

[^2]: If I knew then what I know now, I would have thought it was a good idea to
not even rely on the operating system. We had developed a decoupled architecture
to support various OSes, such as Windows, Linux and MacOS, so we could have
arranged to create a test suite that used a test double instead of a real OS.
