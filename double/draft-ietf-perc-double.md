%%%

    #
    # SRTP Double Encryption Procedures
    #
    # Generation tool chain:
    #   mmark (https://github.com/miekg/mmark)
    #   xml2rfc (http://xml2rfc.ietf.org/)
    #

    Title = "SRTP Double Encryption Procedures"
    abbrev = "Double SRTP"
    category = "std"
    docName = "draft-ietf-perc-double-05"
    ipr= "trust200902"
    area = "Internet"
    keyword = ["PERC", "SRTP", "RTP", "conferencing", "encryption"]

    [pi]
    symrefs = "yes"
    sortrefs = "yes"
    compact = "yes"

    [[author]]
    initials = "C."
    surname = "Jennings"
    fullname = "Cullen Jennings"
    organization = "Cisco Systems"
      [author.address]
      email = "fluffy@iii.ca"

    [[author]]
    initials = "P."
    surname = "Jones"
    fullname = "Paul E. Jones"
    organization = "Cisco Systems"
      [author.address]
      email = "paulej@packetizer.com"

    [[author]]
    initials = "A.B."
    surname = "Roach"
    fullname = "Adam Roach"
    organization = "Mozilla"
      [author.address]
      email = "adam@nostrum.com"

%%%

.# Abstract

In some conferencing scenarios, it is desirable for an intermediary to
be able to manipulate some RTP parameters, while still providing
strong end-to-end security guarantees.  This document defines SRTP
procedures that use two separate but related cryptographic operations to
provide hop-by-hop and end-to-end security guarantees.  Both the
end-to-end and hop-by-hop cryptographic algorithms can utilize an
authenticated encryption with associated data scheme or take advantage
of future SRTP transforms with different properties.


{mainmatter}

# Introduction

Cloud conferencing systems that are based on switched conferencing
have a central Media Distributor device that receives media from
endpoints and distributes it to other endpoints, but does not need to
interpret or change the media content.  For these systems, it is
desirable to have one cryptographic key from the sending endpoint
to the receiving endpoint that can encrypt and authenticate the media
end-to-end while still allowing certain RTP header information to be
changed by the Media Distributor.  At the same time, a separate
cryptographic key provides integrity and optional confidentiality
for the media flowing between the Media Distributor and the endpoints.
See the framework document that describes this concept in more detail
in more detail in [@I-D.ietf-perc-private-media-framework].

This specification defines an SRTP transform that uses the AES-GCM
algorithm [@!RFC7714] to provide encryption and integrity for an RTP
packet for the end-to-end cryptographic key as well as a hop-by-hop
cryptographic encryption and integrity between the endpoint and the
Media Distributor.  The Media Distributor decrypts and checks
integrity of the hop-by-hop security.  The Media Distributor MAY
change some of the RTP header information that would impact the
end-to-end integrity.  The original value of any RTP header field that
is changed is included in a new RTP header extension called the
Original Header Block.  The new RTP packet is encrypted with the
hop-by-hop cryptographic algorithm before it is sent.  The receiving
endpoint decrypts and checks integrity using the hop-by-hop
cryptographic algorithm and then replaces any parameters the Media
Distributor changed using the information in the Original Header Block
before decrypting and checking the end-to-end integrity.

One can think of the double as a normal SRTP transform for encrypting
the RTP in a way where things that only know half of the key, can
decrypt and modify part of the RTP packet but not other parts of if
including the media payload.

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [@!RFC2119].

Terms used throughout this document include:

* Media Distributor: media distribution device that routes media from
  one endpoint to other endpoints

* end-to-end: meaning the link from one endpoint through one or
  more Media Distributors to the endpoint at the other end.

* hop-by-hop: meaning the link from the endpoint to or from the
  Media Distributor.

* OHB: Original Header Block is an RTP header extension that contains
  the original values from the RTP header that might have been changed
  by a Media Distributor.


# Cryptographic Context

This specification uses a cryptographic context with two parts: an inner
(end-to-end) part that is used by endpoints that originate and
consume media to ensure the integrity of media end-to-end, and an
outer (hop-by-hop) part that is used between endpoints and Media
Distributors to ensure the integrity of media over a single hop and to
enable a Media Distributor to modify certain RTP header fields.  RTCP
is also handled using the hop-by-hop cryptographic part.  The
RECOMMENDED cipher for the hop-by-hop and end-to-end algorithm is
AES-GCM.  Other combinations of SRTP ciphers that support the
procedures in this document can be added to the IANA registry.

The keys and salt for these algorithms are generated with the following
steps:

* Generate key and salt values of the length required for the combined
  inner (end-to-end) and outer (hop-by-hop) algorithms.

* Assign the key and salt values generated for the inner (end-to-end)
  algorithm to the first half of the key and salt for the double
  algorithm. 

* Assign the key and salt values for the outer (hop-by-hop) algorithm
  to the second half of the key and salt for the double algorithm. The
  first half of the key is revered to as the inner key while the
  second out half is referred to as the outer key. When a key is used
  by a cryptographic algorithm, the salt used is the part of the salt
  generated with that key.
  
Obviously, if the Media Distributor is to be able to modify header
fields but not decrypt the payload, then it must have cryptographic
key for the outer algorithm, but not the inner (end-to-end) algorithm.  This
document does not define how the Media Distributor should be
provisioned with this information.  One possible way to provide keying
material for the outer (hop-by-hop) algorithm is to use
[@I-D.ietf-perc-dtls-tunnel].

# Requirements for RTP packets

The input to the double transform is an RTP packet.  This packet must contain
certain extensions in order to encode any differences between the inner and
outer transforms.

## End-to-End Extensions Length

It is possible for the sender of an RTP packet to apply end-to-end protections
to the first part of a block of extensions.  It does this by making the first
extension in the packet an End-to-End Extensions Length (E2EEEL) extension,
which specifies the length of the data in the extensions block that should
receive end-to-end protections.  The end-to-end data MUST comprise a set of
complete extensions; extensions MUST NOT be partially protected.

The extension is two octets long, with the following form:

{align="left"}
~~~~~
 0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3
+-+-+-+-+-+-+-+-+-------------------------------+
|  ID   | len=1 |     E2E Extensions Length     |
+-+-+-+-+-+-+-+-+-------------------------------+
~~~~~

If included, the E2EEEL extension MUST be the first extension in the extensions
block.  If there are no end-to-end protected extensions (i.e., the length is
zero), then the E2EEEL extension MUST NOT be included.


## Original Header Block

The OHB contains the original values of any modified header fields.

Any SRTP packet protected with this profile that has any extensions at all MUST
have an OHB extension, even if no headers were modified.  In packet with no
extensions (X=0), which clearly cannot have an OHB, all header fields MUST be
unmodified from the values set by the sender.

The Media Distributor is only permitted to modify the extension (X)
bit, payload type (PT) field, and the RTP sequence number field.

The OHB extension is either one octet in length or three octets in length.  The
length of the OHB indicates what data is contained in the extension.

If the OHB is one octet in length, it contains the original PT field
value.  In this case, the OHB has this form:

{align="left"}
~~~~~
 0                   1
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
+-+-+-+-+-+-+-+-+---------------+
|  ID   | len=0 |R|     PT      |
+-+-+-+-+-+-+-+-+---------------+
~~~~~

If the OHB is three octets in length, it contains the original PT
field value and RTP packet sequence number.  In this case, the OHB has
this form:

{align="left"}
~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 6 4 5 6 7 8 9 1
+-+-+-+-+-+-+-+-+---------------+-------------------------------+
|  ID   | len=2 |R|     PT      |        Sequence Number        |
+-+-+-+-+-+-+-+-+---------------+-------------------------------+
~~~~~

Note that "R" indicates a reserved bit that MUST be set to zero when
sending a packet and verified to be zero upon receipt. ID is the RTP Header
Extension identifier negotiated in the SDP. 


## Placement of Extensions

If an RTP packet would not otherwise have extensions, then it MUST NOT have an
E2EEL extension.  If an RTP packet has extensions, then the sender specifies
whether they receive hop-by-hop or end-to-end integrity protection.

In a packet with only hop-by-hop extensions, the first extension in the
packet MUST be an OHB extension.  The OHB MUST be present even if no
modifications have been made; in this case, the header field values in the OHB
will match those in the RTP header itself.

If any extension are to receive end-to-end integrity protection, then the first
extension in the packet MUST be an E2EEL extension specifying the length of
these headers.  The end-to-end extensions MUST immediately follow the E2EEL
extension, and the OHB extension MUST be the first extension after the
end-to-end extensions.  Hop-by-hop extensions may then follow the OHB in any
order.

The placement of the E2EEL and OHB is important because it allows an SRTP
library to recognize these extensions without needing to know the negotiated
extension type values for these extensions.  The transform at the  receiving
endpoint will authenticate the original packet by restoring the modified RTP
header field values and header extensions.  It does this by copying the
original values from the OHB and then removing the OHB extension and any other
RTP header extensions that appear after the OHB extension.

Note that the difference in lengths between the OHB and the E2EEL allows a
receiver to easily tell whether a packet contains end-to-end protected
extensions, regardless of the negotiated extension IDs.  If the first extension
is two octets long, then it is an E2EEL.  If it is one octet or three octets
long, it is an OHB.

If a Media Distributor modifies an original RTP header value, the
Media Distributor MUST include the OHB extension to reflect the
changed value, setting the X bit in the RTP header to 1 if no header
extensions were originally present.  If another Media Distributor
along the media path makes additional changes to the RTP header and
any original value is already present in the OHB, the Media
Distributor must extend the OHB by adding the changed value to the
OHB.  To properly preserve original RTP header values, a Media
Distributor MUST NOT change a value already present in the OHB
extension.

# RTP Operations

## Additional Authenticated Data

In the base GCM transform, the Additional Authenticated Data (AAD) supplied to
the GCM algorithm comprises the RTP header and all extensions present.  In the
double transform, the AAD for the outer transform is the same as for GCM, while
the AAD for the inner transform reflects header for the original packet (before
any modifications) and any end-to-end protected extensions.  To reconstruct the
inner AAD from a packet:

* If there are no extensions then the AAD is the RTP header provided.

* If the first extension is two octets long, then interpret it as an E2EEL
  extension.

  * Extend the AAD with the 4-octet extension header, the E2EEL extension, and
    the number of bytes indicated in the E2EEL extension.

  * Reset the length field in the 4-octet extension header in the AAD to the
    smallest value that covers the E2E extensions.

* If there are no E2E extensions, then unset the X bit in the header.

* If there are any extensions after the E2E extensions, interpret the first
  non-E2E extension as an OHB and update the relevant RTP header fields
  in the AAD accordingly.


## Encrypting a Packet

To encrypt a packet, the endpoint encrypts the packet using the inner (end-to-end)
cryptographic key and then encrypts using the outer (hop-by-hop) cryptographic
key.  The processes is as follows:

* Form an RTP packet.  If there are any header extensions, they MUST
  use [@!RFC5285].

* If the endpoint wishes to insert header extensions that can be
  modified by an Media Distributor, it MUST insert an OHB header
  extension at the end of any header extensions protected end-to-end
  (if any), then add any Media Distributor-modifiable header
  extensions.  In other cases, the endpoint SHOULD still insert an OHB
  header extension. The OHB MUST replicate the information found in the RTP
  header following the application of the inner cryptographic
  algorithm.  If not already set, the endpoint MUST set the X bit in
  the RTP header to 1 when introducing the OHB extension.

* Apply the inner cryptographic algorithm to the RTP packet.  If 
  encrypting RTP header extensions end-to-end, then [@!RFC6904] MUST 
  be used when encrypting the RTP packet using the inner cryptographic 
  key. 
  
* Apply the outer cryptographic algorithm to the RTP packet.  If
  encrypting RTP header extensions hop-by-hop, then [@!RFC6904] MUST
  be used when encrypting the RTP packet using the outer cryptographic
  key.

When using EKT [@I-D.ietf-perc-srtp-ekt-diet], the EKT Field comes
after the SRTP packet exactly like using EKT with any other SRTP
transform.

## Relaying a Packet

The Media Distributor has the part of the key for the outer
(hop-by-hop), but it does not have the part of the key for the (end-to-end)
cryptographic algorithm.  The cryptographic algorithm and key used to
decrypt a packet and any encrypted RTP header extensions would be the
same as those used in the endpoint's outer algorithm and key.

In order to modify a packet, the Media Distributor decrypts the
packet, modifies the packet, updates the OHB with any modifications
not already present in the OHB, and re-encrypts the packet using the
cryptographic using the outer (hop-by-hop) key.

* Apply the outer (bop-by-hop) cryptographic algorithm to decrypt the
  packet.  If decrypting RTP header extensions hop-by-hop, then
  [@!RFC6904] MUST be used.

* Change any parts of the RTP packet that the relay wishes to change
  and are allowed to be changed. 

* If a changed RTP header field is not already in the OHB, add it with
  its original value to the OHB.  A Media Distributor can add
  information to the OHB, but MUST NOT change existing information in
  the OHB.

* If the Media Distributor resets a parameter to its original value,
  it MAY drop it from the OHB as long as there are no other header
  extensions following the OHB. Note that this might result in a
  decrease in the size of the OHB.  It is also possible for the Media
  Distributor to remove the OHB entirely if all parameters in the RTP
  header are reset to their original values and no other header
  extensions follow the OHB.  If the OHB is removed and no other
  extension is present, the X bit in the RTP header MUST be set to 0.

* The Media Distributor MUST NOT delete any header extensions before
  the OHB, but MAY add, delete, or modify any that follow the OHB.

    * If the Media Distributor adds any header extensions, it must
      append them and it must maintain the order of the original
      header extensions in the [@!RFC5285] block.
    
    * If the Media Distributor appends header extensions, then it MUST
      add the OHB header extension (if not present), even if the OHB
      merely replicates the original header field values, and append
      the new extensions following the OHB.  The OHB serves as a
      demarcation point between original RTP header extensions
      introduced by the endpoint and those introduced by a Media
      Distributor.
    
* The Media Distributor MAY modify any header extension appearing
  after the OHB, but MUST NOT modify header extensions that are
  present before the OHB.

* Apply the outer (hop-by-hop) cryptographic algorithm to the packet. If the RTP Sequence
  Number has been modified, SRTP processing happens as defined in SRTP
  and will end up using the new Sequence Number. If encrypting RTP
  header extensions hop-by-hop, then [@!RFC6904] MUST be used.

## Decrypting a Packet

To decrypt a packet, the endpoint first decrypts and verifies using
the outer (hop-by-hop) cryptographic key, then uses the OHB to
reconstruct the original packet, which it decrypts and verifies with
the inner (end-to-end) cryptographic key.

* Apply the outer cryptographic algorithm to the packet.  If the
  integrity check does not pass, discard the packet.  The result of
  this is referred to as the outer SRTP packet.  If decrypting RTP
  header extensions hop-by-hop, then [@!RFC6904] MUST be used when
  decrypting the RTP packet using the outer cryptographic key.

* Form a new synthetic SRTP packet with:

  * Header = Received header, with header fields replaced with values
    from OHB (if present).

  * Insert all header extensions up to the OHB extension, but exclude
    the OHB and any header extensions that follow the OHB.  If there
    are no extensions remaining, then the X bit MUST bet set to 0.  If
    there are extensions remaining, then the remaining extensions MUST
    be padded to the first 32-bit boundary and the overall length of
    the header extensions adjusted accordingly.

  * Payload is the encrypted payload from the outer SRTP packet.

* Apply the inner cryptographic algorithm to this synthetic SRTP
  packet.  Note if the RTP Sequence Number was changed by the Media
  Distributor, the synthetic packet has the original Sequence
  Number. If the integrity check does not pass, discard the packet.
  If decrypting RTP header extensions end-to-end, then [@!RFC6904]
  MUST be used when decrypting the RTP packet using the inner
  cryptographic key.

Once the packet has been successfully decrypted, the application needs
to be careful about which information it uses to get the correct
behaviour.  The application MUST use only the information found in the
synthetic SRTP packet and MUST NOT use the other data that was in the
outer SRTP packet with the following exceptions:

* The PT from the outer SRTP packet is used for normal matching to SDP
  and codec selection.

* The sequence number from the outer SRTP packet is used for normal
RTP ordering.

The PT and sequence number from the inner SRTP packet can be used for
collection of various statistics. 

If any of the following RTP headers extensions are found in the outer
SRTP packet, they MAY be used:

* Mixer-to-client audio level indicators (See [@RFC6465])


# RTCP Operations

Unlike RTP, which is encrypted both hop-by-hop and end-to-end using
two separate cryptographic key, RTCP is encrypted using only the
outer (hop-by-hop) cryptographic key.  The procedures for RTCP encryption
are specified in [@!RFC3711] and this document introduces no
additional steps.

# Use with Other RTP Mechanisms

There are some RTP related extensions that need special consideration
to be used by a relay when using the double transform due to the
end-to-end protection of the RTP.

## RTX

TODO - Add text to explain how to use RTX as described in Option A of
slides presented at IETF 99.

## RED

TODO - Add text to explain how to use RED as described in Option A of
slides presented at IETF 99.

## FEC

TODO - Add text to explain how to use FlexFEC
[@I-D.ietf-payload-flexible-fec-scheme] as described in Option A of
slides presented at IETF 99.

## DTMF

When DTMF is sent with [@RFC4733], it is end-to-end encrypted and the
relay can not read it so it can not be used to controll the
relay. Other out of band methods to controll the relay need to be used
instead.


# Recommended Inner and Outer Cryptographic Algorithms

This specification recommends and defines AES-GCM as both the inner
and outer cryptographic algorithms, identified as
DOUBLE_AEAD_AES_128_GCM_AEAD_AES_128_GCM and
DOUBLE_AEAD_AES_256_GCM_AEAD_AES_256_GCM.  These algorithm provide
for authenticated encryption and will consume additional processing
time double-encrypting for hop-by-hop and end-to-end.  However, the approach is
secure and simple, and is thus viewed as an acceptable trade-off in
processing efficiency.

Note that names for the cryptographic transforms are of the form
DOUBLE_(inner algorithm)_(outer algorithm).

While this document only defines a profile based on AES-GCM, it is
possible for future documents to define further profiles with
different inner and outer crypto in this same framework.  For
example, if a new SRTP transform was defined that encrypts some or all
of the RTP header, it would be reasonable for systems to have the
option of using that for the outer algorithm.  Similarly, if a new
transform was defined that provided only integrity, that would also be
reasonable to use for the hop-by-hop as the payload data is already encrypted
by the end-to-end.

The AES-GCM cryptographic algorithm introduces an additional 16 octets
to the length of the packet.  When using AES-GCM for both the inner
and outer cryptographic algorithms, the total additional length is 32
octets.  If no other header extensions are present in the packet and
the OHB is introduced, that will consume an additional 8 octets.  If
other extensions are already present, the OHB will consume up to 4
additional octets.


# Security Considerations

To summarize what is encrypted and authenticated, we will refer to all
the RTP fields and headers created by the sender and before the pay
load as the initial envelope and the RTP payload information with the
media as the payload. Any additional headers added by the Media
Distributor are referred to as the extra envelope. The sender uses the
end-to-end key to encrypts the payload and authenticate the payload + initial
envelope which using an AEAD cipher results in a slight longer new
payload.  Then the sender uses the hop-by-hop key to encrypt the new payload
and authenticate the initial envelope and new payload.

The Media Distributor has the hop-by-hop key so it can check the
authentication of the received packet across the initial envelope and
payload data but it can't decrypt the payload as it does not have the
end-to-end key. It can add extra envelope information. It then authenticates
the initial plus extra envelope information plus payload with a hop-by-hop
key. This hop-by-hop for the outgoing packet is typically different than the
hop-by-hop key for the incoming packet.

The receiver can check the authentication of the initial and extra
envelope information.  This, along with the OHB, is used to construct
a synthetic packet that is should be identical to one the sender
created and the receiver can check that it is identical and then
decrypt the original payload.

The end result is that if the authentications succeed, the receiver
knows exactly what the original sender sent, as well as exactly which
modifications were made by the Media Distributor.

It is obviously critical that the intermediary has only the outer
(hop-by-hop) algorithm key and not the half of the key for the the
inner (end-to-end) algorithm.  We rely on an external key management
protocol to assure this property.

Modifications by the intermediary result in the recipient getting two
values for changed parameters (original and modified).  The recipient
will have to choose which to use; there is risk in using either that
depends on the session setup.

The security properties for both the inner (end-to-end) and outer
(hop-by-hop) key holders are the same as the security properties of
classic SRTP.

# IANA Considerations

## RTP Header Extension

This document defines a new extension URI in the RTP Compact Header
Extensions part of the Real-Time Transport Protocol (RTP) Parameters
registry, according to the following data:

Extension URI: urn:ietf:params:rtp-hdrext:ohb

Description:   Original Header Block

Contact: Cullen Jennings <fluffy@iii.ca>

Reference:     RFCXXXX

Note to RFC Editor: Replace RFCXXXX with the RFC number of this
specification.
      

## DTLS-SRTP

We request IANA to add the following values to defines a DTLS-SRTP
"SRTP Protection Profile" defined in [@!RFC5764].

| Value  | Profile                                  | Reference |
|:-------|:-----------------------------------------|:----------|
|  {0x00, 0x09}  | DOUBLE_AEAD_AES_128_GCM_AEAD_AES_128_GCM | RFCXXXX   |
|  {0x00, 0x0A}  | DOUBLE_AEAD_AES_256_GCM_AEAD_AES_256_GCM | RFCXXXX   |

Note to IANA: Please assign value RFCXXXX and update table to point at
this RFC for these values.

The SRTP transform parameters for each of these protection are:

{align="left"}
~~~~
DOUBLE_AEAD_AES_128_GCM_AEAD_AES_128_GCM
    cipher:                 AES_128_GCM then AES_128_GCM 
    cipher_key_length:      256 bits
    cipher_salt_length:     192 bits
    aead_auth_tag_length:   32 octets
    auth_function:          NULL
    auth_key_length:        N/A
    auth_tag_length:        N/A
    maximum lifetime:       at most 2^31 SRTCP packets and
                            at most 2^48 SRTP packets

DOUBLE_AEAD_AES_256_GCM_AEAD_AES_256_GCM
    cipher:                 AES_256_GCM then AES_256_GCM 
    cipher_key_length:      512 bits
    cipher_salt_length:     192 bits
    aead_auth_tag_length:   32 octets
    auth_function:          NULL
    auth_key_length:        N/A
    auth_tag_length:        N/A
    maximum lifetime:       at most 2^31 SRTCP packets and
                            at most 2^48 SRTP packets
~~~~

The first half of the key and salt is used for the inner (end-to-end)
algorithm and the second half is used for the outer (hop-by-hop) algorithm.

# Acknowledgments

Many thanks to Richard Barnes for sending significant text for this
specification. Thank you for reviews and improvements from David
Benham, Paul Jones, Suhas Nandakumar, Nils Ohlmeier, and Magnus
Westerlund.

 
{backmatter}

