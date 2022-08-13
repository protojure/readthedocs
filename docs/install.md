## Prerequisites

* Protocol Buffer Compiler ([protoc](https://github.com/protocolbuffers/protobuf/releases))
* A [Clojure development environment](https://clojure.org/guides/getting_started)

## Installation

Download the latest [Release](https://github.com/protojure/protoc-plugin/releases) and make it executable in your path.

```
sudo curl -L https://github.com/protojure/protoc-plugin/releases/latest/download/protoc-gen-clojure --output /usr/local/bin/protoc-gen-clojure
sudo chmod +x /usr/local/bin/protoc-gen-clojure
```

### Usage with non-`protoc` clients
To use this plugin with other build systems (such as [buf](https://github.com/bufbuild/buf)), you'll need to modify the binary, as the binary is actually a shell wrapper around a `.jar` file. If you see an error that looks like this:

```console
Failure: plugin clojure: fork/exec /Users/me/.local/bin/protoc-gen-clojure: exec format error
```
It's because of this shell wrapper.

In order for `exec` calls to work properly (on a \*nix system), you'll need to prepend a hashbang to the shell block. An example of how to do this:

```sh
cd /path/to/containing/folder
mv protoc-gen-clojure protoc-gen-clojure-other
echo "#!/bin/sh" > protoc-gen-clojure
cat protoc-gen-clojure-other >> protoc-gen-clojure
chmod +x protoc-gen-clojure
rm protoc-gen-clojure-other
```

Note that your editor may reformat the binary in such a way that it doesn't work anymore; hence the above approach.
