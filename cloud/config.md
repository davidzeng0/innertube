## Service

```protobuf
service ConfigService {
	rpc GetConfig(ConfigRequest) returns (ConfigResponse) {
		option (google.api.http) = {
			post: "/config",
			body: "*"
		};
	}
}
```

## Required Fields

- client name: name of client accessing innertube<br>
  body: `.context.client.client_name`<br>
  header: `X-Youtube-Client-Name`

- client version: version of client specified by client name<br>
  body: `.context.client.client_version`<br>
  header: `X-Youtube-Client-Version`