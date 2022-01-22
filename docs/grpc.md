## gRPC Services

gRPC is a "high performance, open-source universal RPC framework".

For those without any prior gRPC experience, gRPC is a standardized way of communicating between processes, often over
a network, whether within a data center or across the wider internet.

Below is a simple [gRPC](https://grpc.io/) service definition:

```
syntax = "proto3";
package com.example.addressbook;

message Person {
    string name = 1;
}

message AddressBook {
    repeated Person people = 1;
}

message HelloResponse {
    string message = 1;
}

service Greeter {
    rpc Hello (Person) returns (HelloResponse);
}

```

The service definition defines an endpoint (often reachable at some well-known URL or IP), called Greeter. The Greeter service exposes a method called Hello. We may interact with the Hello method by contacting the Greeter service and sending a Person message. Refer to [Protocol Buffers](#protocol-buffers) above for a walkthrough of protobuf with protojure.

The message definition of HelloResponse is just like the `message Person` definition discussed in the previous section.

For a gRPC quick-start, open a new terminal and run:

```
lein new protojure demo-server
cd demo-server && make all
lein run
```
You should now have a gRPC server running at http://localhost:8080.  We will use this endpoint for further exploration.

#### gRPC Client

With the gRPC server running as directed above, open a separate terminal and cd to a directory of your choice.  Copy the entire protobuf defined above into your current directory as `greeter.proto`

Next, run:
```
protoc --clojure_out=grpc-client:. greeter.proto
```

If we check the contents of our directory, we will now also see a folder called `com/`. Inside is our generated gRPC
client code.

```
$ tree
.
├── com
│   └── example
│       ├── addressbook
│       │   └── Greeter
│       │       └── client.cljc
│       └── addressbook.cljc
└── greeter.proto
```

We note that we passed the option `grpc-client` to the compiler.  The gRPC code generation for clients and servers are optional in Protojure.  A similar `grpc-server` option exists for server-side deployments.

Next, create another file called `project.clj` in our current directory with contents:

```
(defproject protojure-tutorial "0.0.1-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Apache License 2.0"
            :url "https://www.apache.org/licenses/LICENSE-2.0"
            :year 2022
            :key "apache-2.0"}
  :dependencies [[org.clojure/clojure "1.10.3"]

                 ;; -- PROTOC-GEN-CLOJURE --
                 [io.github.protojure/grpc-client "2.0.1"]
                 [io.github.protojure/google.protobuf "2.0.0"]]
  :source-paths ["."])

```
Save it, and run a REPL
```
$ lein repl
nREPL server started on port 34903 on host 127.0.0.1 - nrepl://127.0.0.1:34903
WARNING: cat already refers to: #'clojure.core/cat in namespace: net.cgrand.regex, being replaced by: #'net.cgrand.regex/cat
REPL-y 0.3.7, nREPL 0.2.12
Clojure 1.10.0
OpenJDK 64-Bit Server VM 1.8.0_222-8u222-b10-1ubuntu1~18.04.1-b10
    Docs: (doc function-name-here)
          (find-doc "part-of-name-here")
  Source: (source function-name-here)
 Javadoc: (javadoc java-object-or-class-here)
    Exit: Control+D or (exit) or (quit)
 Results: Stored in vars *1, *2, *3, an exception in *e

```
Next, `require` the generated `com.example.addressbook.Greeter.client` namespace, we will see output similar to the below:

```
user=> (require '[com.example.addressbook.Greeter.client :as greeter])
nil

```

We can now see that one of the var's refer'd into our REPL is `greeter/Hello`:
```
user=> greeter/He<Tab for auto complete options>
Hello
```

In order to invoke the client call, we'll need to create a client. We do this by requiring the protojure-lib ns below:
```
user=> (require '[protojure.grpc.client.providers.http2 :as grpc.http2])
nil
```
And creating a client connection:
```
user=> (def client @(grpc.http2/connect {:uri "http://localhost:8080"}))
#'user/client
```
Note: Many calls in the SDK return a [promise](https://clojuredocs.org/clojure.core/promise) and we therefore
[deref](https://clojure.github.io/clojure/clojure.core-api.html#clojure.core/deref) the calls to make them synchronous
for illustration purposes.

Now we can use our `Hello` function from above, and with the protoc-plugin example `hello` running we will receive
a HelloResponse message (you can see this message defined in the `greeter.proto` content above):
```
user=> @(greeter/Hello client {:name "Janet Johnathan Doe"})
#com.example.addressbook.HelloResponse{:message "Hello, Janet Johnathan Doe"}

```

If we go back to the source code of the running server (the output of `lein new protojure demo-server` above) and apply the below patch (remove the lines marked
  with `-` and add the lines marked with `+`):

```
diff --git a/src/demo_server/service.clj b/src/demo_server/service.clj
index 51c63f0..b480bec 100644
--- a/src/demo_server/service.clj
+++ b/src/demo_server/service.clj
@@ -8,7 +8,9 @@
             [protojure.pedestal.core :as protojure.pedestal]
             [protojure.pedestal.routes :as proutes]
             [com.example.addressbook.Greeter.server :as greeter]
-            [com.example.addressbook :as addressbook]))
+            [com.example.addressbook :as addressbook]
+            [io.pedestal.log :as log]))

 (defn about-page
   [request]
@@ -40,6 +42,7 @@
   greeter/Service
   (Hello
     [this {{:keys [name]} :grpc-params :as request}]
+    (log/info "Processing com.example.addressbook.Greeter/Hello invocation with request: " name)
     {:status 200
      :body {:message (str "Hello, " name)}}))


```

Stop the running demo-server process and restart with `lein run`.

From your client repl, you can now re-run:

```
user=> (def client @(grpc.http2/connect {:uri "http://localhost:8080"}))
#'user/client
user=> @(greeter/Hello client {:name "Janet Johnathan Doe"})
#com.example.addressbook.HelloResponse{:message "Hello, Janet Johnathan Doe"}
```
After invoking the client call against the `demo-server` above, viewing the logs
of the `lein run` demo-server will show:
```
20-07-08 12:39:18 mrkiouak INFO [demo-server.service:116] - {"Processing com.example.addressbook.Greeter/Hello invocation with request: " "Janet Johnathan Doe", :line 44}

```

Congratulations, you've invoked a remote procedure call round trip with the GRPC protocol using Clojure on both ends.
You may now interoperate with a client or server written in any other language that adheres to the GRPC and .proto spec.

#### Further examples

##### Client Connect Example

```
@(grpc.http2/connect {:uri (str "http://localhost:" port) :content-coding "gzip"})
```

##### Unary Example
[Protocol Buffer Definition](https://github.com/protojure/protoc-plugin/blob/master/examples/hello/resources/addressbook.proto)
```
syntax = "proto3";
package com.example.addressbook;

message Person {
    string name = 1;
    int32 id = 2;  // Unique ID number for this person.
    string email = 3;

    enum PhoneType {
        MOBILE = 0;
        HOME = 1;
        WORK = 2;
    }

    message PhoneNumber {
        string number = 1;
        PhoneType type = 2;
    }

    repeated PhoneNumber phones = 4;
}

```
```
message HelloResponse {
    string message = 1;
}

```
* gRPC Service Definition
```
service Greeter {
    rpc Hello (Person) returns (HelloResponse);
}
```

* Client
```
@(greeter/Hello @(grpc.http2/connect {:uri "http://localhost:8080"}) {:name "Janet Johnathan Doe"})
```
* Server Handler

```
(deftype Greeter []
  greeter/Service
  (Hello
    [this {{:keys [name]} :grpc-params :as request}]
    {:status 200
     :body {:message (str "Hello, " name)}}))
```

Include the below in the interceptors passed to
the pedestal routes key:

```
(proutes/->tablesyntax {:rpc-metadata greeter/rpc-metadata
                        :interceptors common-interceptors
                        :callback-context (Greeter.)})
```

Refer to [src/hello](https://github.com/protojure/protoc-plugin/tree/master/examples/hello/src/hello) in the hello example
in the `examples/` dir of protoc-plugin [here](https://github.com/protojure/protoc-plugin/tree/master/examples/hello)

You can find an additional unary client and server example (a runnable one) in the boilerplate generated by 'lein new protojure'

##### Server Streaming Example

When a client sends a request to the server, two [channels](https://clojuredocs.org/clojure.core.async/chan) are provided in the request: `:grpc-out` and `close-ch`.

* `:grpc-out` channel

Is the streaming channel, used to send all the messages. The handler first acknowledges streaming will start by returning the same grpc-out channel as the :body of the response map (instead of a map as above in the unary example).

When the server is done with the streaming, simply close! the channel:

```
(deftype Greeter []
  greeter/Service
  (SayRepeatHello
    [this {{:keys [name]} :grpc-params :as request}]
    (let [resp-chan (:grpc-out request)]
      (go
        (dotimes [_ 3]
          (>! resp-chan {:message (str "Hello, " name)}))
        (async/close! resp-chan))
      {:status 200
       :body resp-chan})))
```

* `close-ch` channel

Sometimes the client disconnects before expected. The server gets notified of such events via this channel. When this happens, server needs to handle it accordingly:

```
(defn handle-client-disconnect [close-chan]
  (async/take! close-chan
               (fn [signal]
                 (log/info "do stuff to handle client disconnection"))))

(deftype Greeter []
  greeter/Service
  (SayRepeatHello
    [this {{:keys [name]} :grpc-params :as request}]
    (let [close-chan (:close-ch request)
          resp-chan (:grpc-out request)]
      (handle-client-disconnect close-chan)
      (go
        (dotimes [_ 3]
          (>! resp-chan {:message (str "Hello, " name)}))
        (async/close! resp-chan))
      {:status 200
       :body resp-chan})))

```

* Error handling

Maybe the server needs to return an error to the client for any reason. This can be accomplished by using the [grpc-statuses] (https://github.com/protojure/lib/blob/master/src/protojure/grpc/status.clj):


```
(defn valid? [name]
  ;do validation
  )

(deftype Greeter []
  greeter/Service
  (SayRepeatHello
    [this {{:keys [name]} :grpc-params :as request}]
    (let [resp-chan (:grpc-out request)]
      (when-not (valid? name)
        (grpc.status/error :invalid-argument "Invalid parameter."))
      (go
        (dotimes [_ 3]
          (>! resp-chan {:message (str "Hello, " name)}))
        (async/close! resp-chan))
      {:status 200
       :body resp-chan})))

```

The error path (when "name" is not valid) will throw a `java.util.concurrent.ExecutionException` exception, that needs to be handled properly in the client side, while trying to [deref] (https://clojuredocs.org/clojure.core/deref) the promise that was returned on the request:


```

(try
  @(greeter/SayRepeatHello client {:name "Invalid name"} (async/chan 1)
  (catch Exception e
    (log/warn (format "promise compĺeted with error: %s" (:message (ex-data (.getCause e)))))))
```


##### Client Streaming Example
Identical to the above Client example for unary -- instead of closing the channels after pushing a single map,
keep the core.async channel open and push maps as needed.

See the streaming-grpc-check test in Protojure lib's [grpc_test.clj](https://github.com/protojure/lib/blob/master/test/protojure/grpc_test.clj)

Excerpt:

```
    (let [repetitions 50
          input (async/chan repetitions)
          output (async/chan repetitions)
          client (:grpc-client @test-env)
          desc {:service "example.hello.Greeter"
                :method "SayHelloOnDemand"
                :input {:f new-HelloRequest :ch input}
                :output {:f pb->HelloReply :ch output}}]

      (async/onto-chan input (repeat repetitions {:name "World"}))

      @(-> (grpc/invoke client desc)
```
