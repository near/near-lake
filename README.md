# near-lake-indexer

NEAR Lake is an indexer built on top of [NEAR Indexer microframework](https://github.com/nearprotocol/nearcore/tree/master/chain/indexer)
to watch the network and store all the events as JSON files on AWS S3.

## Concept

We used to have [NEAR Indexer for Explorer](https://github.com/near/near-indexer-for-explorer) that was watching for
the network and stored all the events to PostgreSQL database. PostgreSQL became the main bottleneck for us. After some
brainstorming sessions and researches we decided to go with AWS Aurora database.

Knowing the fact that [NEAR Explorer](https://explorer.near.org) is not the only project that uses the Indexer for Explorer's
database, we wanted to come up with the concept that will allow us to cover even more projects that can benefit from the data
from NEAR Protocol.

That's why we decided to store the data from the blockchain as JSON files on AWS S3 bucket that can be used
as a data source for different projects.

As "Indexer for Explorer Remake" project we are going to have `near-lake` as a data writer. There's going to be
another project that will read from AWS S3 bucket and will store all the data in SingleStore database. This
will replace NEAR Indexer for Explorer PostgreSQL database at some moment and will become the main
source for NEAR Explorer.

## How to start

The final setup consists of the following components:
* AWS S3 Bucket as a storage
* NEAR Lake binary that operates as a regular NEAR Protocol peer-to-peer node, so you will operate it as
  any other [Regular/RPC Node in NEAR](https://docs.near.org/docs/develop/node/rpc/hardware-rpc)

### Prepare Development Environment

Before you proceed, make sure you have the following software installed:
* [Rust compiler](https://rustup.rs/) of the version that is mentioned in `rust-toolchain` file in the root of
  [nearcore](https://github.com/nearprotocol/nearcore) project.
* Ensure you have [AWS Credentials configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)
    From AWS Docs:

  > For example, the files generated by the AWS CLI for a default profile configured with aws configure looks similar to the following.
  >
  > ~/.aws/credentials
  > ```
  > [default]
  > aws_access_key_id=AKIAIOSFODNN7EXAMPLE
  > aws_secret_access_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
  > ```

### Compile NEAR Lake

```bash
$ cargo build --release
```

### Configure NEAR Lake

To connect NEAR Lake to the specific chain you need to have necessary configs, you can generate it as follows:

```bash
$ ./target/release/near-lake --home ~/.near/testnet init --chain-id testnet --download-config --download-genesis
```

The above code will download the official genesis config and generate necessary configs. You can replace `testnet` in the command above to different network ID (`betanet`, `mainnet`).

**NB!** According to changes in `nearcore` config generation we don't fill all the necessary fields in the config file.
While this issue is open https://github.com/nearprotocol/nearcore/issues/3156 you need to download config you want and replace the generated one manually.
- [testnet config.json](https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/testnet/config.json)
- [betanet config.json](https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/betanet/config.json)
- [mainnet config.json](https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/mainnet/config.json)

Configs for the specified network are in the `--home` provided folder. We need to ensure that NEAR Lake follows
all the necessary shards, so `"tracked_shards"` parameters in `~/.near/testnet/config.json` needs to be configured properly.
Currently, `nearcore` treats empty value for `"tracked_shards"` as "do not track any shard" and **any value** as "track all shards".
For example, in order to track all shards, you just add the shard #0 to the list:

```
...
"tracked_shards": [0],
...
```


### (Optional) Configuration via environment variables

You can also configure NEAR Lake via environment variables. This is useful if you want to run NEAR Lake in a Docker container.

Add `.env` file to the root of the project with the following content:

```
BUCKET=near-lake-custom # name of the bucket to store data in (e.g. near-lake-data-testnet or near-lake-data-mainnet)
REGION=eu-central-1
AWS_DEFAULT_REGION=eu-central-1
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE # either via env or via ~/.aws
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY # either via env or via ~/.aws
ENDPOINT=http://localhost:4566 # for the custom S3 storage endpoing (e.g. Minio or Localstack)
```

Or you can pass them via command line (see below)

### Run NEAR Lake

Commands to run NEAR Lake, after `./target/release/near-lake`

| Command 	| Key/Subcommand               	| Required/Default                                                 	| Responsible for                                                                                                                                                                                                                                                                                                                                                         	|
|---------	|--------------------------	|------------------------------------------------------------------	|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------	|
|         	| `--home`                 	| Default <br>`~/.near`                                            	| Tells the node where too look for necessary files: <br>`config.json`<br>, <br>`genesis.json`<br>, <br>`node_key.json`<br>, and <br>`data`<br> folder                                                                                                                                                                                                                    	|
| `init`  	|                              	|                                                                  	| Tells the node to generate config files in `--home-dir`                                                                                                                                                                                                                                                                                                                 	|
|         	| `--chain-id`                 	| Required<br><br>  * `localnet`<br>  * `testnet`<br>  * `mainnet` 	| Defines the chain to generate config files for                                                                                                                                                                                                                                                                                                                          	|
|         	| `--download-config`          	| Optional                                                         	| If provided tells the node to download `config.json` from the public URL. You can download them manually<br><br> - [testnet config.json](https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/testnet/config.json)<br> - [mainnet config.json](https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/mainnet/config.json)      	|
|         	| `--download-genesis`         	| Optional                                                         	| If provided tells the node to download `genesis.json` from the public URL. You can download them manually<br><br> - [testnet genesis.json](https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/testnet/genesis.json)<br> - [mainnet genesis.json](https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/mainnet/genesis.json) 	|
|         	| TODO:<br>Other `neard` keys  	|                                                                  	|                                                                                                                                                                                                                                                                                                                                                                         	|
| `run`   	|                              	|                                                                  	| Runs the node                                                                                                                                                                                                                                                                                                                                                           	|
|         	| `--bucket`                   	| Required                                                         	| AWS S3 Bucket name                                                                                                                                                                                                                                                                                                                                                      	|
|         	| `--region`                   	| Required                                                         	| AWS S3 Bucket region                                                                                                                                                                                                                                                                                                                                                    	|
|           | `--fallback-region`           | Default eu-central-1                                              | AWS S3 Fallback region                                                                                                                                                                                                                                                                                                                                                 	|
|           | `--endpoint`                  | Optional                                                          | AWS S3 compatible API endpoint                                                                                                                                                                                                                                                                                                                                            |
|         	| `--stream-while-syncing`     	| Optional                                                         	| If provided Indexer streams blocks while they appear on the node instead of waiting the node to be fully synced                                                                                                                                                                                                                                                         	|
|         	| `--concurrency`              	| Default 1                                                        	| Defines the concurrency for the process of saving block data to AWS S3                                                                                                                                                                                                                                                                                                  	|
|         	| `sync-from-latest`           	| One of the `sync-` subcommands is required                       	| Tells the node to start indexing from the latest block in the network                                                                                                                                                                                                                                                                                                   	|
|         	| `sync-from-interruption`     	| One of the `sync-` subcommands is required                       	| Tells the node to start indexing from the block the node was interrupted on (if it is a first start it will fallback to `sync-from-latest`)                                                                                                                                                                                                                             	|
|         	| `sync-from-block --height N` 	| One of the <br>`sync-`<br> subcommands is required               	| Tells the node to start indexing from the specified block height `N` (**Ensure** you node data has the block you want to start from)                                                                                                                                                                                                                                    	|

```bash
$ ./target/release/near-lake --home ~/.near/testnet run --stream-while-syncing --concurrency 50 sync-from-latest
```

After the network is synced, you should see logs of every block height currently received by NEAR Lake.


## Syncing

Whenever you run NEAR Lake for any network except localnet you'll need to sync with the network.
This is required because it's a natural behavior of `nearcore` node and NEAR Lake is a wrapper
for the regular `nearcore` node. In order to work and index the data your node must be synced
with the network. This process can take a while, so we suggest to download a fresh backup of
the `data` folder and put it in you `--home-dir` of your choice (by default it is `~/.near`)

Running your NEAR Lake node on top of a backup data will reduce the time of syncing process
because your node will download only the data after the backup was cut and it takes reasonable amount time.

All the backups can be downloaded from the public S3 bucket which contains latest daily snapshots:

You will need [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html) to be installed in order to download the backups.

### Mainnet

```
$ aws s3 --no-sign-request cp s3://near-protocol-public/backups/mainnet/rpc/latest .
$ LATEST=$(cat latest)
$ aws s3 --no-sign-request cp --no-sign-request --recursive s3://near-protocol-public/backups/mainnet/rpc/$LATEST ~/.near/data
```

### Testnet

```
$ aws s3 --no-sign-request cp s3://near-protocol-public/backups/testnet/rpc/latest .
$ LATEST=$(cat latest)
$ aws s3 --no-sign-request cp --no-sign-request --recursive s3://near-protocol-public/backups/testnet/rpc/$LATEST ~/.near/data
```



## Running NEAR Lake as an archival node

It's not necessary but in order to index everything in the network it is better to do it from the genesis.
`nearcore` node is running in non-archival mode by default. That means that the node keeps data only
for [5 last epochs](https://docs.near.org/concepts/basics/epoch). In order to index data from the genesis
we need to turn the node in archival mode.

To do it we need to update `config.json` located in `--home-dir` (by default it is `~/.near`).

Find next keys in the config and update them as following:

```json
{
  ...
  "archive": true,
  "tracked_shards": [0],
  ...
}
```

The syncing process in archival mode can take a lot of time, so it's better to download a backup provided by NEAR
and put it in your `data` folder. After that your node will download only the data after the backup was cut and it
takes reasonable amount time.


All the backups can be downloaded from the public S3 bucket which contains the latest daily snapshots:

* [Archival Mainnet data folder](https://near-protocol-public.s3-accelerate.amazonaws.com/backups/mainnet/archive/data.tar)
* [Archival Testnet data folder](https://near-protocol-public.s3-accelerate.amazonaws.com/backups/testnet/archive/data.tar)

See [this link](https://docs.near.org/integrator/exchange-integration#running-an-archival-node) for reference

## Using the data

We write all the data to AWS S3 buckets:

- `near-lake-data-testnet` (`eu-central-1` region) for testnet
- `near-lake-data-mainnet` (`eu-central-1` region) for mainnet

## Custom S3 storage

In case you want to run you own near-lake instance and store data in some S3 compatible storage ([Minio](https://min.io/) or [Localstack](https://localstack.cloud/) as example)
You can owerride default S3 API endpoint by using `--endpoint` option

- run minio

```bash
$ mkdir -p /data/near-lake-custom && minio server /data
```

- run near-lake

```bash
$ ./target/release/near-lake --home ~/.near/testnet run --endpoint http://127.0.0.1:9000 --bucket near-lake-custom sync-from-latest
```

### Data structure

The data structure we use is the following:

```
<block_height>/
  block.json
  shard_0.json
  shard_1.json
  ...
  shard_N.json
```

- `<block_height>` is a 12-character-long `u64` string with leading zeros (e.g `000042839521`). [See this issue for a reasoning](https://github.com/near/near-lake/issues/23)
- `block_json` contains JSON-serialized [`BlockView`](https://github.com/near/nearcore/blob/e9a28c46c2bea505b817abf484e6015a61ea7d01/core/primitives/src/views.rs#L711-L716) struct. **NB!** this struct might change in the future, we will announce it
- `shard_N.json` where `N` is `u64` starting from `0`. Represents the index number of the shard. In order to find out the expected number of shards in the block you can look in `block.json` at `.header.chunks_included`

### Access the data

All NEAR Lake AWS S3 buckets have [Request Payer](https://docs.aws.amazon.com/AmazonS3/latest/userguide/RequesterPaysBuckets.html) enabled. It means that anyone with their own AWS credentials can List and Read the bucket's content and **be charged for it by AWS**. Connections to the bucket have to be done with AWS credentials provided. See [NEAR Lake Framework](https://github.com/near/near-lake-framework) for a reference.

### NEAR Lake Framework

Once we [set up the public access to the buckets](https://github.com/near/near-lake/issues/22) anyone will be able to build their own code to read it through.

For our own needs we are working on [NEAR Lake Framework](https://github.com/near/near-lake-framework) to have a simple way to create an indexer on top of the data stored by NEAR Lake itself.

**See the [official announce of NEAR Lake Framework on the NEAR Gov Forum](https://gov.near.org/t/announcement-near-lake-framework-brand-new-word-in-indexer-building-approach/17668)**

### Common Errors & Solutions

#### If you don't have any peer

Here is a script that asks the RPC node about its peers and makes a list of boot nodes for you:

```--boot-nodes `curl -X POST https://rpc.mainnet.near.org \
  -H "Content-Type: application/json" \
  -d '{
        "jsonrpc": "2.0",
        "method": "network_info",
        "params": [],
        "id": "dontcare"
      }' | \
jq '.result.active_peers as $list1 | .result.known_producers as $list2 |
$list1[] as $active_peer | $list2[] |
select(.peer_id == $active_peer.id) |
"\(.peer_id)@\($active_peer.addr)"' |\
awk 'NR>2 {print ","} length($0) {print p} {p=$0}' ORS="" | sed 's/"//g'````

