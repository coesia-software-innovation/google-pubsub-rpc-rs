# Rust GooglePubSub RPC

The following repo is an example of how to use the Google PubSub proto files in Rust to call the GRPC endpoint.
I am a beginner in Rust. I have not created a library crate for this but instead want to help others out with a set of working examples.

In this code base you should find 2 binary crates. One publisher and one subscriber.

The publisher crate includes support for connecting and publishing to Google's pubsub emulator.

Hopefully this can be used as an alternative to the REST/HTTP API based crate from Byron. [Google-apis-rs](https://github.com/Byron/google-apis-rs)

Presently I have not found any Rust client for Google Pubsub, Google provided client libraries so far only for 
Go, Python, Java, and C#.

Note that this is a barebones example. I have not attempted to wrap any of the calls into something more user friendly so use at your own peril.

Disclaimer: I have no affiliation with Google.

## Quick starts with Pubsub emulator or Google Cloud Pubsub

The following shows how to get started with the examples quickly we can build the pubsub emulator docker image and the rust executables via the following snippet.

By default the publisher will create and send a message to the topic every 30 seconds (this is configurable via the `--interval` switch).

```bash
docker build . -t pubsub-emu
docker run --rm -d -p 8085:8085 -it pubsub-emu
cargo build
```
### Pubsub emulator

```bash
PUBSUB_EMULATOR_HOST='localhost:8085' target/debug/subscriber --topic projects/testproj/topics/test
PUBSUB_EMULATOR_HOST='localhost:8085' target/debug/publisher --topic projects/testproj/topics/test
```

### Google Cloud pubsub

In order to get your messages to the real pubsub service hosted at Google you need to perform a few steps manually in the cloud platform. Log into your Google cloud console and perform the following tasks:

1. Create a service account on Google IAM, Download the json file and save it.
2. Create a topic in Pubsub, don't forget to give your service account permissions to publish messages.
 

```bash
GOOGLE_APPLICATION_CREDENTIALS="/path/to/service/account.json" target/debug/publisher --topic projects/cloud-xxxx/topics/xxxx
GOOGLE_APPLICATION_CREDENTIALS="/path/to/service/account.json" target/debug/subscriber --topic projects/cloud-xxxx/topics/xxxx
```

## GRPC implementation

This example uses Grpcio rust implementation found [here](https://github.com/pingcap/grpc-rs/).

I tried using the popular GRPC rust implementation from Stephancheg (https://github.com/stepancheg/grpc-rust) but could not successfully execute requests towards the Google endpoint.
The internal http client would die with no helpful error message. Additionally it did not support connections to endpoints that respond with multiple addresses. 

### Google/Protobuf Proto files
A copy of the Google proto files from the [google-repo](https://github.com/googleapis/googleapis/tree/master/google)
The following files were used:
* v1/pubsub.proto
* google/api/annotations.proto
* google/api/http.proto
* protobuf/empty.proto

The following is a simple example proto for sending a custom message

* example.proto  - a custom proto file containing message definitions

## Dependencies
* Protoc installed and in your PATH variable. see the following for more details on how to install it (https://github.com/protocolbuffers/protobuf/blob/master/src/README.md)

## Building

This code relies on a cargo build script `build.rs` which will take the proto files and create a folder in src/proto with all the compiled output. 
Should there be an issue building this example you might have to create the folder `src/proto`. I use the excellent grpc-rs compiler [protoc-grpcio](https://github.com/mtp401/protoc-grpcio)

## Google Credentials and authentication

In order to authenticate to the endpoint we need to use the following environmental variable `GOOGLE_APPLICATION_CREDENTIALS`
This must be set to your service account json file created from the Google Cloud Platform. Save this to disk somewhere. 

Either export it to terminal(if using bash) via 
```bash
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service/account.json"
```
or you can run it on the commandline should you not wish to export it.

```bash
GOOGLE_APPLICATION_CREDENTIALS="/path/to/service/account.json" cargo run --bin publisher -- --topic projects/test-123/topics/test-topic
```

### PubSub Emulator

Google has a pubsub emulator you can use to develop locally. 
There is a dockerfile in the repo available for installing and using the emulator
Please refer to the following commands for running it.

```bash
docker build . -t pubsub-emu
docker run --rm -d -p 8085:8085 -it pubsub-emu
```

This example supports the pubsub emulator via the following environmental variable. `PUBSUB_EMULATOR_HOST` It will check for the existance of a topic given on the command line and try to create it if it does not exist.
Topic as per documenation must be in the following format: projects/{project}/topics/{topic}

Note: for the emulator the `{project}` can be any alphanumeric string. 

[Topic details](https://cloud.google.com/pubsub/docs/reference/rpc/google.pubsub.v1#google.pubsub.v1.Topic)

You may use the following environmental variable to set the application to publish to the pubsub emulator
```bash
export PUBSUB_EMULATOR_HOST='localhost:8085'
```

Should you not wish to set an environmental variable you can use the following command line switch `--emulator <host:port>`

```bash
cargo run --bin publisher -- --topic projects/cloud-xxxxx/topics/xxxx --emulator localhost:8085
```

## Running the Publisher

Before you run this example application please insure that you have a topic already created in Google PubSub or that the pubsub emulator is running somewhere that is accessible.
More help can be obtained via the `cargo run  --bin publisher -- --help` flag.

You can control the delay between publishing messages (in seconds) by setting the command line switch `--interval <x>` where x is a number. By default this value is set to 30 seconds.

To run towards Google pubsub simply run with just the `--topic` switch

```bash
cargo run --bin publisher -- --topic projects/cloud-xxxxx/topics/xxxx

Success! message_ids: "XXXXXXXX"
```

To run towards the Google pubsub emulator include the environmental variable  or the switch `--emulator localhost:8085`

*OBS! If no subscriptions are set on the topic, messages will be published but it is like throwing them into /dev/null, to get around this just run an instance of the subscriber*

## Running the Subscriber

The subscriber uses Streaming Pull to recieve messages from the PubSub topic. Messages will come in as they are produced by the publisher.
It demonstrates how one can set up a subscription and then process messages as they get published to the PubSub topic. 

The subscriber code will first attempt to make a subscription with the name given through the commandline switch `--subname` or generate a random one similar to docker if no name is set. 
These subscription names must be unique and in the format of `projects/{projectid}/subscriptions/{subname}`

Should there be an existing subscription on the topic, the subscriber example will attempt at least once to delete and recreate the subscription. There is however a `get_subscription` function which could be used to obtain an existing one.
However I did want to show an example of the deletion being used.


```bash
cargo run --bin subscriber -- --topic projects/cloud-xxxxx/topics/xxxx --emulator localhost:8085
Streaming Client Example
Subscription created successfully: name: "projects/cloud-xxxxx/subscriptions/cool_kournikova" topic: "projects/cloud-xxxxx/topics/xx12" push_config {} ack_deadline_seconds: 10 message_retention_duration {seconds: 604800}
Successfully started sending the request
Recieving messages...
[ack_id:"projects/cloud-xxxxx/subscriptions/cool_kournikova" message: {data: "\n\010test-123\020\014\032\001X" message_id: "2291" publish_time {seconds: 155481415661}}]
id: "test-123" code: 12 type: "X"
```
## Checking results on Google Cloud Platform

Google Cloud Platforms Stackdriver has metrics for the Pubsub Topics. 
Assuming your project and topics have been setup correctly you can check Resources> Cloud Pub/Sub. There should be a list of topics which you can see the number of messages published and any subscrriptions you might have. 


## Resources
* [Grpc Authentication](https://grpc.io/docs/guides/auth.html#authenticate-a-single-rpc-call-1)
* [Google PubSub Api Overview](https://cloud.google.com/pubsub/docs/reference/service_apis_overview)
* [Google PubSub RPC reference](https://cloud.google.com/pubsub/docs/reference/rpc/)
* [Google PubSub Emulator](https://cloud.google.com/pubsub/docs/emulator)

## Credits
* The developers @pingcap for their implementation of Grpc in rust
* @mtp401 for the programmatic api to the grpc-rs compiler


