# Some Notes on IoT Security

Security is an optional feature in, well any system, but certainly not in an IoT ecosystem.  

## Device-side security
IoT devices must have a secure boot process.

Each device must have a _unique_ and _unforgeable_ identity, managed in a secure 

OTP and a boot processor - together a trust zone. Only this processor sees the device secrets.

_Physical Unclonable Function_.
  - Deterministically generate cryptographic keys on boot. Not stored on the device.


## Link security
 - PFS