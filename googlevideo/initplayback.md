## Request format for /initplayback

See [OnesieInnertubeRequest](../protos/youtube/api/innertube/onesie_innertube_request.proto).

[OnesieRequestProto](../protos/video_streaming/onesie_request_proto.proto) is the data sent in `EncryptedInnertubeRequest::encryptedOnesieRequest`

The key can be acquired via a request to /config.

See [ConfigRequest](../protos/youtube/api/innertube/config_request.proto) and [ConfigResponse](../protos/youtube/api/innertube/config_response.proto)

Usually, the rawColdConfigGroup and rawHotConfigGroup are not present. Instead, they are serialized in [ResponseContext::globalConfigGroup](../protos/youtube/api/innertube/response_context.proto#L19)

See [HotConfigGroup](../protos/youtube/api/innertube/hot_config_group.proto)

The client uses the [OnesieHotConfig::clientKey](../protos/youtube/api/innertube/onesie_hot_config.proto#L16) for encrypting the request. The IV is randomly generated.

The `OnesieHotConfig::encryptedClientKey` is sent in the request, along with the hmac and iv produced  by the function below. Responses are encrypted/hmaced with the same key

```rust
// the key does not deviate from this length. otherwise, consider the response invalid
fn encrypt_request(client_key: [u8; 32], data: &[u8]) -> (Vec<u8>, Vec<u8>) {
	let aes_key = &client_key[0..16];
	let hmac_key = &client_key[16..32];

	let iv = random_bytes(16); // 16 bytes
	let mut aes = new_aes_128_ctr(aes_key, &iv);

	let encrypted = aes.update_final(data);
	let mut hmac = new_sha_256_hmac(hmac_key);

	hmac.update(&encrypted);

	(encrypted, hmac.update_final(&iv))
}
```
