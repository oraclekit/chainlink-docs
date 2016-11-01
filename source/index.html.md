---
title: Smart Oracle Documentation

<!-- language_tabs: -->
  <!-- - shell -->

toc_footers:
  - <a href='#'>Get a hosted version</a>

includes:
  - errors

search: true
---

# Introduction

Welcome to the Smart Oracle documentation. An oracle provides data into the blockchain that cannot be accessed by the blockchain itself, due to consensus constraints.

The work specified for a Smart Oracle is called an Assignment. Assignments can be handled by the oracle itself, or the oracle's capabilities can be expanded by creating an adapter for the oracle.

## Installation

First set up your configuration, by creating a `.env` file, that has at least all of the variables set in the [.env.example](https://github.com/smartoracles/core-ruby/blob/master/.env.example). You will need a URL for a running instance of Postgres and URL for an Ethereum connected node.

```shell
docker pull smartcontract/smartoracle
docker run -t --env-file=.env smartcontract/smartoracle
```
Once your configuration is set, pull down the oracle image and run it with your configuration.

For more information on setting up an instance, or building it locally, visit the [wiki page on installation](https://github.com/smartoracles/core-ruby/wiki/Install).

## Coordinators and Authentication
```shell
curl "http://localhost:6688/assignments/XID" -u apiKey:apiSecret
```
To access an assignment you must have a set of coordinator credentials. An initial set of coordinator credentials are generated and printed when the node's `initialize` command is run.

All assignments are associated with a coordinator, so in order to create an assignment you must authenticate using its coordinator's credentials. After creating an assignment all subsequent updates will be sent to its coordinator, if the coordinator has specified a URL.

Authentication is handed via HTTP Authentication.

# Assignments

## Anatomy of an Assignment
Assignments are the core of the Smart Oracle model, they are the specifications of work given to an oracle. The main pieces of an assignment consist of the [adapter](#adapter-type-and-parameters), the [scheduling](#scheduling), and the [payment specification](#payment). Optionally, an assignment can have a description and signatures from the parties involved.

### Adapter Type and Parameters
Every assignment has a type, which specifies the kind of adapter the oracle will use to perform the work requested. Each different type of adapter requires different parameters, some passed on the blockchain, some passed off chain. The parameters each adapter will accept are specified ahead of time by the adapter's creator.

The current Smart Oracle ships with two built in adapters: `ethereumBytes32JSON` for turning JSON APIs into Ethereum oracles, and `bitcoinComparisonJSON` for releasing Bitcoin escrow based on the values of JSON APIs. For more information on setting up custom adapters, see the section on [creating an adapter](TODO).

### Scheduling
```json
{
  "minute": "*",
  "hour": "*",
  "dayOfMonth": "*",
  "monthOfYear": "*",
  "dayOfWeek": "*",
  "startAt": "1477941966",
  "endAt": "1793474734"
}
```
Some oracle services are needed on a scheduled basis. For these use cases, the Smart Oracle a simple way to schedule tasks based on the [Cron](https://en.wikipedia.org/wiki/Cron) scheduling format.

Assignments that do not require a schedule, instead working "on demand," can skip the schedule or offer a combination of both. All assignments require a start time and allow for an optional start time. Times are specified using Unix timestamps(UTC).

### Payment
```json
{
  "currency": "ETH",
  "perDay": "0",
  "perRequest": "1000000000000000"
}
```
Space on blockchains is limited and so generally costly to utilize. On the other side of the oracle, API calls to protected APIs are often behind pay walls. Oracle services by their nature come with expenses. For this reason, the Smart Oracle platform has a way to specify service prices for each adapter.

Prices are generally set per request made by the oracle; This can be prepaid for scheduled services, or paid per request for on demand services. Additionally, prices can be set based on the duration of required availability of the oracle.

The currency is configurable based on the networks that the oracle operates in. Currency amounts are always specified in the lowest possible denomination of the currency(Satoshis, Wei, etc.).

## Create
```shell
curl -u apiKey:apiSecret -X POST -H 'Content-Type: application/json'
  -d '{"assignment":{"adapterType":"ethereumBytes32JSON", "adapterParams": {"endpoint": "https://bitstamp.net/api/ticker/", "fields": ["last"]}, "description":"Bitcoin Price","schedule":{"endAt":"1478028219","hour":"0","minute":"0"}},"assignmentHash":"b2e55902bb7728871fa69f503007577ef8a1ae449f486b5c0aaf644661d216d1","signatures":[],"version":"0.1.0"}'
  http://localhost:6688/assignments
```

> JSON response:

```json
{
  "assignmentHash": "b2e55902bb7728871fa69f503007577ef8a1ae449f486b5c0aaf644661d216d1",
  "signature": "1cb6770f6977710c7c3e0d336d3d2244a0d276056029cb83f414d8641ef412218d2a36df9ccde31ee50310d9eef098fa153b34ad9075d02737243a59c5dbd6d357",
  "xid": "561b78af-e163-4972-9d4a-5dc15e02d977"
}
```

A `POST` to `/assignments` will return the `XID`, or external ID, which is used to identify the assignment in the future. The response also includes a hash of the assignment specified, and a signature of that hash to attest to the oracle's acceptance of the assignment.

<aside class="success">
Make sure to grab the XID to refer to the assignment in the future. The assignment hash is intentionally not used in the future, as the XID is not easily linkable with the assignment.
</aside>

# Assignment Notifications

Once an assignment is created, it will automatically run and update itself based on its schedule and the logic in the adapter. Updates cannot be fed in by the coordinator, but the coordinator can receive push notifications whenever the assignment is updated. Simply set the URL of the assignment's coordinator to receive push notifications.

All push notifications are authorized with the same credentials used to create the assignment.

## Create Assignment Oracle
```json
{
  "oracle": {
    "address": "0x72c8379f845bb3cb30e02ef1feb84742debc1efb",
    "json_abi": "contract Oracle {\n  bytes32 currentValue;\n  address creator;\n\n  function Oracle() {\n    creator = msg.sender;\n  }\n\n  function update(bytes32 newCurrent) {\n    if (msg.sender != creator) return;\n    currentValue = newCurrent;\n  }\n\n  function current() constant returns (bytes32 current) {\n    return currentValue;\n  }\n\n  function () constant returns (bytes32 current) {\n    return currentValue;\n  }\n}",
    "read_address": "9fa6a6e3",
    "solidity_abi": "contract Oracle{function Oracle();function update(bytes32 newCurrent);function current()constant returns(bytes32 current);}"
  },
  "xid": "f0577c3e-9e5a-4840-9c9b-37d326c3d2e3"
}
```

`POST` to `/assignments/:xid/instructions`

If an assignment needs some on-chain setup, it cannot immediately respond with all of its integration details on creation, it needs to wait for the blockchain confirmations before. Once the confirmations needed for the assignment set up have occured, the oracle will push integration instructions to the coordinator.

Parameter | Type | Description
---- | ----- | --------
address | string | Ethereum address location
json_abi | string | a stringified version of the JSON ABI for the contract
read_address | string | the hash for the read function of the Ethereum contract
solidity_abi | string | the Solidity ABI to include in a contract using this oracle
xid | string | the XID of the related assignment


## Create Assignment Snapshot

```json
{
  "assignment_xid": "f0577c3e-9e5a-4840-9c9b-37d326c3d2e3",
  "description": "Blockchain ID: 0x5803d6bb728b002d5a9aedc2eebec404fcbc2e2966f5e81abe297995b2980046",
  "description_url": "https://testnet.etherscan.io/tx/0x5803d6bb728b002d5a9aedc2eebec404fcbc2e2966f5e81abe297995b2980046",
  "details": {
    "current": "10000000034567123",
    "total": "98700000035511224"
  },
  "status": "in progress",
  "summary": "Assignment 'f0577c3e-9e5a-4840-9c9b-37d326c3d2e3' updated its value to \"1,000,000.00\"",
  "value": "1,000,000.00",
  "xid": "0x5803d6bb728b002d5a9aedc2eebec404fcbc2e2966f5e81abe297995b2980046"
}
```

`POST` to `/assignments/:assignment_xid/snapshots`

Each time the assignment is updated it creates a snapshot. A snapshot is the current status of the assignment. Whenever a snapshot is created, it is pushed to the coordinator. Further information is provided in machine readable form in the `details` section, in various formats depending on the adapter. Information is provided in human readable form in the `summary` and `description` fields, as well as the `description_url`.

Parameter | Type | Description
---- | ----- | --------
assignment_xid | string | the XID of the related assignment
description | string | a detailed human readable description of the snapshot
description_url | string | a supporting URL relating to the update
details | object | a JSON object of extra supporting information returned by the adapter
status | string | a description of the assignment's current status
summary | string | a short human readable summary of the snapshot
value | string | the latest value returned by the adapter
xid | string | an unique external ID to identify the snapshot by

<aside class="success">
Snapshot IDs are unique, you should never receive duplicate snapshots.
</aside>

## Update Assignment

```json
{
  "signatures": ["30450221009f4e3e0ab2ede2ca57a03ea67f6d16641568b214e38b650db95cd3490b3f213902202b8ba93235c6cafa4f8a643c72977427e8fde95a2e8010cb3336125182d62cef"],
  "status": "completed",
  "xid": "f0577c3e-9e5a-4840-9c9b-37d326c3d2e3"
}
```

`PATCH` to `/assignments/:xid`

When the assignment is finished, a notification will be pushed to the coordinator. If any signatures were needed from the oracle, for things like releasing Bitcoin escrow, a signature is returned in addition to a status update.

Parameter | Type | Description
---- | ----- | --------
signatures | array(string) | if a signature is required, like for Bitcoin escrow, it returns a signature for the outcome that was determined
status | string | Either "completed" or "failed"
xid | string | the XID of the related assignment


# Adapters

The Smart Oracle image ships with functionality out of the box to connect Ethereum contracts and Bitcoin escrow to JSON APIs(the lingua franca of the web). But the real power of the Smart Oracle platform lies in its abilitiy to be extended.

Adapters can be configured with the Smart Oracle core to add functionality. Whether it's a different data format like XML, or special computation, the Smart Oracle makes it possible to further extend off-chain capabilities via adapters.

### APIs

Adapters integrate with the core in a service oriented model, so they can run locally, next to the core or on remote servers. The minimum integrations an adapter needs to support are based on whether the adapter will push information or the core will pull. The APIs needed to create a custom adapter are listed below.

### Oracle Schema

In order to make an adapter's input and output predictable, adapters need to specify schemas. Schemas are created using a JSON Schema for expected prerequisites and on/off-chain inputs/outputs. See the [specification here](https://github.com/smartoracles/spec).


## Create Assignment

__*Required*__: `POST` to adapter path `/assignments`

The action used for an oracle to pass an assignment over to an adapter.

The expected response should include:

Parameter | Type | Description
---- | ----- | --------
xid | string | the unique identifier to associate an assignment with
end_at | string | a timestamp in Unix Timestamp(UTC) format.
data | object | a JSON object containing all information specified in the adapter's schema


## Create Snapshot _(Pull)_

`POST` to adapter path `/assignments/:assignment_xid/snapshots`

The action used for an oracle to pass an assignment over to an adapter.

Parameter | Type | Description
---- | ----- | --------
description | string | a detailed human readable description of the snapshot
description_url | string | a supporting URL relating to the update
details | object | a JSON object of extra supporting information returned by the adapter
fulfilled | boolean | marks whether the snapshot has been completed or not
status | string | a description of the assignment's current status
summary | string | a short human readable summary of the snapshot
value | string | the latest value returned by the adapter
xid | string | an unique external ID to identify the snapshot by

### Unfulfilled Snapshots

Creating a snapshot may require more time than you are willing to leave a request hanging for. Snapshots that are requested via the pull style can be marked as unfulfilled upon creation, and fulfilled at a later time.

In order to fulfill an unfulfilled snapshot you must implement the update snapshot integration.


## Delete Assignment

`DELETE` to `/assignments/:xid`

Used to indicate the end of an assignment.

Parameter | Type | Description
---- | ----- | --------
status | string | optional string to specify the final state of the assignment
xid | string | identifier of the assignment


## Create Snapshot _(Push)_

`POST` pushed to the core path `/snapshots`

__This is an integration that originates in the adapter and is pushed from the adapter to the core.__

A pushed snapshot is automatically marked as fulfilled.

Parameter | Type | Description
---- | ----- | --------
assignment_xid | string | identifier for the associated assignment
description | string | a detailed human readable description of the snapshot
description_url | string | a supporting URL relating to the update
details | object | a JSON object of extra supporting information returned by the adapter
status | string | a description of the assignment's current status
summary | string | a short human readable summary of the snapshot
value | string | the latest value returned by the adapter
xid | string | an unique external ID to identify the snapshot by

## Update Snapshot _(Push)_

`PATCH` pushed to the core path `/snapshots/:xid`

__This is an integration that originates in the adapter and is pushed from the adapter to the core.__

Only unfulfilled snapshots can be updated. When a snapshot is updated it is automatically marked as fulfilled.

Parameter | Type | Description
---- | ----- | --------
assignment_xid | string | identifier for the associated assignment
description | string | a detailed human readable description of the snapshot
description_url | string | a supporting URL relating to the update
details | object | a JSON object of extra supporting information returned by the adapter
status | string | a description of the assignment's current status
summary | string | a short human readable summary of the snapshot
value | string | the latest value returned by the adapter
xid | string | an unique external ID to identify the snapshot by
