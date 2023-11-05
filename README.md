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
question could be implemented in different languages and what difficulties or
uncertainties there are.
The focus is on scripting languages like Python, not so much on Go-like
single-binary-producing languages like C or Rust (which would be easier to
write providers in, as will be shown below).

## Plugin installation & discovery

### Installing plugins

Users of Terraform request that Terraform load certain providers by putting
[provider
requirements](https://developer.hashicorp.com/terraform/language/providers/requirements)
in their Terraform files.
During `terraform init`, Terraform attempts to download them from the official
registry, although private registries can be used instead. During provider
development, one should specify
[Development Overrides](https://developer.hashicorp.com/terraform/cli/config/config-file#development-overrides-for-provider-developers)
in one's Terraform CLI config instead.

*Non-Go difficulties:* 😏 *I can't see any, because even though Terraform's
docs warn that providers interact with the registry, this doesn't seem to be
the case. Maybe they were actually referring to the issues in the next section,
which relate to the **contents** of what gets downloaded from there.*

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

*Non-Go difficulties:* 😟 *Due to the single-executable model, Terraform's
registry has no concept of providers having dependencies of their own. This
presents a problem for providers written in scripting languages, which require
an interpreter and library dependencies to run. Our options are to either
a) require the user to install any such dependencies themselves before using
the provider, b) ship the interpreter and libraries with the provider, c) make
the provider perform the installation when it first starts, or d) have our own
miniature package manager, which users will need to install and execute before
using our providers. Personally, I favor (d) as the least-bad option, because
(a) is uncomfortable for the user, (b) is bloated and uncomfortable for users
with slow internet, and finally (c) is not only annoying for users who want to
be confident that they'll be able to work offline after installation, but it's
also difficult to communicate a download is happening during provider startup,
as will be clear from the next section.*

## Launching & communicating with plugins

Terraform plugins (= providers, for now[^plugin]) are launched by and
communicate with Terraform Core (= the command-line app) using [a
variant][rpcpluginhashi] of the [RPCPlugin "protocol"][rpcpluginspec].[^forum][^proto]
The reference implementation of this protocol is the [go-plugin][goplugin] Go
package. I don't know of any other implementations (hence the need for this
document).

The diagram below gives an overview over what happens when Terraform launches
and communicates with a plugin. The individual steps are explained in more
detail in this section.

![Sequence diagram](diagram.svg)

### Step 1: Launching plugin executable, passing client cert (handshake start)

Having found a plugin executable, Terraform launches it as a subprocess. The
first bit of communication (part of what the RPCPlugin calls the handshake)
takes place at this point, as Terraform places a temporary certificate (to be
used for TLS in step 3) inside the `PLUGIN_CLIENT_CERT` environment variable of
the subprocess's execution environment.

*Non-Go difficulties:* 😌 *None. Any language can retrieve environment
variables.*

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

*Non-Go difficulties:* 😃 *None, and in fact, being able to pass a message
using nothing but writing to stdout means that things like a check for whether
the dependencies necessary to launch our provider are installed can be written
as a simple shell script. Even the download and installation of these
dependencies could in theory happen here (option (c) above), but as has just
been shown, we wouldn't be able to communicate that this is happening to the
user without crashing the process.*

#### If everything is OK: Valid handshake response

If things look OK and the plugin can proceed, it should ensure its RPC server
is running (explained in the next step) and write a line in the following
format to its standard output:

```
1|6|tcp|127.0.0.1:1234|grpc|$SERVER_CERT
```

`1` is the handshake version (fixed), `6` is the latest protocol version as of
2023 (see [protobuf files][providerprotobuffiles]), `tcp` is self-explanatory,
`127.0.0.1:1234` is the address and port that the plugin's RPC server is
listening on (can be chosen freely) and `grpc` specifies the RPC protocol (a
legacy option here is NetRPC, but gRPC should be used for new plugins).

`$SERVER_CERT` should be replaced by a temporary certificate, ideally adhering
to RPCPlugin's certificate [requirements][rpcpluginservercertreqs] (although in
my testing most of them weren't actually required), in the [format described in
the RPCPlugin spec][rpcpluginservercert], and with the [necessary changes for
go-plugin compatibility][rpcpluginhashiserver] applied (i.e., no base64
padding, which can be achieved by adding dummy data to text fields). This
certificate will be used for TLS (see next step).

> 🛈 It should be pointed out that if the final `|$SERVER_CERT` is left out,
Terraform will merrily try to connect to the plugin's RPC server anyway,
initiating a TLS handshake as normal. I don't think this is meaningful as
Google's gRPC library doesn't even support receiving TLS requests if a server
certificate hasn't been provided[^grpcsec] (see next section).
If one could get around this, it would likely fail down the line, so the
certificate should be provided no matter what (this is also what the RPCPlugin
spec proscribes if a client cert is provided in the env var).

After printing the handshake response, the child process must keep running
until killed by Terraform, otherwise the latter will report and error and exit.

*Non-Go difficulties:* 🙂 *The format of the response is simple enough that it
could also be produced by a shell script, although there probably wouldn't be a
point in doing so, because at the time when the response is sent, the full
provider must be up and running anyway. Also, generating a certificate, while
possible using e.g. the `openssl` CLI tool, is probably a bit annoying to do in
a shell script. Certainly, the certificate and response will be very simple for
a proper scripting language like Python and the necessary cryptography
depdendencies to produce, so there shouldn't be a problem.*

### Step 3: gRPC communication

Terraform connects to the address provided above as a
[gRPC](https://grpc.github.io/grpc/) client.
The other end, managed by the plugin, should be a gRPC server implementing the
latest [`tfplugin*` protobuf
spec](https://github.com/hashicorp/terraform/tree/bdc38b6527ee9927cee67cc992e02cc199f3cae1/docs/plugin-protocol).

#### TLS

As already mentioned, Terraform will always attempt a gRPC connection secured
by TLS (hence the exchange of certificates before) and the gRPC server must be
explicitly instructed[^grpcsec][^grpccred] to handle this, as the default
behavior of Google's gRPC Python library is to reject such requests without
even writing anything to the server logs / output (but the issue can be made
visible by setting the `GRPC_TRACE=all GRPC_VERBOSITY=DEBUG` env vars[^grpclog]
on the server side and comparing the request bytes to a [TLS
handshake](https://tls12.xargs.org/#client-hello)).

*Non-Go difficulties:* 😌 *There shouldn't be any because Google's gRPC library
takes care of it.*

#### Health checking service

According to an older document[^oldnongo], the plugin's gRPC server must also provide
a [gRPC health checking](https://grpc.io/docs/guides/health-checking/) service
that Terraform can query.
This may or may not be true (other parts of that document are definitely
outdated, e.g. the lack of certificate in the handshake response), but it can't
hurt.

*Non-Go difficulties:* 🙂 *There shouldn't be any. For some reason their docs
don't link to the actual, official packages that implement this service; for
Python, it's
[grpcio-health-checking](https://pypi.org/project/grpcio-health-checking/)
and there exists a [usage
example](https://github.com/grpc/grpc/blob/ce75ec23a1a9c5239834b92da4ce0992d367a39c/examples/python/health_checking/greeter_server.py).*

#### gRPC / protobuf communication: Control / extra features

Not mentioned in the RPCPlugin spec but present in its reference implementation
[go-plugin][goplugin] are certain RPC calls that the client can use to control
the server and/or which provide some advanced features. These have their own
[protobuf specs in the go-plugin repo][goplugincontrolprotobufspec].

As far as I can tell, these are optional and only serve to allow for nice to
have but not required features like graceful shutdown (explained in more detail
further below) or stdio forwarding.

*Non-Go difficulties:* 🙂 *None that I can see.*

#### gRPC / protobuf communication: Provider data

Finally we are ready for gRPC communication pertaining to the provider's actual
functions, which follows the format of the [(provider) protobuf
specs][protobuffiles] already mentioned above.
Nothing special here...

*Non-Go difficulties:* 😸 *None, but wait for it.*

#### Special protobuf fields

... sike! The protobuf specs contain fields of type
[`DynamicValue`](https://github.com/hashicorp/terraform/blob/bdc38b6527ee9927cee67cc992e02cc199f3cae1/docs/plugin-protocol/tfplugin6.4.proto#L27-L32),
which are central to the provider's functioning and represent JSON or
msgpack-encoded data that must adhere to a certain format but isn't described
in the protobuf files.[^forum]
There is [something that looks like a spec for
these](https://github.com/hashicorp/terraform/blob/bdc38b6527ee9927cee67cc992e02cc199f3cae1/docs/plugin-protocol/object-wire-format.md)
in Terraform's main repo.
Other than that, the implementation of the [marshalling/unmarshalling
methods](https://pkg.go.dev/github.com/hashicorp/terraform-plugin-go@v0.19.0/tfprotov6#DynamicValue)
and further usage in Terraform's source code might be useful, too.

*Non-Go difficulties:* 😨 *This could be difficult or a lot of effort, although
I haven't looked into the exact `DynamicValue` format enough to see how.*

### Step 4: Higher-level stuff

This part I'm not so sure about, but I guess there are higher-level rules for
how a provider must behave that also don't really have a spec. Perhaps the
implementation and documentation for [Terraform's Plugin
Framework](https://developer.hashicorp.com/terraform/plugin/framework) will
have some/most of the info.

*Non-Go difficulties:* 😖 *This also seems like it could be difficult, although
it might be more amenable to trial-and-error debugging than the preceding
lower-level parts of the protocol.*

### Step 5: Termination

#### Graceful shutdown

The RPCPlugin reference implementation ([go-plugin][goplugin]) contains a
[graceful plugin shutdown mechanism][goplugingracefulshutdown] not mentioned
in the spec.
The client triggers this on the server by making a
[`plugin.ShutDown`][pluginshutdownprotobufspec] gRPC call, which is part of the
"control" protobuf spec mentioned further above.
If the server doesn't shut down within a short time period, the non-graceful
shutdown explained in the next section will be attempted instead.

*Non-Go difficulties:* 😉 *Shouldn't be any, but make sure to also implement
proper handling of the non-graceful shutdown because the timeout is fairly
short.*

#### RPCPlugin spec termination (`SIGKILL`)

The RPCPlugin spec specifies that clients stop servers by `SIGKILL`ing them.
This may only happen when no RPC calls are still running and means that, during
such times, the provider must not perform any operations that mess things up
when suddenly interrupted.

There is another consequence relevant for providers launched by e.g. a shell
script: `SIGKILL` kills a process but not its children, so in this case, it
would kill the shell script's process but not the actual provider process.
Experimentation has shown that Terraform then waits indefinitely for the
children to terminate, which never happens unless special arrangements are made
to handle this situation.
There are [multiple ways](https://stackoverflow.com/a/23587108) to let a child
process terminate when its parent gets killed, the easiest of which is probably
to let it retrieve its parent process ID periodically (`getppid()` on POSIX)
and terminate when it has changed.

*Non-Go difficulties:* 🙄 *Having to implement special handling for the
parent-`SIGKILL` issue is annoying and wouldn't be necessary if the client
implementation was more robust. But it's not that difficult: `getppid` should
be available in most languages, e.g. Python's standard library comes with
[`os.getppid`](https://docs.python.org/3/library/os.html#os.getppid).*

## Miscellaneous resources

TODO incorporate these into text / footnotes

- [Implementation of handshake response in go-plugin (server
  part)](https://github.com/hashicorp/go-plugin/blob/303d84fc850fc2ad18981220339702809f8be06a/server.go#L417-L423)
- [Implementation of handshake response parsing in go-plugin (client
  part)](https://github.com/hashicorp/go-plugin/blob/303d84fc850fc2ad18981220339702809f8be06a/client.go#L793)
- [Some other in-repo plugin docs mainly concerning gRPC and
  versioning](https://github.com/hashicorp/terraform/tree/bdc38b6527ee9927cee67cc992e02cc199f3cae1/docs/plugin-protocol)


[^plugin]: [Terraform docs: Plugin development
  overview](https://developer.hashicorp.com/terraform/plugin)
[^forum]: [HashiCorp employee's forum post about plugin protocol and
  issues](https://discuss.hashicorp.com/t/terraform-grpc-alternative-client-implementation/35825/2)
  (from the other direction though, as the question is about implementing a
  *client*)
[^rpcpluginhashi]: [RPCPlugin docs on relation to HashiCorp plugins based on
  go-plugin e.g. Terraform providers)][rpcpluginhashi]
[^proto]: "Protocol" is in quotes because unlike typical protocols, it
  specifies not just message formats and sequences but also the modalities of
  how plugins are to be launched. Moreover, it spans 3 different communication
  channels (env vars, stdin/stdout, and local socket connections) which are all
  necessary for its basic operation.
[^grpcsec]: [gRPC docs:
  `add_secure_port`](https://grpc.github.io/grpc/python/grpc.html#grpc.Server.add_secure_port)
[^grpccred]: [gRPC docs: Create Server
  Credentials](https://grpc.github.io/grpc/python/grpc.html#create-server-credentials)
[^grpclog]: [gRPC logging info in Chromium
  docs](https://chromium.googlesource.com/external/github.com/grpc/grpc/+/HEAD/examples/python/debug/)
[^oldnongo]: [Older document on writing non-Go
  plugins](https://github.com/hashicorp/go-plugin/blob/303d84fc850fc2ad18981220339702809f8be06a/docs/guide-plugin-write-non-go.md)

[rpcpluginspec]: https://github.com/rpcplugin/spec/blob/e73dcf973a3fc589cc8687bf1bee6765ef134270/rpcplugin-spec.md
[providerprotobuffiles]: https://github.com/hashicorp/terraform/tree/bdc38b6527ee9927cee67cc992e02cc199f3cae1/docs/plugin-protocol
[rpcpluginservercert]: https://github.com/rpcplugin/spec/blob/e73dcf973a3fc589cc8687bf1bee6765ef134270/rpcplugin-spec.md#temporary-server-certificate
[rpcpluginservercertreqs]: https://github.com/rpcplugin/spec/blob/e73dcf973a3fc589cc8687bf1bee6765ef134270/rpcplugin-spec.md#temporary-keys-and-certificates
[rpcpluginhashi]: https://github.com/rpcplugin/spec/blob/e73dcf973a3fc589cc8687bf1bee6765ef134270/rpcplugin-spec.md#hashicorp-go-plugin-compatibility
[rpcpluginhashiserver]: https://github.com/rpcplugin/spec/blob/e73dcf973a3fc589cc8687bf1bee6765ef134270/rpcplugin-spec.md#go-plugin-clients-with-rpcplugin-servers
[goplugin]: https://github.com/hashicorp/go-plugin
[goplugincontrolprotobufspec]: https://github.com/hashicorp/go-plugin/tree/017b758bf4d495212a55db3de61b2d95ab104e53/internal/plugin
[goplugingracefulshutdown]: https://github.com/hashicorp/go-plugin/blob/017b758bf4d495212a55db3de61b2d95ab104e53/client.go#L496-L515
[pluginshutdownprotobufspec]: https://github.com/hashicorp/go-plugin/blob/017b758bf4d495212a55db3de61b2d95ab104e53/internal/plugin/grpc_controller.proto#L13
