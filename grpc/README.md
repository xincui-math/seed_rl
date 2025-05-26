# TensorFlow gRPC

This directory contains a simple framework for creating servers and clients
with deep TensorFlow integration. It makes it possible to register a
`@tf.function` on a server which is called without interacting with Python
and thus avoids the global interpreter lock. Asynchronous streaming gRPC is used
which means the implementation can achieve up to a million QPS. Additionally, it
supports unix domain sockets that can be used for multi processing on a single
machine.

## Example

### Server

```python
server = grpc.Server([‘unix:/tmp/foo’, ‘localhost:8000’])

# This function is batched meaning it will be called once there are, in this
# case, 5 incoming calls.
@tf.function(input_signature=[tf.TensorSpec([5], tf.int32)])
def foo(x):
  return x + 1

server.bind(foo)

@tf.function(input_signature=[tf.TensorSpec([], tf.int32),
                              tf.TensorSpec([], tf.int32)])
def bar(x, y):
  return x + y

server.bind(bar)

server.start()
```

### Client

```python
client = grpc.Client(‘unix:/tmp/foo’)

# The following calls are TensorFlow operations which means they can be used
# eagerly with tensors or numpy arrays, but they can also be used in a
# `tf.function`.
client.foo(42)    # Returns 43
client.bar(1, 2)  # Returns 3
```

## Building the custom op

The gRPC ops are not part of the standard TensorFlow distribution. They must
be compiled as a [custom operation](https://www.tensorflow.org/guide/create_op).
The steps below summarize the process using TensorFlow's **custom-op** build
tools:

1. Clone the TensorFlow `custom-op` repository and run `./configure.sh` to
   download headers matching your installed TensorFlow version.

```bash
git clone https://github.com/tensorflow/custom-op.git
cd custom-op
./configure.sh
```

2. Add the gRPC and protobuf dependencies to the `WORKSPACE` file. The
   following block matches the configuration used in `docker/Dockerfile.grpc`:

```python
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "com_google_protobuf",
    strip_prefix = "protobuf-3.9.1",
    urls = [
        "https://github.com/protocolbuffers/protobuf/archive/v3.9.1.tar.gz",
    ],
)

http_archive(
    name = "com_github_grpc_grpc",
    urls = [
        "https://github.com/grpc/grpc/archive/ac1c5de1b36da4a1e3d72ca40b0e43f24266121a.tar.gz",
    ],
    strip_prefix = "grpc-ac1c5de1b36da4a1e3d72ca40b0e43f24266121a",
)

load("@com_github_grpc_grpc//bazel:grpc_deps.bzl", "grpc_deps")
grpc_deps()
load("@com_github_grpc_grpc//bazel:grpc_extra_deps.bzl", "grpc_extra_deps")
grpc_extra_deps()
```

3. Copy the contents of this `grpc/` directory into the cloned repository and
   build the shared library:

```bash
cp -r /path/to/seed_rl/grpc custom-op/
cd custom-op
bazel build grpc:ops/grpc.so grpc:service_py_proto \
  --incompatible_remove_legacy_whole_archive=0
```

4. Copy the resulting files back into `seed_rl`:

```bash
cp bazel-bin/grpc/ops/grpc.so /path/to/seed_rl/grpc/grpc_cc.so
cp bazel-bin/grpc/service_pb2.py /path/to/seed_rl/grpc/service_pb2.py
```

After copying `grpc_cc.so`, the Python wrapper will load it automatically via
`tf.load_op_library` when importing `seed_rl.grpc.python.ops`.
