# EspSoftwareSerial

Implementation of the Arduino software serial library for the ESP8266

This fork implements interrupt service routine best practice.
In the receive interrupt, instead of blocking for whole bytes
at a time - voiding any near-realtime behavior of the CPU - phase
detection and byte assembly is done in the main code.

Except at high bitrates, depending on other ongoing activity,
interrupt handling in particular, this software serial adapter
supports full duplex receive and send.

Same functionality is given as the corresponding AVR library but
several instances can be active at the same time. Speed up to 115200 baud
is supported. The constructor also has an optional input buffer size
arguments.

Please note that due to the fact that the ESP always have other activities
ongoing, there will be some inexactness in interrupt timings. This may
lead to inevitable, but few, bit errors when having heavy data traffic
at high baud rates.

Resource optimization

The memory footprint can be optimized to just fit the amount of expected
incoming asynchronous data.
For this, the SoftwareSerial constructor provides two arguments. First, the
octet buffer size for assembled received octets can be set. Read calls are
satisfied from this buffer, freeing it in return.
Second, the signal edge detection buffer of 32bit fields can be resized.
One octet may require up to to 10 fields, but fewer may be needed,
depending on the bit pattern. Any read or write calls check this buffer
to assemble received octets, thus promoting completed octets to the octet
buffer, freeing fields in the edge detection buffer.

Look at the swsertest.ino example. There, on reset, ASCII characters ' ' to 'z'
are send. This happens not as a block write, but in single write calls per
character. As the example uses a local loopback wire, every outgoing bit is
immediately received back. Therefore, any single write call causes up to
10 fields - depending on the exact bit pattern - to be occupied in the signal
edge detection buffer. In turn, as explained before, each single write call
also causes received bit assembly to be performed, promoting these bits from
the signal edge detection buffer to the octet buffer as soon as possible.
Explaining by way of contrast, if during a a single write call, perhaps because
of using block writing, more than a single octet is received, there will be a
need for more than 10 fields in the signal edge detection buffer.
The necessary size of the octet buffer only depends on the amount of incoming
data until the next read call.

For the swsertest.ino example, this results in the following optimized
constructor arguments to spend only the minimum RAM on buffers required:

The octet buffer size (bufSize) is 93 (91 characters net plus two tolerance).
The signal edge detection buffer (isrBufSize) is 10, as each octet has 10
bits on the wire, which are immediately received during the write, and each
write call causes the signal edge detection to promote the previously sent and
received bits to the octet buffer.

In a more generalized scenario, calculate the bits (use message size in octets
times 10) that may be asynchronously received to determine the value for
isrBufSize in the constructor. Also use the number of received octets
that must be buffered for reading as the value of bufSize.
The more frequently your code calls write or read functions, the greater the
chances are that you can reduce the isrBufSize footprint without losing data,
and each time you call read to fetch from the octet buffer, you reduce the
need for space there.
