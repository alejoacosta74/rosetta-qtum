<p align="center">
  <a href="https://www.rosetta-api.org">
    <img width="90%" alt="Rosetta" src="https://www.rosetta-api.org/img/rosetta_header.png">
  </a>
</p>
<h3 align="center">
   Rosetta Qtum
</h3>
<p align="center">
  <a href="https://circleci.com/gh/coinbase/rosetta-bitcoin/tree/master"><img src="https://circleci.com/gh/coinbase/rosetta-bitcoin/tree/master.svg?style=shield" /></a>
  <a href="https://coveralls.io/github/coinbase/rosetta-bitcoin"><img src="https://coveralls.io/repos/github/coinbase/rosetta-bitcoin/badge.svg" /></a>
  <a href="https://goreportcard.com/report/github.com/qtumproject/rosetta-qtum"><img src="https://goreportcard.com/badge/github.com/qtumproject/rosetta-qtum" /></a>
  <a href="https://github.com/qtumproject/rosetta-qtum/blob/master/LICENSE.txt"><img src="https://img.shields.io/github/license/coinbase/rosetta-bitcoin.svg" /></a>
  <a href="https://pkg.go.dev/github.com/qtumproject/rosetta-qtum?tab=overview"><img src="https://img.shields.io/badge/go.dev-reference-007d9c?logo=go&logoColor=white&style=shield" /></a>
</p>

<p align="center"><b>
ROSETTA-QTUM IS CONSIDERED <a href="https://en.wikipedia.org/wiki/Software_release_life_cycle#Alpha">ALPHA SOFTWARE</a>.
USE AT YOUR OWN RISK! COINBASE ASSUMES NO RESPONSIBILITY NOR LIABILITY IF THERE IS A BUG IN THIS IMPLEMENTATION.
</b></p>

## Overview
`rosetta-qtum` provides a reference implementation of the Rosetta API for
Qtum in Golang. If you haven't heard of the Rosetta API, you can find more
information [here](https://rosetta-api.org).

## Features
* Rosetta API implementation (both Data API and Construction API)
* UTXO cache for all accounts (accessible using the Rosetta `/account/balance` API)
* Stateless, offline, curve-based transaction construction from any SegWit-Bech32 Address


## Usage
As specified in the [Rosetta API Principles](https://www.rosetta-api.org/docs/automated_deployment.html),
all Rosetta implementations must be deployable via Docker and support running via either an
[`online` or `offline` mode](https://www.rosetta-api.org/docs/node_deployment.html#multiple-modes).

**YOU MUST INSTALL DOCKER FOR THE FOLLOWING INSTRUCTIONS TO WORK. YOU CAN DOWNLOAD DOCKER [HERE](https://www.docker.com/get-started).**

### Install
Clone this repository and run the following command to create a Docker image called `rosetta-qtum:latest`.

```text
make build-local
```

### Run
Running the following commands will start a Docker container in
[detached mode](https://docs.docker.com/engine/reference/run/#detached--d) with
a data directory at `<working directory>/qtum-data` and the Rosetta API accessible
at port `8080`.

#### Mainnet:Online
```text
docker run -d --rm --ulimit "nofile=100000:100000" -v "$(pwd)/qtum-data:/data" -e "MODE=ONLINE" -e "NETWORK=MAINNET" -e "PORT=8080" -p 8080:8080 -p 8333:8333 rosetta-qtum:latest
```
_If you cloned the repository, you can run `make run-mainnet-online`._

#### Mainnet:Offline
```text
docker run -d --rm -e "MODE=OFFLINE" -e "NETWORK=MAINNET" -e "PORT=8081" -p 8081:8081 rosetta-qtum:latest
```
_If you cloned the repository, you can run `make run-mainnet-offline`._

#### Testnet:Online
```text
docker run -d --rm --ulimit "nofile=100000:100000" -v "$(pwd)/qtum-data:/data" -e "MODE=ONLINE" -e "NETWORK=TESTNET" -e "PORT=8080" -p 8080:8080 -p 18333:18333 rosetta-qtum:latest
```
_If you cloned the repository, you can run `make run-testnet-online`._

#### Testnet:Offline
```text
docker run -d --rm -e "MODE=OFFLINE" -e "NETWORK=TESTNET" -e "PORT=8081" -p 8081:8081 rosetta-qtum:latest
```
_If you cloned the repository, you can run `make run-testnet-offline`._

## System Requirements
`rosetta-qtum` has been tested on an GCP `T2d-standard-16` instance.
This instance type has 16 vCPU with 64 gb of RAM, 10gb of swap and 100gb of ssd storage.

### Network Settings
To increase the load `rosetta-qtum` can handle, it is recommended to tune your OS
settings to allow for more connections. On a linux-based OS, you can run the following
commands ([source](http://www.tweaked.io/guide/kernel)):

```text
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
sysctl -w net.ipv4.tcp_max_syn_backlog=10000
sysctl -w net.core.somaxconn=10000
sysctl -p (when done)
```
_We have not tested `rosetta-qtum` with `net.ipv4.tcp_tw_recycle` and do not recommend
enabling it._

You should also modify your open file settings to `100000`. This can be done on a linux-based OS
with the command: `ulimit -n 100000`.

### Memory-Mapped Files
`rosetta-qtum` uses [memory-mapped files](https://en.wikipedia.org/wiki/Memory-mapped_file) to
persist data in the `indexer`. As a result, you **must** run `rosetta-qtum` on a 64-bit
architecture (the virtual address space easily exceeds 100s of GBs).

If you receive a kernel OOM, you may need to increase the allocated size of swap space
on your OS. There is a great tutorial for how to do this on Linux [here](https://linuxize.com/post/create-a-linux-swap-file/).

## Architecture
`rosetta-qtum` uses the `syncer`, `storage`, `parser`, and `server` package
from [`rosetta-sdk-go`](https://github.com/coinbase/rosetta-sdk-go) instead
of a new Qtum-specific implementation of packages of similar functionality. Below
you can find a high-level overview of how everything fits together:
```text
                               +------------------------------------------------------------------+
                               |                                                                  |
                               |                 +--------------------------------------+         |
                               |                 |                                      |         |
                               |                 |                 indexer              |         |
                               |                 |                                      |         |
                               |                 | +--------+                           |         |
                               +-------------------+ pruner <----------+                |         |
                               |                 | +--------+          |                |         |
                         +-----v----+            |                     |                |         |
                         | qtumd |            |              +------+--------+       |         |
                         +-----+----+            |     +--------> block_storage <----+  |         |
                               |                 |     |        +---------------+    |  |         |
                               |                 | +---+----+                        |  |         |
                               +-------------------> syncer |                        |  |         |
                                                 | +---+----+                        |  |         |
                                                 |     |        +--------------+     |  |         |
                                                 |     +--------> coin_storage |     |  |         |
                                                 |              +------^-------+     |  |         |
                                                 |                     |             |  |         |
                                                 +--------------------------------------+         |
                                                                       |             |            |
+-------------------------------------------------------------------------------------------+     |
|                                                                      |             |      |     |
|         +------------------------------------------------------------+             |      |     |
|         |                                                                          |      |     |
|         |                     +---------------------+-----------------------+------+      |     |
|         |                     |                     |                       |             |     |
| +-------+---------+   +-------+---------+   +-------+-------+   +-----------+----------+  |     |
| | account_service |   | network_service |   | block_service |   | construction_service +--------+
| +-----------------+   +-----------------+   +---------------+   +----------------------+  |
|                                                                                           |
|                                         server                                            |
|                                                                                           |
+-------------------------------------------------------------------------------------------+
```

### Optimizations
* Automatically prune qtumd while indexing blocks
* Reduce sync time with concurrent block indexing
* Use [Zstandard compression](https://github.com/facebook/zstd) to reduce the size of data stored on disk without needing to write a manual byte-level encoding

#### Concurrent Block Syncing
To speed up indexing, `rosetta-qtum` uses concurrent block processing
with a "wait free" design (using channels instead of sleeps to signal
which threads are unblocked). This allows `rosetta-qtum` to fetch
multiple inputs from disk while it waits for inputs that appeared
in recently processed blocks to save to disk.
```text
                                                   +----------+
                                                   | qtumd |
                                                   +-----+----+
                                                         |
                                                         |
          +---------+ fetch block data / unpopulated txs |
          | block 1 <------------------------------------+
          +---------+                                    |
       +-->   tx 1  |                                    |
       |  +---------+                                    |
       |  |   tx 2  |                                    |
       |  +----+----+                                    |
       |       |                                         |
       |       |           +---------+                   |
       |       |           | block 2 <-------------------+
       |       |           +---------+                   |
       |       +----------->   tx 3  +--+                |
       |                   +---------+  |                |
       +------------------->   tx 4  |  |                |
       |                   +---------+  |                |
       |                                |                |
       | retrieve previously synced     |   +---------+  |
       | inputs needed for future       |   | block 3 <--+
       | blocks while waiting for       |   +---------+
       | populated blocks to save to    +--->   tx 5  |
       | disk                               +---------+
       +------------------------------------>   tx 6  |
       |                                    +---------+
       |
       |
+------+--------+
|  coin_storage |
+---------------+
```

## Testing with rosetta-cli
To validate `rosetta-qtum`, [install `rosetta-cli`](https://github.com/coinbase/rosetta-cli#install)
and run one of the following commands:
* `rosetta-cli check:data --configuration-file rosetta-cli-conf/testnet/config.json` - This command validates that the Data API information in the `testnet` network is correct. It also ensures that the implementation does not miss any balance-changing operations.
* `rosetta-cli check:construction --configuration-file rosetta-cli-conf/testnet/config.json` - This command validates the blockchain’s construction, signing, and broadcasting.
* `rosetta-cli check:data --configuration-file rosetta-cli-conf/mainnet/config.json` - This command validates that the Data API information in the `mainnet` network is correct. It also ensures that the implementation does not miss any balance-changing operations.

## Issues
Interested in helping fix issues in this repository? You can find to-dos in the [Issues](https://github.com/coinbase/rosetta-bitcoin/issues) section with the `help wanted` tag. Be sure to reach out on our [community](https://community.rosetta-api.org) before you tackle anything on this list.


## Development
* `make deps` to install dependencies
* `make test` to run tests
* `make lint` to lint the source code
* `make salus` to check for security concerns
* `make build-local` to build a Docker image from the local context
* `make coverage-local` to generate a coverage report

## License
This project is available open source under the terms of the [Apache 2.0 License](https://opensource.org/licenses/Apache-2.0).

© 2021 Coinbase