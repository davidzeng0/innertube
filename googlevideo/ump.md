# UMP Format

Thanks to contributors to [gsuberland/UMP_Format](https://github.com/gsuberland/UMP_Format/blob/main/UMP_Format.md) for kickstarting this repository.<br>
Original document (derived from gsuberland): [davidzeng0/UMP_Format](https://github.com/davidzeng0/UMP_Format/blob/main/UMP_Format.md)

The UMP (likely Universal Media Playback or U(stream/streamer) Multi Part) format is used as the response format by all Google services that play videos. This document details the format for the purposes of interoperability.

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

## Part types

See [UMPPartId](../protos/video_streaming/umppart_id.proto)

### Part 10: ONESIE_HEADER

Data related to a onesie (`/initplayback`) request. See [OnesieHeader](../protos/video_streaming/onesie_header.proto).

Depending on the header type, there is exactly 0-1 ONESIE_DATA parts that follow.

See [OnesieHeaderType](../protos/video_streaming/onesie_header_type.proto) for possible values.

### Part 11: ONESIE_DATA

If present, must be preceded by OnesieHeader (part 10).

#### Type 0: OnesieHeaderType::ENCRYPTED_ONESIE_PLAYER_RESPONSE

The [OnesieHeader::cryptoParams](../protos/video_streaming/onesie_header.proto#L19) field must be set. If any of the fields mentioned below arent set, panic or throw an exception. Decrypt the contents using the same clientKey used in the [request](./initplayback.md), and the IV from crypto params.

If the compression algorithm is set, decompress accordingly.

The contents are now in protobuf. See [OnesieInnertubeResponse](../protos/youtube/api/innertube/onesie_innertube_response.proto) for the format.

If the onesie proxy status is not `OK`, panic or throw an exception.
The http status and response body can be considered a part of a normal Google Cloud API invocation.
See [errors](https://cloud.google.com/apis/design/errors) for information about API errors.
For `X-Goog-Api-Format-Version: 2`, this corresponds to `message Status`. Otherwise, it is the json `Error` format.

The body can be deserialized as a `PlayerResponse`.

#### Type 2: OnesieHeaderType::MEDIA_DECRYPTION_KEY

The body is 16 bytes. This is the key used to decrypt ONESIE_ENCRYPTED_MEDIA.
The key is sent after a successful player request/response, which may come after instances of ONESIE_ENCRYPTED_MEDIA.

#### Type 6: OnesieHeaderType::MEDIA_STREAMER_HOSTNAME

Unknown. No ONESIE_DATA follows

#### Type 14: OnesieHeaderType::RESTRICTED_FORMATS_HINT

Indicates that [OnesieHeader::restrictedFormats](../protos/video_streaming/onesie_header.proto#L23) is set. A string list of itags that needs to be converted to ints. No ONESIE_DATA follows

#### Type 16: OnesieHeaderType::STREAM_METADATA

Indicates some stream metadata is set in the header. No ONESIE_DATA follows

#### Type 25: OnesieHeaderType::ENCRYPTED_INNERTUBE_RESPONSE_PART

Parts of a GetWatchResponse.

Decrypted in the same way as OnesieHeaderType::ENCRYPTED_ONESIE_PLAYER_RESPONSE.

See [EncryptedInnertubeResponsePart](../protos/video_streaming/encrypted_innertube_response_part.proto)

### Part 12: ONESIE_ENCRYPTED_MEDIA

Starts with a UMP varint that corresponds to [MediaHeader::headerId](../protos/video_streaming/media_header.proto#L23). The rest is encrypted media.

The key is found in `OnesieHeaderType::ONESIE_MEDIA_DECRYPTION_KEY`.
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

### Part 32: HOSTNAME_CHANGE_HINT (deprecated)

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

See [FormatInitializationMetadata](../protos/video_streaming/format_initialization_metadata.proto)

### Part 43: SABR_REDIRECT

Contains a url. The client should use this url for further requests.

### Part 44: SABR_ERROR

A server abr related error.

### Part 45: SABR_SEEK

See [SabrSeek](../protos/video_streaming/sabr_seek.proto)

### Part 46: RELOAD_PLAYER_RESPONSE

The streams have expired and a new `/player` request needs to be sent.

See [ReloadPlayerResponse](../protos/video_streaming/reload_player_response.proto)

### Part 47: PLAYBACK_START_POLICY

Unknown purpose and format.

### Part 48: ALLOWED_CACHED_FORMATS

Unknown purpose and format.

### Part 49: START_BW_SAMPLING_HINT

Probably related to [BandwidthSamplingConfig](../protos/video_streaming/bandwidth_sampling_config.proto)

### Part 50: PAUSE_BW_SAMPLING_HINT

Unknown purpose and format.

### Part 51: SELECTABLE_FORMATS

See [SelectableFormats](../protos/video_streaming/selectable_formats.proto)

### Part 52: REQUEST_IDENTIFIER

See [RequestIdentifier](../protos/video_streaming/request_identifier.proto)

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
