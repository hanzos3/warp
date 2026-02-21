# Hanzo S3 Warp (Benchmarking Tool)

High-performance S3 benchmarking tool for testing object storage systems.

- **Repo**: [github.com/hanzos3/warp](https://github.com/hanzos3/warp)
- **Server**: [github.com/hanzoai/s3](https://github.com/hanzoai/s3)
- **Domain**: [s3.hanzo.ai](https://s3.hanzo.ai) / [hanzo.space](https://hanzo.space)

## Download

### From binary

[Download Binary Releases](https://github.com/hanzos3/warp/releases) for various platforms.

### Build from source

Warp requires minimum Go `go1.21`, please ensure you have compatible version for this build.

```
git clone https://github.com/hanzos3/warp.git
cd warp && go build
./warp [options]
```

## Configuration

Warp can be configured either using commandline parameters or environment variables.
The S3 server to use can be specified on the commandline using `--host`, `--access-key`,
`--secret-key` and optionally `--tls` and `--region` to specify TLS and a custom region.

It is also possible to set the same parameters using the `WARP_HOST`, `WARP_ACCESS_KEY`,
`WARP_SECRET_KEY`, `WARP_REGION` and `WARP_TLS` environment variables.

The credentials must be able to create, delete and list buckets and upload files and perform the operation requested.

By default operations are performed on a bucket called `warp-benchmark-bucket`.
This can be changed using the `--bucket` parameter.

> [!WARNING]
> Note the bucket will be *completely wiped* before and after each run, so it should **not** contain any data.

If you are running TLS, you can enable
[server-side-encryption](https://docs.aws.amazon.com/AmazonS3/latest/dev/ServerSideEncryptionCustomerKeys.html)
of objects using `--encrypt`. A random key will be generated and used for objects.
To use [SSE-S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingServerSideEncryption.html) encryption use the `--sse-s3-encrypt` flag.

If your server is incompatible with [AWS v4 signatures](https://docs.aws.amazon.com/AmazonS3/latest/API/sig-v4-authenticating-requests.html) the older v2 signatures can be used with `--signature=S3V2`.

## Usage

`warp command [options]`

Example running a mixed type benchmark against 8 servers named `s3-server-1` to `s3-server-8`
on port 9000 with the provided keys:

`warp mixed --host=s3-server{1...8}:9000 --access-key=ACCESS_KEY --secret-key=SECRET_KEY --autoterm`

This will run the benchmark for up to 5 minutes and print the results.

### YAML configuration

As an alternative configuration option you can use an on-disk YAML configuration file.

See [yml-samples](https://github.com/hanzos3/warp/tree/master/yml-samples) for a collection of
configuration files for each benchmark type.

To run a benchmark use `warp run <file.yml>`.

Values can be injected from the commandline using one or multiple `-var VarName=Value`.
These values can be referenced inside YAML files with `{{.VarName}}`.
Go [text templates](https://pkg.go.dev/text/template) are used for this.

## Benchmarks

All benchmarks operate concurrently. By default, 20 operations will run concurrently.
This can however also be tweaked using the `--concurrent` parameter.

Tweaking concurrency can have an impact on performance, especially if latency to the server is tested.
Most benchmarks will also use different prefixes for each "thread" running.

By default all benchmarks save all request details to a file named `warp-operation-yyyy-mm-dd[hhmmss]-xxxx.csv.zst`.
A custom file name can be specified using the `--benchdata` parameter.
The raw data is [zstandard](https://facebook.github.io/zstd/) compressed CSV data.

### Multiple Hosts

Multiple S3 hosts can be specified as comma-separated values, for instance
`--host=10.0.0.1:9000,10.0.0.2:9000` will switch between the specified servers.

Alternatively numerical ranges can be specified using `--host=10.0.0.{1...10}:9000` which will add
`10.0.0.1` through `10.0.0.10`. This syntax can be used for any part of the host name and port.

A file with newline separated hosts can also be specified using `file:` prefix and a file name.
For distributed tests the file will be read locally and sent to each client.

By default a host is chosen between the hosts that have the least number of requests running
and with the longest time since the last request finished. This will ensure that in cases where
hosts operate at different speeds that the fastest servers will get the most requests.
It is possible to choose a simple round-robin algorithm by using the `--host-select=roundrobin` parameter.
If there is only one host this parameter has no effect.

When benchmarks are done per host averages will be printed out.
For further details, the `--analyze.v` parameter can also be used.

## Distributed Benchmarking

It is possible to coordinate several warp instances automatically.
This can be useful for testing performance of a cluster from several clients at once.

For reliable benchmarks, clients should have synchronized clocks.
Warp checks whether clocks are within one second of the server,
but ideally, clocks should be synchronized with [NTP](http://www.ntp.org/) or a similar service.

To use Kubernetes see [Running warp on kubernetes](https://github.com/hanzos3/warp/blob/master/k8s/README.md).

### Client Setup

WARNING: Never run warp clients on a publicly exposed port. Clients have the potential to DDOS any service.

Clients are started with

```
warp client [listenaddress:port]
```

`warp client` Only accepts an optional host/ip to listen on, but otherwise no specific parameters.
By default warp will listen on `127.0.0.1:7761`.

Only one server can be connected at the time.
However, when a benchmark is done, the client can immediately run another one with different parameters.

There will be a version check to ensure that clients are compatible with the server,
but it is always recommended to keep warp versions the same.

### Server Setup

Any benchmark can be run in server mode.
When warp is invoked as a server no actual benchmarking will be done on the server.
Each client will execute the benchmark.

The server will coordinate the benchmark runs and make sure they are run correctly.

When the benchmark has finished, the combined benchmark info will be collected, merged and saved/displayed.
Each client will also save its own data locally.

Enabling server mode is done by adding `--warp-client=client-{1...10}:7761`
or a comma separated list of warp client hosts.
Finally, a file with newline separated hosts can also be specified using `file:` prefix and a file name.
If no host port is specified the default is added.

Example:

```
warp get --duration=3m --warp-client=client-{1...10} --host=s3-server-{1...16} --access-key=ACCESS_KEY --secret-key=SECRET_KEY
```

Note that parameters apply to *each* client.
So if `--concurrent=8` is specified each client will run with 8 concurrent operations.
If a warp server is unable to connect to a client the entire benchmark is aborted.

If the warp server looses connection to a client during a benchmark run an error will
be displayed and the server will attempt to reconnect.
If the server is unable to reconnect, the benchmark will continue with the remaining clients.

### Manually Distributed Benchmarking

While it is highly recommended to use the automatic distributed benchmarking warp can also
be run manually on several machines at once.

When running benchmarks on several clients, it is possible to synchronize
their start time using the `--syncstart` parameter.
The time format is 'hh:mm' where hours are specified in 24h format,
and parsed as local computer time.

Using this will make it more reliable to [merge benchmarks](https://github.com/hanzos3/warp#merging-benchmarks)
from the clients for total result.
This will combine the data as if it was run on the same client.
Only the time segments that was actually overlapping will be considered.

When running benchmarks on several clients it is likely a good idea to specify the `--noclear` parameter
so clients don't accidentally delete each others data on startup.

### Benchmark Data

By default warp uploads random data.

See the full documentation above for details on object sizes, automatic termination, and all benchmark types.

## Server Profiling

When running against an S3 server that supports profiling it is possible to enable profiling while the benchmark is running.

This is done by adding `--serverprof=type` parameter with the type of profile you would like.
This requires that the credentials allows admin access for the first host.

## License

GNU Affero General Public License v3.0 - see [LICENSE](LICENSE) for details.
