# Warp D.R.E.

A Delegated Resolution Environment for Warp Contracts.  
D.R.E. nodes are a special kind of nodes that are responsible for evaluating and serving contracts' states - 
in a decentralized manner - taking the burden of the state evaluation from the end users.

* [The reasoning](#the-reasoning)
* [Nodes](#nodes)
* [How it works?](#how-it-works)
    * [1. Messages processing](#1-messages-processing)
    * [2. Caching](#2-caching)
    * [3. Events](#3-events)
    * [4. Blacklisting](#4-blacklisting)
    * [5. Endpoints](#5-endpoints)
* [How to run?](installation)
* [Future work](#future-work)

## The reasoning

The purpose of implementing Warp D.R.E. is to address the following issues.

1. Evaluation of high-interaction contracts - evaluating contracts with thousands of interactions is a hassle for the
   user's CPU. Response-time and UX can be improved significantly when computation is delegated to D.R.E.

2. Interaction with high-risk contracts - some contracts may perform risky/unsafe operations. Contract interaction via
   Warp D.R.E. ensures the safety of users' devices.

3. Insights into PST tokens' - Providing aggregate information about PST tokens' status and count has always been
   challenging. With D.R.E.'s aggregation tool, one can check global address data with just a few clicks.

4. Avoid centralised, closed-sourced solutions - private, unknown processing logic, and tightly coupled to a specific
   cloud vendor (such as GCP or AWS) belong to the web2 era. Warp D.R.E. is built on the principle of "Don't trust,
   verify". Once initial testing is complete, we will open up D.R.E nodes for public participation.

## Nodes

Currently, there are five nodes are available:

1. https://dre-1.warp.cc
2. https://dre-2.warp.cc
3. https://dre-3.warp.cc
4. https://dre-4.warp.cc
5. https://dre-5.warp.cc

To verify if your contract is available - check the `/contract` endpoint,  
eg: https://dre-1.warp.cc/contract?id=-8A6RexFkpfWwuyVO98wzSFZh0d6VJuI-buTJvlwOJQ&events=true

## How it works?

### 1. Messages processing

1. The **Warp Gateway** and **Warp Sequencer** are publishing messages to a pub/sub (currently Redis, Streamr in the
   future) - to a `contracts` channel.
2. Two kind of messages are being sent:
    1. contract deployment notification (whenever a new contract is being deployed, either via Warp or directly to
       Arweave L1).
       Message format:
       ```js
       {
          "contractTxId": "<contract_txId>",
          "test": <true|false>,
          "source": "warp-gw"
          "initialState": <contract_initial_state>
       }
       ```
    2. contract interaction notification (whenever a new interaction is being registered, either via **Warp Sequencer**
       or directly to Arweave L1).   
       Message format:
       ```js
       {
          "contractTxId": "<contract_txId>",
          "test": <true|false>,
          "source": "warp-gw"
          "interaction": <new_interaction>
       }
       ```
3. D.R.E. is subscribing for messages on the `contracts` channel.
4. D.R.E. maintains two internal queues. The queues are implemented
   with [BullMQ](https://github.com/taskforcesh/bullmq#readme).
    1. **register** queue for registering new contracts (for messages with an initial state)
    2. **update** queue for processing contracts' interactions (for messages with a new interaction)
5. Each of the queues have its own set of workers. Each worker runs as a
   separate, [sandboxed processor](https://docs.bullmq.io/guide/workers/sandboxed-processors)
   that is isolated from the rest of the code.
6. Each message that comes to D.R.E. is first validated (message format, tx id format, etc.).

* If it is a `contract deployment` notification AND contract is not yet registered - a new job is added to the **
  register** queue.  
  The processor that picks up such job simply sets the `initialState` from the incoming message in the D.R.E. cache.
* If it is a `contract update` notification AND contract is not yet registered - a new job is added to the **register**
  queue.    
  The processor that picks up such job evaluates the contract's state from scratch.
* If it is a `contract update` notification AND contract is already registered - a new job is added to the **update**
  queue.
  The processor that picks up such job either
    * evaluates the state only for the interaction from the message - if D.R.E. has a contract cached
      at `message.interaction.lastSortKey`
    * evaluates the state for all the interactions from the lastly cached - if D.R.E has a contract cached at a lower
      sort key than `message.interaction.lastSortKey`.

### 2. Caching

D.R.E. is currently using two kinds of caches

1. Warp Contracts SDK's internal cache (with
   the [LMDB plugin](https://github.com/warp-contracts/warp-contracts-lmdb#warp-contracts-lmdb-cache))
2. [better-sqlite3](https://github.com/WiseLibs/better-sqlite3#better-sqlite3-) based cache. This cache is used for
   serving data via D.R.E. endpoints - in order not to interfere
   with the Warp Contracts SDK internal cache.
   It also serves as a form of a backup.  
   [WAL mode](https://github.com/WiseLibs/better-sqlite3/blob/master/docs/performance.md#performance) is being used for
   increased performance.

The evaluated state (apart from being automatically cached by the Warp Contracts SDK) is additionally:

1. Signed by the D.R.E.'s wallet. The data for the signature consists of:
    ```js
   
   // state is stringified with 'safe-stable-stringify', not JSON.stringify.
   // JSON.stringify is non-deterministic.
   const stringifiedState = stringify(state);
   const hash = crypto.createHash('sha256');
   hash.update(stringifiedState);
   const stateHash = hash.digest('hex');
   
   const dataToSign = await deepHash([
      arweave.utils.stringToBuffer(owner), // a jwk.n of the D.R.E's wallet
      arweave.utils.stringToBuffer(sortKey), // a sort key at which the state has been evaluated
      arweave.utils.stringToBuffer(contractTxId), // what could it be....
      arweave.utils.stringToBuffer(stateHash), // a state hash
      arweave.utils.stringToBuffer(stringify(manifest)) // a full node's manifest
    ]);
    ```

The `manifest` contains all the data that was used for the evaluation of the state, including

* the libraries version (warp-contracts and all warp plugins)
* the evaluation options
* git commit hash at which the node was running during state evaluation.  
  A manifest may look like this:

```json
{
  "gitCommitHash": "19a327e141300772985ef1b9e44c01c17de4f668",
  "warpSdkConfig": {
    "warp-contracts": "1.2.23",
    "warp-contracts-lmdb": "1.1.1",
    "warp-contracts-nlp-plugin": "1.0.5"
  },
  "evaluationOptions": {
    "useVM2": true,
    "maxCallDepth": 5,
    "maxInteractionEvaluationTimeSeconds": 10,
    "allowBigInt": true,
    "internalWrites": true
  },
  "owner": "<very_long_string>",
  "walletAddress": "uOImfFuq_KVTHZfsKdC-3WriZwtInyEkxe2MlU_le9Q"
}
```

2. After signing - the state is stored in the `better-sqlite3` (including the validity, error messages,
   state hash, node's manifest at the time of evaluation, signature).  
   **NOTE:** This data allows to recreate the exact environment that was used to evaluate the state
   (and verify it locally).
3. As a last step - a new state is being published on pub/sub `states` channel.
   The messages on this channel are being listened by
   the [Warp Aggregate Node](https://github.com/warp-contracts/warp-aggregate-node), which combines
   the data from all the D.R.E.s.

### 3. Events

To give a better understanding of that is going on, the D.R.E. registers events at certain points of processing.
Each event consists of:

1. event type
2. timestamp
3. optional message

The event type is one of:

* `REQUEST_REGISTER` - a registration request has been received for a contract
* `REQUEST_UPDATE` - an update request has been received for a contract
* `REJECT` - an update or registration request has been rejected either because of incoming
  message validation errors or because contract is blacklisted.
* `FAILURE` - an error have occurred during contract evaluation
* `EVALUATED` - contract has been successfully evaluated (but it required loading interactions from Warp GW)
* `PROGRESS` - added after evaluating each 500 interactions (useful for tracking the evaluation progress of a registered
  contract that has thousands of interactions)
* `UPDATED` - contract has been successfully updated using the interaction sent in the message

### 4. Blacklisting

Whenever a contract reaches a certain amount of failures (3 for both dre-1 and dre-2) - it is blacklisted
and ignored.

### 5. Endpoints

:::tip
If you're using Rust, you can interact with D.R.E. with the help of a great library developed by the [https://archivetheweb.com/](archivetheweb)
team - [https://github.com/archivetheweb/warp_dre](https://github.com/archivetheweb/warp_dre).
:::

1. `/`, `/status` - contains information about node manifest, workers configuration,
   queues status, etc.
2. `/contract` - returns all data about a given contract.
   Parameters:
    * `id` - the contract tx id
    * `state` - `true|false` - whether state should be returned. `true` by default  
    * `validity` - `true|false` - whether validity should be returned. `false` by default
    * `errorMessages` -`true|false` - whether error messages should be returned. `false` by default
    * `errors` - whether errors thrown during the evaluation process should be returned (e.g. the errors thrown by Warp Gateway, Arweave gateway). `false` by default
    * `events` `true|false` - whether events for this contract should be returned.. `false` by default
    * `query` - a [jsonpath-plus](https://www.npmjs.com/package/jsonpath-plus) expression that allows to query the
      current state (for example query for the balance of a concrete wallet address).
      Whenever a `query` parameter is passed, a `result` is returned instead of `state`.  
      Example query:  
      https://dre-1.warp.cc/contract?id=5Yt1IujBmOm1LSux9KDUTjCE7rJqepzP7gZKf_DyzWI&query=$.balances.7FljANNIG0FfumNxShdt3ELtL33HMoxnoHHct2TQdvE  
3. `/cached` - returns a list of all cached contracts
4. `/blacklist` - returns all contracts that failed at least once. If contract failed 3 times - it is blacklisted.
5. `/errors` - returns all errors for all contracts
6. `/sync` - schedule force synchronization of a given contract. This endpoint can't be called more frequent then every 10 seconds.
   Parameters:
    * `id` - text contract tx id

   Example:
   ```bash
   http ":8080/sync?id=KT45jaf8n9UwgkEareWxPgLJk4oMWpI5NODgYVIF1fY"
   HTTP/1.1 200 OK
   
   Scheduled for update
   ```
   Error when too frequent requests per one contract:
   ```bash
   http ":8080/sync?id=KT45jaf8n9UwgkEareWxPgLJk4oMWpI5NODgYVIF1fY"
   HTTP/1.1 500 Internal Server Error

   Chill out and wait 10s
   ```

## Future work

1. Sync the local state with D.R.E. inside the Warp Contract SDK while connecting to a contract
   (i.e. while calling `warp.contract(<contract_tx_id)`).
2. Replace Redis with [Streamr](https://streamr.network/) - in progress
3. More nodes
4. One-command deployment via Docker
5. `unsafeClient` compatible nodes
6. Bundle all the evaluated states
7. A decentralized validation layer built on of the D.R.E. nodes
8. Stats/dashboard


