# Introduction
Transmitting satellites broadcast LoRa message-sequences. Each sequence consists
of a wakeup-frame, an optional wakeup signature frame, and zero or more dataframes.
Some of the dataframes will contain almanac-data, but there may be other types
of dataframes as well. Finally there is an end-of-sequence frame.

Satellites may broadcast message-sequences on one or more frequencies. When broadcasting
on more than one frequency, the satellite will alternate sequences between frequencies.

An example of a satellite broadcasting on three different frequencies looks like this:

```
     -------------- frequency -------------------------->

    |         +-----------------+
    |         |   Wakeup frame  |
   time       +-----------------+
    | 
    |         +-----------------+
    |         |   Almanac data  |
    |         +-----------------+
    |         |   Almanac data  |
    |         +-----------------+
    |         | End of sequence |
    |         +-----------------+
    |
    |                             +-----------------+
    |                             |   Wakeup frame  |
    |                             +-----------------+
    | 
    |                             +-----------------+
    |                             |   Almanac data  |
    |                             +-----------------+
    |                             |   Almanac data  |
    |                             +-----------------+
    |                             | End of sequence |
    |                             +-----------------+
    |
    |                                                 +-----------------+
    |                                                 |   Wakeup frame  |
    |                                                 +-----------------+
    | 
    |                                                 +-----------------+
    |                                                 |   Almanac data  |
    |                                                 +-----------------+
    |                                                 |   Almanac data  |
    |                                                 +-----------------+
    |                                                 | End of sequence |
    |                                                 +-----------------+
    |
    |         +-----------------+
    |         |   Wakeup frame  |
    |         +-----------------+
    | 
    |         +-----------------+
    |         |   Almanac data  |
    |         +-----------------+
    |         |   .....         |
    V
```

A few remarks about the example (concepts discussed in more detail later):
* In the example, there is a pause after the wakeup frame, but not between
  consecutive almanac data frames, or between the final almanac data frame and
  the end-of-sequence frame. This is representative og the actual timing - there
  is a pause after the wakeup frame but not between data-frames.
* The example does not contain wakeup signature frames. If it would, they would 
  immediately follow the wakeup frame, without pause.
* In the example, the wakeup frame and the subsequent almanac data frames (and
  the end-of-sequence frame) are all transmitted on the same frequency. It is
  possible for the wakeup frame to contain data that indicates that the remainder
  of the sequence is broadcast on a different frequency.
* Apart from almanac data, the sequence could contain other frame-types as well.
  Those are not shown in the example.

# Frame format
Wakeup frames, wakeup signature frames, almanac data frames and end-of-sequence
frames all share a common layout (other data frames may or may not use the same
format). 

At top-level they are LoraWAN frames, but with a frame-type of `proprietary` (the
frame-type is encoded in the 3 highest bits of the MHDR field, and value `0b111`
signifies `proprietary`). They don't have a device-address or a MIC, so practically
speaking, all that makes them a LoraWAN frame is the fact that the first byte 
of the frame is `0b11100000`.

The second byte of the frame indicates the frame-type. Currently the following
frame-types are known:
* `0`: Wakeup frame
* `1`: Almanac data frame
* `2`: Wakeup signature frame
* `3`: End-of-sequence frame

## Wakeup frame format
The wakeup frame is a proprietary LoraWAN frame with a frame-type of `0`.

The wakeup frame's payload (which starts at byte 2, right after the frame-type)
consists of a fixed header followed by zero or more TLVs (type/length/value).
The fixed header has a fixed length, the length of the TLVs is variable.

### Fixed header
The fixed header consists of 5 bytes, as follows:
* Total frames following (`uint8_t`): the total number of frames
  following the wakeup frame. This includes the signature frame
  and almanac frames, if present, but may also include additional
  frames that are not announced in the wakup frame (for example
  LoraWAN multicast frames that are received autonomously by devices on the
  ground). It also includes the end-of-sequence frame.
* Transmitting satellite ID (`uint8_t`): numeric ID uniquely identifying
  the transmitting satellite. This is the same ID as used in the almanac.
* Time between wakeup frames (`uint16_t`, BE): the time in seconds between the
  start of this wakeup frame and the next.
* Time until sequence (`uint8_t`, BE): the time in seconds between the end
  of this wakeup frame and the start of the first frame of the remainder of 
  the sequence. The wakeup signature frame is _not_ considered part of the
  sequence - therefore, if a wakeup signature frame is present, the `time until
  sequence` will always be long enough to allow the wakeup signature frame to 
  be broadcast first.
  The actual time until the first frame of the sequence may be up to (but not
  including) one second longer than indicated (in other words, the time-until-sequence
  value is truncated)


### TLVs
The remainder of the wakup frame consists of TLVs. There are two TLV formats:
a short one and a long one.

The type is an integer in the range 0..70 indicating the meaning of the TLV. 
The length is the length of the value, in bytes. The value, also called payload,
 is a sequence of `length` (which may be 0) bytes, the meaning of which is
dependent on the type and is known to the receiver (provided the receiver
recognizes the TLVs type). 

When processing a wakeup frame, the receiver can process those TLVs that it knows
about, and skip those it doesn't, since their lengths can be determined without
understanding their meaning.

#### Short TLV format
TLVs with a type < 7 can be encoded by one byte containing both the type and the
length, followed by `length` bytes of payload.

The byte containing type and length is structed as follows:

    bit 7|6|5|4|3|2|1|0
        -+-+-+-+-+-+-+-
        t|t|t|l|l|l|l|l 

Here, `ttt` is the type as a 3-bit integer, and `lllll` the length of the value
as a 5-bit number.

As an example, a TLV with type 3 with payload `0x10 0x20 0x30` is encoded as

    0x63 0x10 0x20 0x30

A TLV with type 6 without a payload is encoded as

    0xc0

The maximum allowed payload length for short format TLVs is 31 bytes - TLVs
requiring a longer payload should preferably use a type >= 7, or use the long
format encoding (which is allowed even for types < 7).

#### Long TLV format
TLVs with a type >= 7 are encoded with two bytes containing the type and payload
length, followed by `length` bytes.

The two bytes containing the type and payload length are structured as follows:


    byte         1                 2
     bit 7|6|5|4|3|2|1|0 | 7|6|5|4|3|2|1|0
         -+-+-+-+-+-+-+--+--+-+-+-+-+-+-+-
         1|1|1|t|t|t|t|t | t|l|l|l|l|l|l|l

Here, bits 5..7 of the first byte have fixed values 1, to tell a long format TLV
apart from a short format TLV. The 6-bit integer formed by bits 4..0 of the first
byte and bit 7 of the second byte is the type minus 7. The 7-bit integer formed
by bits 6..0 of the second byte is the payload length.

As an example, a TLV with type 15 and payload `0x0a 0x0b 0xc` is encoded as

    0xe4 0x03 0x0a 0x0b 0x0c

The maximum payload size of a TLV using long encoding is 127 bytes.

#### Types

For the first version, the following TLV types are specified:

##### `WAKEUP_SIGNATURE_FOLLOWS`, type is `0`
If this TLV, which does not have a payload, is present, then the wakeup frame
is immediately followed by a frame containing a cryptographic signature of the
wakeup frame. The exact format of the signature frame is tbd.

##### `ALMANAC_FOLLOWS`, type is `1`
If this TLV is present, then the wakeup frame, or the wakeup signature frame
if present, is followed by 0 or more frames containing blocks of an almanac.
The payload of this TLV contains almanac metadata, which allows the receiver
to decide if it needs to receive the almanac. The payload of this TLV consists
of 16 bytes, as follows:

* Number of blocks following wakeup frame (`uint8_t`). This is just the number
  of blocks following the wakeup frame (or wakeup signature frame) which is 
  in general not enough to contain an entire almanac.
* Almanac version (`uint8_t`)
* Almanac valid from, as seconds since 1/1/1970 (`uint32_t`, BE)
* Localisation ID (`uint8_t`)
* Service provider mask (`uint16_t`, BE)
* Expected CRC (`uint32_t`, BE). This is the 32-bit integer formed by the
  first 4 bytes of the SHA-2 digest over the almanac data.
* Almanac size in bytes (`uint16_t`, BE)
* Block size (`uint8_t`, BE)

The total number of blocks used to transmit the almanac is

    total_blocks = ceiling(almanac_size/block_size)

The payload of each almanac block consists of one byte indicating the (0-based)
sequence number of the block, followed by `block_size` bytes of content
(except for the block with sequence-number `total_blocks-1` which may have a
shorter payload).

##### `TIME`, type is `2`
Current time. The payload consists of 10 bytes, as follows:
* UNIX time (`uint32_t`, BE): the number of seconds since Jan 1st, 1970 
* GPS time (`uint32_t`, BE): the number of seconds since the GPS epoch (Jan 6th,
  1980), not taking into account leap seconds.
* Milliseconds (`uint16_t`, BE): the milliseconds value, which is valid for both
  the UNIX and the GPS time.

The specified time is the time at the end of the frame.

##### `ORBIT_EXTRAPOLATION`, type is `3`
Consists of 9 24-bit numbers, and one 8-bit number being the interval over which
the extrapolation is valid. Exact format TBD.

##### `SWITCH_FREQUENCY`, type is `4`
If this TLV is present, then the frames following this header, except the wakup
signature frame if present, will be broadcast on a different frequency, and
using different modulation parameters from the wakeup frame. The payload
consists of:
* Frequency (`uint16_t`, BE): the alternate frequency divided by 50KHz.
* Lora configuration 1 (`uint8_t`):
    - bits 3-0: spreading factor
    - bits 7-4: bandwidth
* Lora configuration 2 (`uint8_t`):
    - bit 0: LDRO
    - bit 1: Invert IQ
    - bits 3-2: sync word: `0x0`: public; `0x1`: private; `0x02` and `0x03` reserved
* Preamble length (`uint16_t`, BE): length of the preamble

##### `SERVICE_PRESENCE_DURATION`, type is `5`
This TLV is intended in case the wakeupframe is received by the terminal as service
presence detection. Its payload consists of 2 bytes, being a `uint16_t` (BE) that
represents the number of seconds that the terminal is allowed to transmit after receiving
the wakeup frame.

#### Preamble length
The wakeup frame is transmitted with a LoRa preamble longer than usual. This
allows terminals to wake up periodically, see if a preamble is present, and
immediately go back to sleep if it isn't, thus saving power.

## Wakeup signature frame format
The wakeup signature frame is a proprietary LoraWAN frame with a frame-type of `2`.

Its payload (starting at byte 2 of the frame, immediately following the frame-type)
is formatted as follows:

* `signature_type`: (`uint8_t`) indicates the algorithm used for signing. Currently
                    the only value supported is `0`, meaning `SHA256+secp256r1`.
* `key_id`: (`uint8_t[4]`) identifies the keypair used for signing. The meaning depends
            on the algorithm. In case of `SHA256+secp256r1`, it consists of the
            first 4 bytes of the public key of the keypair.
* `signature`: (`uint8_t[]`) the signature. Length and meaning depends on the
               algorithm. In case of `SHA256+secp256r1`, it is the `secp256r1`
               signature over the `SHA256` digest of the entire wakeup frame,
               including the LoraWAN MHDR and frame-type, and the length is 64.

## Almanac data frame format
The almanac data frame is a proprietary LoraWAN frame with a frame-type of `1`.

The first byte of the payload (which is the second byte of the LoraWAN frame, 
directly following the frame-type) indicates the blocknumber. 

The remainder of the frame is block-data.

The wakeup-frame contains an `ALMANAC_FOLLOWS` TLV, which contains the block-size
and the total size of the almanac. The total number of blocks can be calculated
as `ceiling(total_size / block_size)`, and the blocknumber will always be smaller
than this.

The offset of the data in the block within the almanac data is simply `block_size * blocknumber`.

All blocks except for the last one (that is, the one with blocknumber `total_blocks-1`)
will be exactly of size `block_size`. The last one will generally be shorter.

## End-of-sequence frame format
The end-of-sequence frame is a proprietary LoraWAN frame with a frame-type of `2`.

The data following the frame-type is undefined - it may be empty, or it may be used
by later specifications.

# Frequency-cycling
As described in the introduction, satellites will broadcast on one or more frequencies.
The idea is that terminals will normally try to receive the sequences on all 
frequencies, but will still be able to receive all data (albeit a bit slower) if
one of the frequencies is blocked due to interference.

If broadcasting on multiple frequencies, the satellite will always start broadcasting
on the lowest frequency, broadcast the next sequence on the next-higher frequency, etc.
After broadcasting a sequence on the highest frequency, it will restart on the 
lowest frequency.

The terminal will know on how many frequencies a satellite will broadcast, and what
the interval between sequences is. 
Therefore, when listening on one frequency, it will expect to receive a frame
within a time `(N-1) * interval + margin`, where `N` is the number of frequencies,
`interval` the interval between sequences, and `margin` a certain
implementation-defined margin. 

If the terminal has not received any frames within this time, it must conclude that
apparently there is interference on that particular frequency, and try again
with the next frequency on the list.

If the terminal does receive a frame, the action it takes depends on the type
of frame.

If it is a wakeup-frame, it will parse it and use the data in the wakeup frame
to determine if it wants to receive and process some or all data of the subsequent
messages of the sequence. If so, it will receive frames until either:
* The total number of frames in the sequence (as present in the fixed header of
  the wakeup frame) has been reached, or
* The end-of-sequence frame has been received, or
* The time between wakeup frames (as present in the fixed header of the wakeup 
  frame) has been reached. This is a fallback in case on or more of the frames
  in the sequence could not be received for some reason.
After this, tht terminal will switch to the next frequency in the list, and wait
for next wakeup frame.

If it is not a wakeup-frame, the terminal has no way to know where in the sequence
it is, and therefore it will keep receiving frames until the end-of-sequence frame
is received. After having received it, it will switch to the next frequency in the
list, and start receiving. The first frame it receives now should be a wakeup frame.
