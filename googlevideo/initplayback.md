## Request format for /initplayback

The request for this endpoint is [OnesieRequestProto](../protos/video_streaming/onesie_request_proto.proto).

Encryption is required for the player request. It can be acquired via a request to `/config`.

The formats are [ConfigRequest](../protos/youtube/api/innertube/config_request.proto) and [ConfigResponse](../protos/youtube/api/innertube/config_response.proto)

If the rawColdConfigGroup and rawHotConfigGroup are not present, then check the [ResponseContext::global_config_group](../protos/youtube/api/innertube/response_context.proto#L33). See [GlobalConfigGroup](../protos/youtube/api/innertube/global_config_group.proto) for more information. The [OnesieHotConfig](../protos/youtube/api/innertube/onesie_hot_config.proto) can be found via `HotConfigGroup::media_hot_config::onesie_hot_config`

The client uses the [OnesieHotConfig::client_key](../protos/youtube/api/innertube/onesie_hot_config.proto#L16) for encrypting the request. The IV is randomly generated.

[OnesieInnertubeRequest](../protos/youtube/api/innertube/onesie_innertube_request.proto) is the data sent in [`EncryptedInnertubeRequest::encrypted_onesie_innertube_request`](../protos/youtube/api/innertube/encrypted_innertube_request.proto#L20)

The `OnesieHotConfig::encrypted_client_key` is sent in the request, along with the hmac and iv produced by the function below. Responses are encrypted/hmaced with the same key

```rust
// the key does not deviate from this length. otherwise, consider the config response invalid
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
