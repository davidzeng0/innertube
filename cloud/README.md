The [projects.yaml](./projects.yaml) file defines a list of known google cloud projects for which the API is accessible.<br>
This list is non-exhaustive.

The [services.yaml](./services.yaml) specifies the InnerTube API as a part of the Google Cloud ecosystem. For the client, it is a standard HTTP REST client with custom error handling. See [errors](https://cloud.google.com/apis/design/errors) for information about Google Cloud API errors.

The `protoOverRest` implementation refers to [innertube.proto](../protos/youtube/api/innertube/innertube.proto). Note that `innertube.proto` is not a part of the official API. It is intended as a starter template for integrating proto defined RPC services into your preferred language.

The current `services.yaml` was intended for use with ts-proto and mosaic-googlecloud, both in typescript, of which the latter is now discontinued, due to being RIIR and multiple limitations of the javascript language. If you are using ts-proto, generate with the following options

```bash
--ts_proto_out "${PROTO_OUTPUT_DIR}"
--ts_proto_opt "useOptionals=all"
--ts_proto_opt "env=node"
# note that the field option `jstype` is JS_STRING for longs in transit with json
# bigint is used for compatibility with the Map type
--ts_proto_opt "forceLong=bigint"
--ts_proto_opt "esModuleInterop=true"
--ts_proto_opt "unknownFields=true"
--ts_proto_opt "initializeFieldsAsUndefined=false"
--ts_proto_opt "useDate=false"
--ts_proto_opt "outputServices=generic-definitions"
--ts_proto_opt "outputServices=default"
--ts_proto_opt "outputClientImpl=false"
--ts_proto_opt "outputPartialMethods=false"
# this google cloud api uses standard camel casing for json
# --ts_proto_opt "snakeToCamel=false"
--ts_proto_opt "lowerCaseServiceMethods=true"
--ts_proto_opt "outputExtensions=true"
--ts_proto_opt "useMapType=true"
--ts_proto_opt "exportCommonSymbols=false"
--ts_proto_opt "stringEnums=true"
```