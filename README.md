# HapiGER

<img src="./assets/hapiger300x200.png" align="right" alt="HapiGER logo" />

Providing good recommendations can create greater user engagement and directly provide value by recommending items the customer might additionally like. However, many applications don't provide recommendations to users because of the difficulty in implementing a custom engine or the pain of using an off-the-shelf engine.

**HapiGER** is a recommendations service that uses the [Good Enough Recommendations (**GER**)](https://www.npmjs.com/package/ger), a scalable, simple recommendations engine, and the [Hapi.js](https://hapijs.com/) framework. It has been developed to be easy to integrate, easy to use and very scalable.

[Project Site](http://www.hapiger.com)

## Quick Start Guide

<br/>
***
#### Install HapiGER

Install with `npm`

```bash
npm install -g hapiger
```

<br/>
***

#### Start HapiGER

By default it will start with an in-memory event store (events are not persisted)

```bash
hapiger
```

*There are also PostgreSQL, RethinkDB and MySQL event stores for persistence and scaling*

<br/>
***

#### Create a Namespace

The first thing to do is to create a namespace, which is a bucket that person events are put into:

```bash
curl -X POST 'http://localhost:3456/namespaces' -d'{
    "namespace": "movies"
  }'
```

<br/>
***

#### Create some Events

An event occurs when a person actions something, e.g. `Alice` `view`s `Harry Potter`:

```bash
curl -X POST 'http://localhost:3456/events' -d '{
    "events": [
    {
      "namespace": "movies",
      "person":    "Alice",
      "action":    "view",
      "thing":     "Harry Potter"
    }
  ]
}'
```

Then, `Bob` also `view`s `Harry Potter` (now `Bob` has similar viewing habits to `Alice`)

```bash
curl -X POST 'http://localhost:3456/events' -d '{
    "events": [
    {
      "namespace": "movies",
      "person":    "Bob",
      "action":    "view",
      "thing":     "Harry Potter"
    }
  ]
}'
```

When a person actions and thing, it serves two purposes in HapiGER:

1. It is used to measure a persons similarity to other people
2. It can be a recommendation of that thing

For example, when `Bob` `buy`s `LOTR`

```bash
curl -X POST 'http://localhost:3456/events' -d '{
    "events": [
    {
      "namespace":  "movies",
      "person":     "Bob",
      "action":     "buy",
      "thing":      "LOTR",
      "expires_at": "2017-10-12"
    }
  ]
}'
```

This is an action that can be used to find similar people **AND** it can be seen as `Bob` recommending  `LOTR`. For an event to be a recommendation as well it must have an expiration date set with `expires_at`, which is how long the recommendation will be available for.

<br/>
***

#### Recommendations

Recommendations can be generated by either passing the name of a `person` or a `thing`, the `namespace` to generate recommendations from, and a `configuration` which defines how to search for recommendations. The `configuration` is passed to GER and the available variables are listed in the [GER Documentation](https://github.com/grahamjenson/ger).

What movies should we recommend `Alice`?

```bash
curl -X POST 'http://localhost:3456/recommendations' -d '{
    "namespace": "movies",
    "person": "Alice",
    "configuration": {
      "actions" : {"view": 5, "buy": 10}
    }
}'
```

will return:

```
{
  "recommendations": [
    {
      "thing": "LOTR",
      "weight": 0.44721359549996,
      "last_actioned_at": "2015-10-12T17:04:14+01:00",
      "last_expires_at": "2017-10-12T01:00:00+01:00",
      "people": [
        "Bob"
      ]
    }
  ],
  "neighbourhood": {
    "Bob": 0.44721359549996,
    "Alice": 1
  },
  "confidence": 0.00036398962692384
}
```

`Alice` should buy `LOTR` as it was recommended by `Bob` with a weight of about `0.4`.

<br/>

What movies should we recommend to someone looking at `Harry Potter`?

```
curl -X POST 'http://localhost:3456/recommendations' -d '{
    "namespace": "movies",
    "thing": "Harry Potter",
    "configuration": {
      "actions" : {"view": 5, "buy": 10}
    }
}'
```

returns

```
{
  "recommendations": [
    {
      "thing": "LOTR",
      "weight": 0.70710678118655,
      "last_actioned_at": "2015-10-13T08:53:00.885Z",
      "last_expires_at": "2017-10-12T00:00:00.000Z",
      "people": [
        "Bob"
      ]
    }
  ],
  "neighbourhood": {
    "LOTR": 0.70710678118655
  },
  "confidence": 0.0010667601060058
}
```

The person should buy `LOTR` as it was recommended by `Bob` with a weight of about `0.7`.

*The `confidence` of these recommendations is low because there are not many events in the system. As you add events the confidence will increase*

<br/>
***

#### Event Stores

The "in-memory" memory event store is the default, this will not scale well or persist event so is not recommended for production.

The **recommended** event store is **PostgreSQL**, which can be used with:

```
hapiger --es pg --esoptions '{
    "connection":"postgres://localhost/hapiger"
  }'
```

*Options are passed to [knex](http://knexjs.org/).*

HapiGER also supports a [RethinkDB](http://rethinkdb.com/) event store:

```
hapiger --es rethinkdb --esoptions '{
    "host":"127.0.0.1",
    "port": 28015,
    "db":"hapiger"
  }'
```

*Options passed to [rethinkdbdash](https://github.com/neumino/rethinkdbdash).*

HapiGER also supports a [MySQL](https://www.mysql.com/) event store:

```
hapiger --es mysql --esoptions '{
    "connection": {
      "host": "localhost",
      "port": 3306,
      "user": "root",
      "password": ""
    }
  }'
```

*Options are passed to [knex](http://knexjs.org/).*

<br/>
***

#### Compacting the Event Store

The event store needs to be regularly maintained by removing old, outdated, or superfluous events; this is called **compacting**

```
curl -X POST 'http://localhost:3456/compact' -d '{
  "namespace": "movies"
}'
```

<br/>
***

#### Namespaces

Namespaces are used to separate events for different applications or categories of things. You can create namespaces by:

```
curl -X POST 'http://localhost:3456/namespaces' -d'{
    "namespace": "newnamespace"
  }'
```

To delete a namespace (**and all its events!**):

```
curl -X DELETE 'http://localhost:3456/namespaces/movies'
```

<br/>
***

#### Clients

1. Node.js client [ger-client](https://www.npmjs.com/package/ger-client)
2. HapiGER NodeJS client [hapigerjs](https://www.npmjs.com/package/hapigerjs)


#### ESM
1. [RethinkDB ESM](https://github.com/grahamjenson/ger_rethinkdb_esm)


#### Test Coverage
* [Code Coverage Report][coverage-url]
* Run it with: `npm run coverage`

## Changelog

* 5/02/17 -- Updated readme, bump versions, lock npm dependencies, added test coverage section, added ESM section
* 12/10/15 -- Updated README, new version of GER
* 8/02/15 -- Updated readme and bumped version


[coverage-url]: https://github.com/grahamjenson/hapiger/master/coverage/lcov-report/index.html