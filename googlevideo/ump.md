# UMP Format

The UMP (likely Universal Media Playback) format is used as the response format by all Google services that play videos. This document details the format for the purposes of interoperability.

## Variable sized integers

The UMP format uses variable size integers in a number of places.

The first 4 bits of the first byte set the size of the integer. Getting the size of the integer from the first byte can be implemented as follows:

```rust
fn varint_size(byte: u8) -> u32 {
	byte.leading_ones().min(4) + 1
}
```

The remainder of the bits in the first byte are used as part of the integer, except for in 5-byte integers where those bits are ignored. The variable integer decoding can be implemented as follows:

```rust
fn read_varint(input: &mut Reader) -> u32 {
	let prefix = input.read_u8();
	let size = varint_size(prefix);

	let mut shift = 0;
	let mut result = 0;

	if size != 5 {
		shift = 8 - size;

		// compute mask of prefix
		let mask = (1 << shift) - 1;

		result |= prefix as u32 & mask;
	}

	for i in 1..size {
		let byte = input.read_u8();

		result |= (byte as u32) << shift;
		shift += 8;
	}

	result
}
```

## High level structure

The UMP format requires that the Content-Type header in the HTTP response contains "application/vnd.yt-ump".

A UMP response is split into parts. Each part is prefixed by a pair of variable length integers, the first being the part type and the second being the part payload length.

```rust
fn read_part(input: &mut Reader) -> (Vec<u8>, u32) {
	let ty = read_varint(input);
	let size = read_varint(input);
	let data = input.read_bytes(size);

	(data, ty)
}
```

### Partial parts

Parts are not guaranteed to be wholly contained within one response payload. It is quite common to find that a part's length exceeds the length of the HTTP response payload. This means that the part will continue in the next response payload.

If a response payload is self-contained, i.e. it does not end with a partial part, the last part in the buffer will end precisely at the end of the buffer. Typically the last part will be `MEDIA_END` (type 22) with a single null byte payload, but this is not guaranteed.

If a payload is not self-contained, i.e. its final part has a length exceeding the amount of remaining data in the buffer, its data will continue in the next response payload. In such a case, the next payload will start with a media header part (type 20) followed by a part of the same type as the partial one, whose data is a continuation of the partial part from the previous payload. Parts can be split up over an arbitrary number of response payloads.

This is a little hard to picture, so here's an example. Let's say you've got a part with a length of 2,500,000 bytes, and each response payload can be maximum of 1MB (this is just for example; in practice there is no such hard limit). The resulting response payloads will look something like this:

```
response 1:
	part 20 (media header)
		size=...
		data=...
	part 21 (media data)
		size=2500000
		data=... (len=1000000)

response 2:
	part 20 (media header)
		size=...
		data=...
	part 21 (media data)
		size=1500000
		data=... (len=1000000)

response 3:
	part 20 (media header)
		size=...
		data=...
	part 21 (media data)
		size=500000
		data=... (len=500000)
	part 22 (MEDIA_END)
		size=1
		data=00
```

This gets decoded as a single type 21 part of size 2,500,000 bytes, followed by a type 22 part. Note that there could be a different part type in response 3, after part 21, instead of `MEDIA_END`. The end of a part is determined solely by all of its data being read; the next part type is irrelevant.

Once a partial part begins, responding with a different part type (e.g. sending a partial part 22, then following up with a part 32 before sending the rest of the first part) has undefined behaviour. As far as I could tell from the implementations, error handling is variable here. Some will throw an exception, but some appear to blindly accept the data as a continuation of the data, even if the part type ID is wrong. Fun! I would recommend being rigorous in checking for the correct type when decoding partial parts.

### Reading UMP parts

You can read UMP parts with a state machine:

1. Read a response payload in chunks.
2. If the amount of data remaining in the buffer is zero, go back to step 1.
3. Read the UMP part type as a variable sized integer.
4. Read the UMP part size as a variable sized integer.
5. If the part size is zero, decode the part as a zero-length part (i.e. no payload), passing in an empty buffer, then go back to step 2.
6. If the part size is less than or equal to the remaining buffer size, decode the part from that slice of the buffer, then increment the position within the buffer and go back to step 3.
7. If the part size is greater than the remaining buffer size, keep a copy of the remaining buffer data, read the next buffer, stitch the two together, and continue parsing from step 5.

## Part types

The following part types have been observed.

### Part 10: ONESIE_HEADER

OnesieHeader. Depending on the header type, there is exactly 0-1 ONESIE_DATA parts that follow.

See [OnesieHeaderType](../protos/video_streaming/onesie_header_type.proto)

### Part 11: ONESIE_DATA

If present, must be preceded by OnesieHeader (part 10).

#### Type 0: OnesieHeaderType::PLAYER_RESPONSE

The [OnesieHeader::cryptoParams](../protos/video_streaming/onesie_header.proto#L19) field must be set. If any of the fields mentioned below arent set, panic or throw an exception. Decrypt the contents using the same clientKey used in the [request](./initplayback.md), and the IV from crypto params.

If the compression type is BROTLI, decompress using BROTLI, otherwise decompress using GZIP.

The contents are now in protobuf.

See [OnesieInnertubeResponse](../protos/video_streaming/onesie_innertube_response.proto) and [ProxyStatus](../protos/video_streaming/proxy_status.proto)

If the proxy status is not `OK`, panic or throw an exception.
If the status is not 200, panic or throw an exception. Error codes have a response body in the same format as a Google Cloud API error response.

The body can be deserialized as a `PlayerResponse`.

#### Type 2: OnesieHeaderType::MEDIA_DECRYPTION_KEY

The body is 16 bytes. This is the key used to decrypt ONESIE_ENCRYPTED_MEDIA.
The key is sent after a successful player request/response, which may come after instances of ONESIE_ENCRYPTED_MEDIA.

#### Type 6: OnesieHeaderType::NEW_HOST

Unknown. No ONESIE_DATA follows

#### Type 14: OnesieHeaderType::RESTRICTED_FORMATS_HINT

Indicates that [OnesieHeader::restrictedFormats](../protos/video_streaming/onesie_header.proto#L23) is set. A string list of itags that needs to be converted to ints. No ONESIE_DATA follows

#### Type 16: OnesieHeaderType::STREAM_METADATA

Indicates some stream metadata is set in the header. No ONESIE_DATA follows

#### Type 25: OnesieHeaderType::ENCRYPTED_INNERTUBE_RESPONSE_PART

Unknown purpose.

Decrypted in the same way as OnesieHeaderType::PLAYER_RESPONSE.

See [EncryptedInnertubeResponsePart](../protos/video_streaming/encrypted_innertube_response_part.proto)

### Part 12: ONESIE_ENCRYPTED_MEDIA

Starts with a UMP varint that corresponds to [MediaHeader::headerId](../protos/video_streaming/media_header.proto#L19). The rest is encrypted media.

The key is found in `OnesieHeaderType::MEDIA_DECRYPTION_KEY`.
The initialization vector is 16 bytes of zeros. There is no hmac.

All ONESIE_ENCRYPTED_MEDIA chunks are to be decrypted with the same cipher instance, without resetting the IV.
Unknown if an IV reset is used if the `headerId` changes, but likely not.

This data is usually sent under the same `headerId` as MEDIA.
The data should be processed in order of receiving.

### Part 20: MEDIA_HEADER

See [MediaHeader](../protos/video_streaming/media_header.proto)

### Part 21: MEDIA

Contains the actual media itself. Starts with a UMP varint corresponding to `MediaHeader::headerId`, followed by the media data.

If the header compression type is set to GZIP, decompress using gzip.

### Part 22: MEDIA_END

Terminator part. Usually included at the end of all payloads where there is no partial payload at the end.

Size is usually 1, with the data being a UMP varint corresponding to [MediaHeader::headerId](../protos/video_streaming/media_header.proto#L19).

### Part 31: LIVE_METADATA

Metadata related to live streams.

See [LiveMetadata](../protos/video_streaming/live_metadata.proto)

### Part 32: HOSTNAME_CHANGE_HINT

Unknown format.

### Part 33: LIVE_METADATA_PROMISE

See [OnesieLiveMetadataPromise](../protos/video_streaming/onesie_live_metadata_promise.proto)

### Part 34: LIVE_METADATA_PROMISE_CANCELLATION

Cancellation of a SABR Live Metadata Promise (originally sent as part 33), related to "SABR Live Protocols".

Same format as LIVE_METADATA_PROMISE.

### Part 35: NEXT_REQUEST_POLICY

Unknown purpose and format.

### Part 36: USTREAMER_VIDEO_AND_FORMAT_DATA

Probably related to signalling the media format information for livestreams.

Unknown format.

### Part 37: FORMAT_SELECTION_CONFIG

See [FormatSelectionConfig](../protos/video_streaming/format_selection_config.proto)

### Part 38: USTREAMER_SELECTED_MEDIA_STREAM

Clearly streaming related but not sure what this is for.

Unknown format.

### Part 39: ???

Non existent (deprecated and removed or never existed in the first place).

### Part 40: ???

Non existent (deprecated and removed or never existed in the first place).

### Part 41: ???

Non existent (deprecated and removed or never existed in the first place).

### Part 42: FORMAT_INITIALIZATION_METADATA

Unknown purpose and format.

### Part 43: SABR_REDIRECT

Contains a url. The client should use this url for further requests.

### Part 44: SABR_ERROR

Error in request payload.

### Part 45: SABR_SEEK

Unknown purpose and format.

### Part 46: RELOAD_PLAYER_RESPONSE

The streams have expired and a new /player request needs to be sent.

Unknown format.

### Part 47: PLAYBACK_START_POLICY

Unknown purpose and format.

### Part 48: ALLOWED_CACHED_FORMATS

Unknown purpose and format.

### Part 49: START_BW_SAMPLING_HINT

Unknown purpose and format.

### Part 50: PAUSE_BW_SAMPLING_HINT

Unknown purpose and format.

### Part 51: SELECTABLE_FORMATS

Unknown purpose and format.

### Part 52: REQUEST_IDENTIFIER

Unknown purpose and format.

### Part 53: REQUEST_CANCELLATION_POLICY

Unknown purpose and format.

### Part 54: ONESIE_PREFETCH_REJECTION

Unknown purpose and format.

### Part 55: TIMELINE_CONTEXT

Unknown purpose and format.

### Part 56: REQUEST_PIPELINING

Unknown purpose and format.

### Part 57: SABR_CONTEXT_UPDATE

Unknown purpose and format.

### Part 58: STREAM_PROTECTION_STATUS

Unknown purpose and format.

### Part 59: SABR_CONTEXT_SENDING_POLICY

Unknown purpose and format.

### Part 60: LAWNMOWER_POLICY

Unknown purpose and format.

### Part 61: SABR_ACK

Unknown purpose and format.

### Part 62: END_OF_TRACK

See [EndOfTrack](../protos/video_streaming/end_of_track.proto)

### Part 63: CACHE_LOAD_POLICY

Unknown purpose and format.

### Part 64: LAWNMOWER_MESSAGING_POLICY

Unknown purpose and format.

### Part 65: PREWARM_CONNECTION

Generate 204s and keep alive the connection for faster streaming.
