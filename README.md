# How difficult is it to write a Terraform provider in a language other than Go?

Terraform's documentation on writing new providers
[states](https://developer.hashicorp.com/terraform/plugin/best-practices/provider-code):

>Go is currently the only programming language supported by HashiCorp for
building Terraform providers.
[...]
While it is possible to write a non-Go provider, thanks to Terraform's use of
the gRPC protocol, it is harder than it may appear at first glance. Multiple
packages, from encoders and decoders to Terraform's type system, would all need
to be reimplemented in that new language. The Terraform Plugin Framework would
also need to be reimplemented, which is not a trivial challenge. And the way
non-Go providers would interact with the Registry, terraform init, and other
pieces of the Terraform ecosystem is unclear.

This document is meant to elucidate in more detail what exactly these
challenges involve and how one could make progress.

To this end, what follows is a description of how Terraform discovers and
interacts with providers, accompanied by some notes on how the interfaces in
question could be implemented in different languages. The focus will be on
scripting languages like Python, not so much on Go-like single-binary-producing
languages like C or Rust (which would be easier to write providers in, as will
be shown below).

## Downloading plugins & finding their executables

### Downloading plugins

This is actually not part of RPCPlugin, but it's important for Terraform
anyway: Users of Terraform request that Terraform load certain providers by
putting [provider requirements](https://developer.hashicorp.com/terraform/language/providers/requirements)
in their Terraform files.
During `terraform init`, Terraform attempts to download them from the official
registry, although private registries can be used instead. During provider
development, one should specify
[Development Overrides](https://developer.hashicorp.com/terraform/cli/config/config-file#development-overrides-for-provider-developers)
in one's Terraform CLI config instead.

### Finding plugin executables

Normal plugins that are written in Go come in the shape of a single executable
inside a folder specifying the provider it belongs to and its version and
architecture. There are typically some other files like the license or a
changelog, and I think one could put any number of files in a plugin package,
although I'm not 100% sure.

At the beginning of commands like `terraform apply`, Terraform finds the
executable based only on the name, architecture and version as reflected in the
folder names and filename.

The executable can be an ELF file, a bash script, or anything else that's,
well, executable.

## Launching & communicating with plugins

Terraform plugins (= providers, for now [^1]) are launched by and communicate
with Terraform Core (= the command-line app) using a variant of the
[RPCPlugin "protocol"](https://github.com/rpcplugin/spec/blob/e73dcf973a3fc589cc8687bf1bee6765ef134270/rpcplugin-spec.md).[^2][^3][^4]
The reference implementation of this protocol is the
[go-plugin](https://github.com/hashicorp/go-plugin) Go package. I don't know of
any other implementations (hence the need for this document).

The diagram below gives an overview over what happens when Terraform launches
and communicates with a plugin. The individual steps are explained in more
detail in this section.

![Sequence diagram](diagram.svg)

### Step 1: Launching plugin executable, passing client cert (handshake start)

Having found a plugin executable, Terraform launches is as a subprocess. The
first bit of communication (part of what the RPCPlugin calls the handshake)
takes place at this point, as Terraform places a temporary certificate (to be
used for TLS in step 3) inside the `PLUGIN_CLIENT_CERT` environment variable of
the subprocess's execution environment.

### Step 2: Handshake response from plugin on stdout

At this stage, the launched plugin executable can communicate with Terraform
Core by writing to standard output, but Terraform Core will read at most 1 line
of this before either aborting or proceeding with the next step.

#### In case of irrecoverable error: Write message to the user

If something is wrong and the plugin has to abort anyway, it is possible to
pass a 1-line message explaining the situation to the user by writing something
that doesn't follow the message format described in the next section to
standard output. Terraform will print it to its CLI, writing that it's an
"unrecognized remote plugin message" (so this is a bit of a hack and not a
channel that was designed for this purpose).

Standard error is completely ignored and not passed through to the user, so the
above method is the only one I know of to communicate with the user at this
stage.

#### If everything is OK: Valid handshake response

If things look OK and the plugin can proceed, it should ensure its RPC server
is running (explained in the next step) and write a line in the following
format to its standard output:

```
1|6|tcp|127.0.0.1:1234|grpc|$SERVER_CERT
```

`1` is the handshake version (fixed), `6` is the latest protocol version as of
2023 (see [protobuf files](https://github.com/hashicorp/terraform/tree/bdc38b6527ee9927cee67cc992e02cc199f3cae1/docs/plugin-protocol)),
`tcp` is self-explanatory, `127.0.0.1:1234` is the address and port that the
plugin's RPC server is listening on (can be chosen freely) and `grpc` specifies
the RPC protocol (a legacy option here is NetRPC, but gRPC should be used for
new plugins).

`$SERVER_CERT` should be replaced by a temporary certificate in the format
described in the RPCPlugin spec, which will be used for TLS (see next step).

> 🛈 It should be pointed out that if the final `|$SERVER_CERT` is left out,
Terraform will merrily try to connect to the plugin's RPC server anyway,
initiating a TLS handshake as normal. I don't think this is meaningful as
Google's gRPC library doesn't even support receiving TLS requests if a server
certificate hasn't been provided (see next section).
If one could get around this, it would likely fail down the line, so the
certificate should be provided no matter what (this is also what the RPCPlugin
spec proscribes if a client cert is provided in the env var).

### Step 3: gRPC communication

Terraform connects to the address provided above as a
[gRPC](https://grpc.github.io/grpc/) client.
The other end, managed by the plugin, should be a gRPC server implementing the
latest [`tfplugin*` protobuf
spec](https://github.com/hashicorp/terraform/tree/bdc38b6527ee9927cee67cc992e02cc199f3cae1/docs/plugin-protocol).

#### TLS

As already mentioned, Terraform will always attempt a gRPC connection secured
by TLS (hence the exchange of certificates before) and the gRPC server must be
explicitly configured to handle this, as the default behavior of Google's gRPC
Python library is to reject such requests without even writing anything to the
server logs / output (but the issue can be made visible by setting the
`GRPC_TRACE=all GRPC_VERBOSITY=DEBUG` env vars on the server side).

#### Health checking service

According to an older document [^5], the plugin's gRPC server must also provide
a [gRPC health checking](https://grpc.io/docs/guides/health-checking/) service
that Terraform can query.
This may or may not be true (other parts of that document are definitely
outdated, e.g. the lack of certificate in the handshake response), but it can't
hurt.

#### gRPC / protobuf communication

Then finally we are ready for the actual gRPC communication, which follows the
format of the protobuf specs linked above. Nothing special here...

#### Special protobuf fields

... sike! The protobuf specs contain fields of type
[`DynamicValue`](https://github.com/hashicorp/terraform/blob/bdc38b6527ee9927cee67cc992e02cc199f3cae1/docs/plugin-protocol/tfplugin6.4.proto#L27-L32),
which are central to the provider's functioning and contain JSON or
msgpack-encoded data that must adhere to a certain format but isn't described
in the protobuf files. [^2]
There is [something that looks like a spec for
these](https://github.com/hashicorp/terraform/blob/bdc38b6527ee9927cee67cc992e02cc199f3cae1/docs/plugin-protocol/object-wire-format.md)
in Terraform's repo.
Other than that, the implementation of the [marshalling/unmarshalling
methods](https://pkg.go.dev/github.com/hashicorp/terraform-plugin-go@v0.19.0/tfprotov6#DynamicValue)
and further usage in Terraform's source code might be useful, too.

### Step 4: Higher-level stuff

This part I'm not so sure about, but I guess there are higher-level rules for
how a provider must behave that also don't really have a spec. Perhaps the
implementation and documentation for [Terraform's Plugin
Framework](https://developer.hashicorp.com/terraform/plugin/framework) will
have some/most of the info.

I'm not overly worried about this part as it should be *much* more amenable to
trial and error than the lower-level protocol parts.

## Miscellaneous resources

TODO incorporate these into text / footnotes

- [Implementation of handshake response in go-plugin (server
  part)](https://github.com/hashicorp/go-plugin/blob/303d84fc850fc2ad18981220339702809f8be06a/server.go#L417-L423)
- [Implementation of handshake response parsing in go-plugin (client
  part)](https://github.com/hashicorp/go-plugin/blob/303d84fc850fc2ad18981220339702809f8be06a/client.go#L793)
- [Some other in-repo plugin docs mainly concerning gRPC and
  versioning](https://github.com/hashicorp/terraform/tree/bdc38b6527ee9927cee67cc992e02cc199f3cae1/docs/plugin-protocol)
- [TLS handshake byte-by-byte
  structure](https://tls12.xargs.org/#client-hello), useful for debugging TLS
  issues in gRPC
- [gRPC logging
  info](https://chromium.googlesource.com/external/github.com/grpc/grpc/+/HEAD/examples/python/debug/)
- [gRPC server credential creation
  methods](https://grpc.github.io/grpc/python/grpc.html#create-server-credentials)
- [gRPC health checking Python package](https://pypi.org/project/grpcio-health-checking/)
  - [usage example](https://github.com/grpc/grpc/blob/ce75ec23a1a9c5239834b92da4ce0992d367a39c/examples/python/health_checking/greeter_server.py)


[^1]: https://developer.hashicorp.com/terraform/plugin
[^2]: https://discuss.hashicorp.com/t/terraform-grpc-alternative-client-implementation/35825/2
[^3]: https://github.com/rpcplugin/spec/blob/e73dcf973a3fc589cc8687bf1bee6765ef134270/rpcplugin-spec.md#hashicorp-go-plugin-compatibility
[^4]: "Protocol" is in quotes because unlike typical protocols, it specifies
  not just message formats and sequences but also the modalities of how plugins
  are to be launched. It also spans 3 different communication channels (env
  vars, stdin/stdout, and local socket connections) which are all necessary for
  its basic operation.
[^5]: https://github.com/hashicorp/go-plugin/blob/303d84fc850fc2ad18981220339702809f8be06a/docs/guide-plugin-write-non-go.md
