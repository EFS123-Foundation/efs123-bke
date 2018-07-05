# <img src='https://i.imgur.com/E56MPry.png' height='60' alt='Demux Logo' />

Demux is a backend infrastructure pattern for sourcing blockchain events to deterministically update queryable datastores and trigger side effects. This library serves as a reference implementation of that pattern for use with Node applications.

## Installation

Using `yarn`:
```bash
yarn add demux-js
```

Using `npm`:
```bash
npm install demux-js --save
```
## Overview

Taking inspiration from the [Flux Architecture](https://facebook.github.io/flux/docs/in-depth-overview.html#content) pattern and [Redux](https://github.com/reduxjs/redux/), Demux was born out of the following qualifications:

1. A separation of concerns between how state exists on the blockchain and how it is queried by the client front-end
1. Client front-end not solely responsible for determining derived, reduced, and/or accumulated state
1. Ability for blockchain events to trigger new transactions, as well as other side effects outside of the blockchain
1. The blockchain as the single source of truth for all application state

### Separated Persistence Layer

Storing data in indexed state on blockchains can be useful for three reasons: decentralized consensus of computation results, usage of state from within other blockchain computations, and for retrieval of state for use in client front-ends. When building more complicated front-ends, you run into a few problems when retrieving directly from indexed blockchain state:

* The query interface used to retrieve the indexed data is limited. Complex data requirements can mean you either have to make an excess number of queries and process the data on the client, or you must store additional derivative data on the blockchain itself.
* Scaling your query load means creating more blockchain endpoint nodes, which can be very expensive.

Demux solves these problems by off-loading queries to any persistence layer that you want. As blockchain events happen, your chosen persistence layer is updated deterministically by `updater` functions, which can then be queried by your front-end through a suitable API (for example, REST or GraphQL).

This means that we can separate our concerns: for data that needs decentralized consensus of computation or access from other blockchain events, we can still store the data in indexed blockchain state, without having to worry about tailoring to front-end queries. For data required by our front-end, we can pre-process and index data in a way that makes it easy for it to be queried, in a horizontally scalable persistence layer of our choice. The end result is that both systems can serve their purpose more effectively.

### Side Effects

Since we have a system for acting upon specific blockchain events deterministically, we can utilize this system to manage non-deterministic events as well. These `effect` functions work almost exactly the same as `updater` functions, except they run asynchronously, are not run during replays, and modifying the deterministic datastore is off-limits. Examples include: signing and broadcasting a transaction, sending an email, and initiating a traditional fiat payment.

### Single Source of Truth

There are other solutions to the above problems that involve legacy persistence layers that are their own sources of truth. By deriving all state from the blockchain, however, we gain the following benefits:

* If the accumulated datastore is lost or deleted, it may be regenerated by replaying blockchain actions
* As long as application code is open source, and the blockchain is public, all application state can be audited
* No need to maintain multiple ways of updating state (submitting transactions is the sole way)

## Data Flow

<img src='https://i.imgur.com/MFfGOe3.png' height='492' alt='Demux Logo' />

1. Client queries API for data
1. Client sends transaction to blockchain
1. Action Watcher invokes Action Reader to check for new blocks
1. Action Reader sees transaction in new block, parses actions
1. Action Watcher sends actions to Action Handler
1. Action Handler processes actions through Updaters and Effects
1. Actions run their corresponding Updaters, updating the state of the Datastore
1. Actions run their corresponding Effects, triggering external events
1. Client queries API for (updated) data

## Usage

This library provides the following classes:

* [**`AbstractActionReader`**](src/demux/readers/): Abstract class used for implementing your own Action Readers
    * [`NodeosActionReader`](src/demux/readers/eos/): Action reader that reads actions from EOS Nodeos nodes


* [**`AbstractActionHandler`**](src/demux/handlers/): Abstract class used for implementing your own Action Handlers
    * [`MassiveActionHandler`](src/demux/handlers/postgres/): Action handler that utilizes Massive.js to make SQL queries to a Postgres database


* [**`BaseActionWatcher`**](src/demux/watchers/): Base class that implements a ready-to-use Action Watcher

*(Click each above for detailed method and subclassing usage.)*

## Example

```js
const {
  readers: {
    eos: { NodeosActionReader } // Let's read from an EOS node
  },
  watchers: { BaseActionWatcher }, // Don't need anything special, so let's use the base Action Watcher
} = require("demux-js")

// Assuming you've already created a subclass of AbstractActionHandler
const MyActionHandler = require("./MyActionHandler")

// Import Updaters and Effects, which are arrays of objects:
// [ { actionType:string, (updater|effect):function }, ... ] 
const updaters = require("./updaters")
const effects = require("./effects")

const actionHandler = new MyActionHandler({
  updaters,
  effects,
})

const actionReader = new NodeosActionReader({
  nodeosEndpoint: "http://some-nodeos-endpoint:8888", // Locally hosted node needed for reasonable indexing speed
  startAtBlock: 12345678, // First actions relevant to this dapp happen at this block
})

const actionWatcher = new BaseActionWatcher({
  actionReader,
  actionHandler,
  pollInterval: 250, // Poll at twice the block interval for less latency
})

actionWatcher.watch() // Start watch loop
```

For more complete examples, [see the examples directory](examples/).
