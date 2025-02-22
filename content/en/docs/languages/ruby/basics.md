---
title: Basics tutorial
description: A basic tutorial introduction to gRPC in Ruby.
weight: 50
---

This tutorial provides a basic Ruby programmer's introduction to working with gRPC.

By walking through this example you'll learn how to:

- Define a service in a .proto file.
- Generate server and client code using the protocol buffer compiler.
- Use the Ruby gRPC API to write a simple client and server for your service.

It assumes that you have read the [Introduction to gRPC](/docs/what-is-grpc/introduction/) and are familiar
with [protocol
buffers](https://protobuf.dev/overview). Note
that the example in this tutorial uses the proto3 version of the protocol
buffers language: you can find out more in
the [proto3 language
guide](https://protobuf.dev/programming-guides/proto3).

### Why use gRPC?

{{< why-grpc >}}

### Example code and setup

The example code for our tutorial is in
[grpc/grpc/examples/ruby/route_guide](https://github.com/grpc/grpc/tree/{{< param grpc_vers.core >}}/examples/ruby/route_guide).
To download the example, clone the `grpc` repository by running the following
command:

```sh
git clone -b {{< param grpc_vers.core >}} --depth 1 --shallow-submodules https://github.com/grpc/grpc
cd grpc
```

Then change your current directory to `examples/ruby/route_guide`:

```sh
cd examples/ruby/route_guide
```

You also should have the relevant tools installed to generate the server and
client interface code - if you don't already, follow the setup instructions in
[Quick start](../quickstart/).

### Defining the service

Our first step (as you'll know from the [Introduction to gRPC](/docs/what-is-grpc/introduction/)) is to
define the gRPC *service* and the method *request* and *response* types using
[protocol
buffers](https://protobuf.dev/overview). You can
see the complete .proto file in
[`examples/protos/route_guide.proto`](https://github.com/grpc/grpc/blob/{{< param grpc_vers.core >}}/examples/protos/route_guide.proto).

To define a service, you specify a named `service` in your .proto file:

```protobuf
service RouteGuide {
   ...
}
```

Then you define `rpc` methods inside your service definition, specifying their
request and response types. gRPC lets you define four kinds of service method,
all of which are used in the `RouteGuide` service:

- A *simple RPC* where the client sends a request to the server using the stub
  and waits for a response to come back, just like a normal function call.

  ```protobuf
  // Obtains the feature at a given position.
  rpc GetFeature(Point) returns (Feature) {}
  ```

- A *server-side streaming RPC* where the client sends a request to the server
  and gets a stream to read a sequence of messages back. The client reads from
  the returned stream until there are no more messages. As you can see in our
  example, you specify a server-side streaming method by placing the `stream`
  keyword before the *response* type.

  ```protobuf
  // Obtains the Features available within the given Rectangle.  Results are
  // streamed rather than returned at once (e.g. in a response message with a
  // repeated field), as the rectangle may cover a large area and contain a
  // huge number of features.
  rpc ListFeatures(Rectangle) returns (stream Feature) {}
  ```

- A *client-side streaming RPC* where the client writes a sequence of messages
  and sends them to the server, again using a provided stream. Once the client
  has finished writing the messages, it waits for the server to read them all
  and return its response. You specify a client-side streaming method by placing
  the `stream` keyword before the *request* type.

  ```protobuf
  // Accepts a stream of Points on a route being traversed, returning a
  // RouteSummary when traversal is completed.
  rpc RecordRoute(stream Point) returns (RouteSummary) {}
  ```

- A *bidirectional streaming RPC* where both sides send a sequence of messages
  using a read-write stream. The two streams operate independently, so clients
  and servers can read and write in whatever order they like: for example, the
  server could wait to receive all the client messages before writing its
  responses, or it could alternately read a message then write a message, or
  some other combination of reads and writes. The order of messages in each
  stream is preserved. You specify this type of method by placing the `stream`
  keyword before both the request and the response.

  ```protobuf
  // Accepts a stream of RouteNotes sent while a route is being traversed,
  // while receiving other RouteNotes (e.g. from other users).
  rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
  ```

Our `.proto` file also contains protocol buffer message type definitions for all
the request and response types used in our service methods - for example, here's
the `Point` message type:

```protobuf
// Points are represented as latitude-longitude pairs in the E7 representation
// (degrees multiplied by 10**7 and rounded to the nearest integer).
// Latitudes should be in the range +/- 90 degrees and longitude should be in
// the range +/- 180 degrees (inclusive).
message Point {
  int32 latitude = 1;
  int32 longitude = 2;
}
```

### Generating client and server code

Next we need to generate the gRPC client and server interfaces from our .proto
service definition. We do this using the protocol buffer compiler `protoc` with
a special gRPC Ruby plugin.

If you want to run this yourself, make sure you have installed
[gRPC](../quickstart/#grpc) and [protoc](../quickstart/#grpc-tools).

Once that's done, the following command can be used to generate the ruby code.

```sh
grpc_tools_ruby_protoc -I ../../protos --ruby_out=../lib --grpc_out=../lib ../../protos/route_guide.proto
```

Running this command regenerates the following files in the lib directory:

- `lib/route_guide.pb` defines a module `Examples::RouteGuide`
  - This contain all the protocol buffer code to populate, serialize, and
    retrieve our request and response message types
- `lib/route_guide_services.pb`, extends `Examples::RouteGuide` with stub and
  service classes
   - a class `Service` for use as a base class when defining RouteGuide service
     implementations
   - a class `Stub` that can be used to access remote RouteGuide instances

### Creating the server {#server}

First let's look at how we create a `RouteGuide` server. If you're only
interested in creating gRPC clients, you can skip this section and go straight
to [Creating the client](#client) (though you might find it interesting
anyway!).

There are two parts to making our `RouteGuide` service do its job:
- Implementing the service interface generated from our service definition:
  doing the actual "work" of our service.
- Running a gRPC server to listen for requests from clients and return the
  service responses.

You can find our example `RouteGuide` server in
[examples/ruby/route_guide/route_guide_server.rb](https://github.com/grpc/grpc/blob/{{< param grpc_vers.core >}}/examples/ruby/route_guide/route_guide_server.rb).
Let's take a closer look at how it works.

#### Implementing RouteGuide

As you can see, our server has a `ServerImpl` class that extends the generated
`RouteGuide::Service`:

```ruby
# ServerImpl provides an implementation of the RouteGuide service.
class ServerImpl < RouteGuide::Service
```

`ServerImpl` implements all our service methods. Let's look at the simplest type
first, `GetFeature`, which just gets a `Point` from the client and returns the
corresponding feature information from its database in a `Feature`.

```ruby
def get_feature(point, _call)
  name = @feature_db[{
    'longitude' => point.longitude,
    'latitude' => point.latitude }] || ''
  Feature.new(location: point, name: name)
end
```

The method is passed a _call for the RPC, the client's `Point` protocol buffer
request, and returns a `Feature` protocol buffer. In the method we create the
`Feature` with the appropriate information, and then `return` it.

Now let's look at something a bit more complicated - a streaming RPC.
`ListFeatures` is a server-side streaming RPC, so we need to send back multiple
`Feature`s to our client.

```ruby
# in ServerImpl

  def list_features(rectangle, _call)
    RectangleEnum.new(@feature_db, rectangle).each
  end
```

As you can see, here the request object is a `Rectangle` in which our client
wants to find `Feature`s, but instead of returning a simple response we need to
return an [Enumerator](https://ruby-doc.org//core-2.2.0/Enumerator.html) that
yields the responses. In the method, we use a helper class `RectangleEnum`, to
act as an Enumerator implementation.

Similarly, the client-side streaming method `record_route` uses an
[Enumerable](https://ruby-doc.org//core-2.2.0/Enumerable.html), but here it's
obtained from the call object, which we've ignored in the earlier examples.
`call.each_remote_read` yields each message sent by the client in turn.

```ruby
call.each_remote_read do |point|
  ...
end
```
Finally, let's look at our bidirectional streaming RPC `route_chat`.

```ruby
def route_chat(notes)
  RouteChatEnumerator.new(notes, @received_notes).each_item
end
```

Here the method receives an
[Enumerable](https://ruby-doc.org//core-2.2.0/Enumerable.html), but also returns
an [Enumerator](https://ruby-doc.org//core-2.2.0/Enumerator.html) that yields the
responses. Although each side will always get the other's messages in the order they were written,
both the client and server can read and write in any order — the streams operate completely
independently.

#### Starting the server

Once we've implemented all our methods, we also need to start up a gRPC server
so that clients can actually use our service. The following snippet shows how we
do this for our `RouteGuide` service:

```ruby
port = '0.0.0.0:50051'
s = GRPC::RpcServer.new
s.add_http2_port(port, :this_port_is_insecure)
GRPC.logger.info("... running insecurely on #{port}")
s.handle(ServerImpl.new(feature_db))
# Runs the server with SIGHUP, SIGINT and SIGQUIT signal handlers to
#   gracefully shutdown.
# User could also choose to run server via call to run_till_terminated
s.run_till_terminated_or_interrupted([1, 'int', 'SIGQUIT'])
```
As you can see, we build and start our server using a `GRPC::RpcServer`. To do
this, we:

1. Create an instance of our service implementation class `ServerImpl`.
1. Specify the address and port we want to use to listen for client requests
   using the builder's `add_http2_port` method.
1. Register our service implementation with the `GRPC::RpcServer`.
1. Call `run` on the`GRPC::RpcServer` to create and start an RPC server for our
   service.

### Creating the client {#client}

In this section, we'll look at creating a Ruby client for our `RouteGuide`
service. You can see our complete example client code in
[examples/ruby/route_guide/route_guide_client.rb](https://github.com/grpc/grpc/blob/{{< param grpc_vers.core >}}/examples/ruby/route_guide/route_guide_client.rb).

#### Creating a stub

To call service methods, we first need to create a *stub*.

We use the `Stub` class of the `RouteGuide` module generated from our .proto.

```ruby
stub = RouteGuide::Stub.new('localhost:50051')
```

#### Calling service methods

Now let's look at how we call our service methods. Note that the gRPC Ruby only
provides  *blocking/synchronous* versions of each method: this means that the
RPC call waits for the server to respond, and will either return a response or
raise an exception.

##### Simple RPC

Calling the simple RPC `GetFeature` is nearly as straightforward as calling a
local method.

```ruby
GET_FEATURE_POINTS = [
  Point.new(latitude:  409_146_138, longitude: -746_188_906),
  Point.new(latitude:  0, longitude: 0)
]
..
  GET_FEATURE_POINTS.each do |pt|
    resp = stub.get_feature(pt)
	...
    p "- found '#{resp.name}' at #{pt.inspect}"
  end
```

As you can see, we create and populate a request protocol buffer object (in our
case `Point`), and create a response protocol buffer object for the server to
fill in.  Finally, we call the method on the stub, passing it the context,
request, and response. If the method returns `OK`, then we can read the response
information from the server from our response object.


##### Streaming RPCs

Now let's look at our streaming methods. If you've already read [Creating the
server](#server) some of this may look very familiar - streaming RPCs are
implemented in a similar way on both sides. Here's where we call the server-side
streaming method `list_features`, which returns an `Enumerable` of `Features`.

```ruby
resps = stub.list_features(LIST_FEATURES_RECT)
resps.each do |r|
  p "- found '#{r.name}' at #{r.location.inspect}"
end
```

Non-blocking usage of the RPC stream can be achieved with multiple threads and 
the `return_op: true` flag. When passing the `return_op: true` flag, the 
execution of the RPC is deferred and an `Operation` object is returned. The RPC 
can then be executed in another thread by calling the operation `execute` 
function. The main thread can utilize contextual methods and getters such as 
`status`, `cancelled?`, and `cancel` to manage the RPC. This can be useful for 
persistent or long running RPC sessions that would block the main thread for an
unacceptable period of time.


```ruby
op = stub.list_features(LIST_FEATURES_RECT, return_op: true)
Thread.new do 
  resps = op.execute
  resps.each do |r|
    p "- found '#{r.name}' at #{r.location.inspect}"
  end
rescue GRPC::Cancelled => e
  p "operation cancel called - #{e}"
end

# controls for the operation
op.status
op.cancelled?
op.cancel # attempts to cancel the RPC with a GRPC::Cancelled status; there's a fundamental race condition where cancelling the RPC can race against RPC termination for a different reason - invoking `cancel` doesn't necessarily guarantee a `Cancelled` status
```


The client-side streaming method `record_route` is similar, except there we pass
the server an `Enumerable`.

```ruby
...
reqs = RandomRoute.new(features, points_on_route)
resp = stub.record_route(reqs.each)
...
```

Finally, let's look at our bidirectional streaming RPC `route_chat`. In this
case, we pass `Enumerable` to the method and get back an `Enumerable`.

```ruby
sleeping_enumerator = SleepingEnumerator.new(ROUTE_CHAT_NOTES, 1)
stub.route_chat(sleeping_enumerator.each_item) { |r| p "received #{r.inspect}" }
```

Although it's not shown well by this example, each enumerable is independent of
the other - both the client and server can read and write in any order — the
streams operate completely independently.

### Try it out!

Work from the example directory:

```sh
cd examples/ruby
```

Build the client and server:

```sh
gem install bundler && bundle install
```

Run the server:

```sh
bundle exec route_guide/route_guide_server.rb ../python/route_guide/route_guide_db.json
```

{{% alert title="Note" color="info" %}}
The `route_guide_db.json` file is actually language-agnostic, it happens to be located in the `python` folder.
{{% /alert %}}

From a different terminal, run the client:

```sh
bundle exec route_guide/route_guide_client.rb ../python/route_guide/route_guide_db.json
```
