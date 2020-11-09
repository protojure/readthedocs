# Protojure

Protojure is first-class [Clojure](https://clojure.org/) support for
[Protocol Buffers](https://developers.google.com/protocol-buffers/) and [gRPC Services](https://grpc.io/)

[https://github.com/protojure](https://github.com/protojure)

## Features

* Supports Protobuf message serialization (tested with proto3 format)
* Support for GRPC Clients and Servers
* In-process [GRPC-WEB](https://github.com/grpc/grpc-web) Proxy
* core.async based GRPC streaming
* Integration with the [Pedestal](https://github.com/pedestal/pedestal) web framework included, and extensible to support others (Ring, Compojure, etc)

## Etymology

Protojure is a portmanteau of **Proto**-col Buffers and Clo-**jure**

## Prerequisites

* Protocol Buffer Compiler ([protoc](https://github.com/protocolbuffers/protobuf/releases))
* A [Clojure development environment](https://clojure.org/guides/getting_started)

## Installation

Download the latest [Release](https://github.com/protojure/protoc-plugin/releases) and make it executable in your path.

```
sudo curl -L https://github.com/protojure/protoc-plugin/releases/download/v0.9.3/protoc-gen-clojure --output /usr/local/bin/protoc-gen-clojure
sudo chmod +x /usr/local/bin/protoc-gen-clojure
```

## Clojure Docs

* [Protojure lib cljdoc](https://cljdoc.org/d/protojure/protojure)

## Contributing

Pull requests welcome.  Please be sure to include a [DCO](https://en.wikipedia.org/wiki/Developer_Certificate_of_Origin) in any commit messages.
