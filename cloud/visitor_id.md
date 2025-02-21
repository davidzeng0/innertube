## Service

```protobuf
service VisitorIdService {
	rpc GetVisitorId(VisitorIdRequest) returns (VisitorIdResponse) {
		option (google.api.http) = {
			post: "/visitor_id",
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

## Response

- visitor id: the generated visitor id
  body: `.response_context.visitor_data`