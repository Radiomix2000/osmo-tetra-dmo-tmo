
OSMO-TETRA-DMO-TMO
==================

This is a very hacky TMO implementation, based on an experimental, unsupported, DMO-appended fork of osmo-tetra library.
It is expected to use [suo](https://github.com/tejeez/suo) as communication medium.

At this state, this is a Tx-only base station proof of concept. Only the downlink is transmitted, and only a few radio models will try to transmit to it (Sepuras mainly). Uplink is being worked on. 

Please first refer to the [HamTETRA](https://github.com/rats-ry/HamTetra) repository, install all dependencies. 
Then clone this repository inside HamTetra:

    cd
    cd HamTetra
    git clone https://github.com/ManoDaSilva/osmo-tetra-dmo-tmo/
    git checkout tmo-dev
    cd osmo-tetra-dmo-tmo/src
    make

You might also encounter a "DSO Missing from command line". In this case, you'll have to run `make` using this command: 

    make LDFLAGS="-Wl,--copy-dt-needed-entries"

To use TMO, run `./hamtetra_main2` using option number 2. Example:

    ./hamtetra_main2 sx 438.275 2
	
On lower-end devices such as older Raspberry Pi, you might experience choppy downlink. This can be mitigated by running the process at higher priority:
	
	sudo chrt -r 70 ./hamtetra_main2 sx 438.275 2
	
As there are no configuration files yet, most base station values are set directly in the code, including: 
- Downlink frequency: 439.275. If you wish to run it on any other frequency, edit pdus.c, line 120.
- MCC: 206. hamtetra_config.h, line 10
- MNC: 1001. hamtetra_config.h, line 11
- Duplex spacing ID: 7. pdus.c, line 123
	

Original osmo-tetra README.md:

TETRA MAC/PHY layer experimentation code
========================================

(C) 2010-2016 by Harald Welte <laforge@gnumonks.org> and contributors

This code aims to implement the sending and receiving part of the
TETRA MAC/PHY layer.

If you read the ETSI EN 300 392-2 (TETRA V+D Air Interface), you will
find this code implementing the parts between the MAC-blocks (called
type-1 bits) and the bits that go to the DQPSK-modulator (type-5 bits).

It is most useful to look at Figure 8.5, 8.6, 9.3 and 19.12 in conjunction
with this program.

You will need
[libosmocore](https://osmocom.org/projects/libosmocore/wiki/Libosmocore)
to build this softwar

Homepage
--------

The official homepage of the project is
https://osmocom.org/projects/tetra/wiki/OsmocomTETRA

GIT Repository
--------------

You can clone from the official osmo-tetra.git repository using

	git clone git://git.osmocom.org/osmo-tetra.git

There is a cgit interface at http://git.osmocom.org/osmo-tetra/

Mailing List
------------

Discussions related to osmo-tetra are happening on the
tetra@lists.osmocom.org mailing list, please see
https://lists.osmocom.org/mailman/listinfo/tetra for subscription
options and the list archive.

Please observe the [Osmocom Mailing List
Rules](https://osmocom.org/projects/cellular-infrastructure/wiki/Mailing_List_Rules)
when posting.

Contributing
------------

Our coding standards are described at
https://osmocom.org/projects/cellular-infrastructure/wiki/Coding_standards

We us a gerrit based patch submission/review process for managing
contributions.  Please see
https://osmocom.org/projects/cellular-infrastructure/wiki/Gerrit for
more details

The current patch queue for osmo-tetra can be seen at
https://gerrit.osmocom.org/#/q/project:osmo-tetra+status:open


Demodulator
===========

src/demod/python/cpsk.py
 * contains a gnuradio based pi4/DQPSK demodulator, courtesy of KA1RBI

src/demod/python/osmosdr-tetra_demod_fft.py
 * call demodulator on any source supported by gr-osmosdr
   (uhd, fcd, hackrf, blaerf, etc.)

src/demod/python/simdemod2.py
 * call demodulator on a 'cfile' containing complex baseband samples

src/demod/python/{uhd,fcdp}-tetra_demod.py
 * use demodulator directly with UHd or FCDP hadware (no gr-osmosdr)

The output of the demodulator is a file containing one float value for each symbol,
containing the phase shift (in units of pi/4) relative to the previous symbol.

You can use the "float_to_bits" program to convert the float values to unpacked
bits, i.e. 1-bit-per-byte


PHY/MAC layer
=============

library code
------------

Specifically, it implements:
lower_mac/crc_simple.[ch]
* CRC16-CCITT (currently defunct/broken as we need it for
  non-octet-aligned bitfields)
lower_mac/tetra_conv_enc.[ch]
* 16-state Rate-Compatible Punctured Convolutional (RCPC) coder
lower_mac/tetra_interleave.[ch]
* Block interleaving (over a single block only)
lower_mac/tetra_rm3014.[ch]
* (30, 14) Reed-Muller code for the ACCH (broadcast block of
  each downlink burst)
lower_mac/tetra_scramb.[ch]
* Scrambling
lower_mac/viterbi*.[ch]
* Convolutional decoder for signalling and voice channels
phy/tetra_burst.[ch]
* Routines to encode continuous normal and sync bursts
phy/tetra_burst_sync.[ch]


Receiver Program
----------------

The main receiver program 'tetra-rx' expects an input file containing a
stream of unpacked bits, i.e. 1-bit-per-byte.


Transmitter Program
-------------------

The main program conv_enc_test.c generates a single continuous downlinc sync
burst (SB), contining:
	* a SYNC-PDU as block 1
	* a ACCESS-ASSIGN PDU as broadcast block
	* a SYSINFO-PDU as block 2

Scrambling is set to 0 (no scrambling) for all elements of the burst.

It does not actually modulate and/or transmit yet.


Quick example
=============

	# assuming you have generated a file samples.cfile at a sample rate of
	# 195.312kHz (100MHz/512 == USRP2 at decimation 512)
	src/demod/python/tetra-demod.py -i /tmp/samples.cfile -o /tmp/out.float -s 195312 -c 0
	src/float_to_bits /tmp/out.float /tmp/out.bits
	src/tetra-rx /tmp/out.bits

