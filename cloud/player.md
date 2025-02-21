## Service

```protobuf
service PlayerService {
	rpc GetPlayer(PlayerRequest) returns (PlayerResponse) {
		option (google.api.http) = {
			post: "/player",
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

## Recommended Fields

- video id: the id of the video you want<br>
  body: `.video_id`<br>
  query: `video_id`

- content check status: user wishes to see content restricted video<br>
  body: `.content_check_ok`

- racy check status: user wishes to see age restricted video<br>
  body: `.racy_check_ok`

## Response

- playability status: whether or not the video can be played<br>
  body: `.playability_status`

- streaming data: urls for playback. only present when `.playability_status.status == OK`<br>
  body: `.streaming_data`

- video details: title, description, likes, views, etc<br>
  body: `.video_details`

- player config: audio loudness config, abr config, etc<br>
  body: `.player_config`

- storyboards: scrubbing thumbnail previews<br>
  body: `.storyboards`

- endscreen: youtube video endscreen<br>
  body: `.endscreen`