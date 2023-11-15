---
description: Run a mainnet validator and bridge operator on one machine using Docker.
title: Run a validator and bridge operator together
tags:
- docker-mainnet
---

## Overview

This guide demonstrates how to run a mainnet validator node and a bridge operator on one machine using Docker.

## Before you start

### Docker

* Install [Docker Engine](https://docs.docker.com/engine/install/)
* Install [Docker Compose plugin](https://docs.docker.com/compose/install/)
* Install [Zstandard](https://github.com/facebook/zstd)

### Ethereum endpoint

You need an Ethereum RPC endpoint to listen for events from Ethereum chain and send events to Ethereum. This can be an [Alchemy](https://www.alchemy.com/overviews/private-rpc-endpoint), Infura or any other Ethereum RPC endpoint.

### Private keys

Generate three private keys as described in [Generate keys](./../generate-keys.md):

* One key for the validator
* One key for the bridge voter
* One key for the bridge operator

### System requirements

Recommended system requirements for running a validator node on the mainnet:

* 8-core CPU
* 32 GB RAM
* 700 GB high-speed SSD
* x86-64 architecture

These requirements are rough guidelines, and each node operator
should monitor their node to ensure good performance for the intended task.
The size of your node will also grow over time.

## Run the node

1. Set up directories:

  Create a ronin directory:
  ```
  mkdir ~/ronin
  ```

  Go to the newly created directory:
  ```
  cd ~/ronin
  ```

  Create a directory for bridge data:
  ```
  mkdir -p data
  ```

  Create a directory for chain data:
  ```
  mkdir -p chaindata/data/ronin
  ```

2. Create a file called `docker-compose.yml`:

  ```
  vim docker-compose.yml
  ```

3. Paste the following into `docker-compose.yml`:

  ```
  version: "3"
  services:
    node:
      image: ${NODE_IMAGE}
      stop_grace_period: 5m
      stop_signal: SIGINT
      hostname: node
      container_name: node
      ports:
        - 127.0.0.1:8545:8545
        - 127.0.0.1:8546:8546
        - 30303:30303
        - 30303:30303/udp
        - 6060:6060
      volumes:
        - ~/ronin/chaindata:/ronin
      environment:
        - SYNC_MODE=full
        - PASSWORD=${PASSWORD}
        - PRIVATE_KEY=${VALIDATOR_PRIVATE_KEY}
        - BOOTNODES=${BOOTNODES}
        - NETWORK_ID=${NETWORK_ID}
        - RONIN_PARAMS=${RONIN_PARAMS}
        - VERBOSITY=${VERBOSITY}
        - MINE=${MINE}
        - GASPRICE=${GASPRICE}
        - ETHSTATS_ENDPOINT=${INSTANCE_NAME}:${CHAIN_STATS_WS_SECRET}@${CHAIN_STATS_WS_SERVER}:443
        - BLS_PASSWORD=${BLS_PASSWORD}
        - BLS_PRIVATE_KEY=${BLS_PRIVATE_KEY}
        - BLS_AUTO_GENERATE=${BLS_AUTO_GENERATE}
        - ENABLE_FAST_FINALITY=${ENABLE_FAST_FINALITY}
        - ENABLE_FAST_FINALITY_SIGN=${ENABLE_FAST_FINALITY_SIGN}
        - BLS_SHOW_PRIVATE_KEY=${BLS_SHOW_PRIVATE_KEY}
    db:
      image: postgres:14.3
      restart: always
      command: postgres -c 'max_connections=1000'
      hostname: db
      container_name: db
      ports:
        - 127.0.0.1:5432:5432
      environment:
        POSTGRES_PASSWORD: ${DB_PASSWORD}
      volumes:
        - ~/ronin/data:/var/lib/postgresql/data
    bridge:
      image: ${BRIDGE_IMAGE}
      restart: always
      container_name: bridge
      environment:
        - RONIN_RPC=${RPC_ENDPOINT}
        - RONIN_BRIDGE_VOTER_KEY=${BRIDGE_VOTER_PRIVATE_KEY}
        - RONIN_BRIDGE_OPERATOR_KEY=${BRIDGE_OPERATOR_PRIVATE_KEY}
        - ETHEREUM_RPC=${ETHEREUM_ENDPOINT}
        - DB_HOST=db
        - DB_NAME=${DB_NAME}
        - DB_PORT=5432
        - DB_USERNAME=${DB_USERNAME}
        - DB_PASSWORD=${DB_PASSWORD}
        - VERBOSITY=${VERBOSITY}
        - CONFIG_PATH=${CONFIG_PATH}
        - RONIN_TASK_INTERVAL=${RONIN_TASK_INTERVAL}
        - RONIN_TRANSACTION_CHECK_PERIOD=${RONIN_TRANSACTION_CHECK_PERIOD}
        - RONIN_MAX_PROCESSING_TASKS=${RONIN_MAX_PROCESSING_TASKS}
        - ETHEREUM_GET_LOGS_BATCH_SIZE=${ETHEREUM_GET_LOGS_BATCH_SIZE}
        - BRIDGE_STATS_URL=${BRIDGE_STATS_URL}
        - BRIDGE_STATS_NODE_NAME=${BRIDGE_STATS_NODE_NAME}
        - BRIDGE_STATS_SECRET=${BRIDGE_STATS_SECRET}
      depends_on:
        - db
        - node
  ```

  This compose file defines three services:
     * `node`, which pulls a Ronin node image from the GitHub Container Registry.
     * `bridge`, which pulls a bridge image.
     * `db`, which builds a Postgres database for the bridge.

4. Create an `.env` file to store configuration parameters for the services:

  ```
  vim .env
  ```

5. Paste the following into `.env` and replace placeholder values (like *`INSTANCE_NAME`*) with your node's information:

  ```
  # Your Ethereum RPC endpoint
  ETHEREUM_ENDPOINT=ETHEREUM_ENDPOINT

  # The name of your node that you want displayed on https://stats.roninchain.com/
  INSTANCE_NAME=INSTANCE_NAME

  # The latest version of the node's image as listed in https://docs.roninchain.com/validators/setup/upgrade-validator
  NODE_IMAGE=NODE_IMAGE

  # The latest version of the bridge's image as listed in https://docs.roninchain.com/bridge-operators/setup/upgrade-bridge-operator
  BRIDGE_IMAGE=BRIDGE_IMAGE

  # Your bridge operator private key without the 0x prefix
  BRIDGE_OPERATOR_PRIVATE_KEY=BRIDGE_OPERATOR_PRIVATE_KEY

  # If you're a governor, uncomment this line and replace with your bridge voter key, without the 0x prefix
  # BRIDGE_VOTER_PRIVATE_KEY=BRIDGE_VOTER_PRIVATE_KEY

  # Your validator private key without the 0x prefix
  VALIDATOR_PRIVATE_KEY=VALIDATOR_PRIVATE_KEY

  # The password used to encrypt the node's private key file
  PASSWORD=PASSWORD

  # BLS Wallet features
  # Your BLS private key without the 0x prefix
  BLS_PRIVATE_KEY=BLS_PRIVATE_KEY_WITHOUT_0x

  # The password used to encrypt the BLS private key file
  BLS_PASSWORD=BLS_PASSWORD

  # Whether to participate in the finality vote broadcast
  ENABLE_FAST_FINALITY=true

  # Whether to produce the finality vote
  ENABLE_FAST_FINALITY_SIGN=true

  # Whether to automatically generate BLS private key
  BLS_AUTO_GENERATE=false

  # Whether to show the raw BLS private key after decryption
  BLS_SHOW_PRIVATE_KEY=false

  DB_NAME=bridge
  DB_USERNAME=postgres

  # The Postgres database password
  DB_PASSWORD=DB_PASSWORD

  RPC_ENDPOINT=http://node:8545

  CONFIG_PATH=config.mainnet.json

  RONIN_TASK_INTERVAL=3
  RONIN_TRANSACTION_CHECK_PERIOD=50
  RONIN_MAX_PROCESSING_TASKS=200
  ETHEREUM_GET_LOGS_BATCH_SIZE=100

  BOOTNODES=enode://cfa5f00c55eba79f359c9d95f5c0b2bb8e173867ffbb6e212c6799a52918502519e56650970e34caf1cd17418d4da46c3243588578886c3b4f8c42d1934bf108@104.198.242.88:30303,enode://f500391c41906a1dae249df084a3d1659fe602db671730b2778316114a5f7df44a0c6864a8dfffdc380fc81c6965dd911338e0e2591eb78a506857015d166250@34.135.18.26:30303,enode://fc7b8ceafe16e6f79ab2da3e73d0a3163d0c28efe0778863102f8f27758986fe28c1540a9a0bbdff29ab93ad1c5803462efe6c98165bbb404d9d099a55f1d2c9@130.211.208.201:30303

  NETWORK_ID=2020

  GASPRICE=20000000000

  MINE=true

  VERBOSITY=3

  CHAIN_STATS_WS_SECRET=WSyDMrhRBe111
  CHAIN_STATS_WS_SERVER=ronin-stats-ws.roninchain.com

  RONIN_PARAMS=--http.api eth,net,web3,consortium --miner.gaslimit 100000000 --miner.gasreserve 10000000
  
  BRIDGE_STATS_NODE_NAME=<your_node_name>
  BRIDGE_STATS_URL=wss://ronin-stats-ws.roninchain.com/bridge
  BRIDGE_STATS_SECRET=WSyDMrhRBe111
  ```

1. (Optional) Download the snapshot from [ronin-snapshot](https://github.com/axieinfinity/ronin-snapshot)

  ```
  cd ~/ronin/chaindata/data/ronin/
  wget -q -O - <snapshot URL> | tar -I zstd -xvf -
  ```

7. Start the node:

  ```
  cd ~/ronin && docker-compose up -d
  ```

  This command pulls a Ronin node image, a bridge image, a Postgres database, and starts the services you defined.

8. After a few minutes, check the status of your node on the [Ronin Network Status](https://ronin-stats.roninchain.com/) page. If it's green, the node is connected and up to date with the network.

9. Review the log for the validator and the bridge (the node should sync to the latest block for making the bridge work).

  ```
  docker logs node -f --tail 100
  docker logs bridge -f --tail 100
  ```