![Firefly](./assets/logo.png)

A fast key-value pair in memory database. With a very simple and fast API. At
[Xiler](https://www.xiler.net) it gets used to store and manage client sessions
throughout the platform.

![CLI help menu](https://user-images.githubusercontent.com/38541241/184537875-5fcbdfd3-3da8-429e-ab34-f755e5ee3192.png)

## Installation

**Cargo:**
`$ cargo install ffly`

**AUR:**
`$ paru -S ffly`

**Docker image:**
[arthurdw/firefly](https://hub.docker.com/repository/docker/arthurdw/firefly)

## Performance comparison

| Database                                         | ops  |
| ------------------------------------------------ | ---- |
| Firefly                                          | 167k |
| [Skytable](https://github.com/skytable/skytable) | 143k |
| [Redis](https://github.com/redis/redis)          | 67k  |

_(`push_it` scripts can be found in `ffly-rs/examples/`)_

## Query Language

Firefly only has three data operators, `NEW`, `GET`, `DROP`. We'll walk over
each one.

### Important note on values

ALL values must be valid ASCII, if not the server will reject the query.

### Defining query types

The server may only accept one query type per TCP connection type. The default
type is a string query. But this can be changed by using the `QUERY TYPE`
keyword.

```ffly
QUERY TYPE 'STRING' | 'BITWISE';
```

### String queries

The simplest way to query something is by querying it using a string query. But
because the parsing, and bandwidth of this is more than that is necessary we
also provide bitwise queries.

String queries are very loosely defined, as it evaluates the query definition
from left to right. And can derive that if you use e.g. `GETV`, you mean
`GET VALUE`. This works for any query, identifiers are case-insensitive.

#### Create

You can create a new record by using the `NEW` keyword, the arguments should be
wrapped within quotes. The first argument _(after `NEW`)_ is the key, this
should be unique throughout the db. If no TLL value is provided, the server
will use 0 _(aka no expiry)_.

```ffly
NEW '{key}'
[ VALUE ] '{value}'
[ WITH TTL '{ttl}'];
```

##### Create examples

```ffly
NEW 's2.8eqYursP2McHeQvHB2bauyE6n3vptOj8M96PxmGAQMDfimeZ31WAzP3hSw5Ixv5Y'
VALUE '86ebe1a0-11bf-11ed-aa8e-13602e2ad46b';
```

```ffly
NEW 's2.8eqYursP2McHeQvHB2bauyE6n3vptOj8M96PxmGAQMDfimeZ31WAzP3hSw5Ixv5Y'
VALUE '86ebe1a0-11bf-11ed-aa8e-13602e2ad46b'
WITH TTL '604800';
```

#### Fetch

The `GET` keyword returns the value and TTL by default. But if you only want
one of the two, you can specify this. You can only search by key!

```ffly
GET [VALUE | TTL] '{key}';
```

##### Fetch examples

```ffly
GET 's2.8eqYursP2McHeQvHB2bauyE6n3vptOj8M96PxmGAQMDfimeZ31WAzP3hSw5Ixv5Y';
```

```ffly
GET TTL 's2.8eqYursP2McHeQvHB2bauyE6n3vptOj8M96PxmGAQMDfimeZ31WAzP3hSw5Ixv5Y';
```

#### Delete

Deleting one record is as straightforward as fetching one. You can only delete
whole records. If required all records that have a value can also be deleted,
but this action is very expensive and generally not recommended.

```ffly
DROP '{key}';
DROP ALL '{value}';
```

### Bitwise queries

Because string queries can consume more resources than what is required, there
is a more efficient _(less friendly)_ way to interact with Firefly. This is by
sending the bits in a specific format. This section just describes the formats,
if you want more information about the queries itself, then you can find it in
the `String queries` section.

Please keep in mind that all the bitwise queries are very strict, and if a
query is unrecognized it will discard it.

#### General notes

-   All values are delimited by a NUL character. _(`0x0`)_
-   The end of the query is assumed to be the last byte.
-   Queries start with their type, this is a numeric value
    -   0: `NEW`
    -   1: `GET`
    -   2: `GET VALUE`
    -   3: `GET TTL`
    -   4: `DROP`
    -   5: `DROP ALL`
    -   6: `QUERY TYPE STRING`
    -   7: `QUERY TYPE BITWISE`
-   The query type does not need to be delimited

#### Bitwise create

A with TTL must always be provided. If you don't want a TTL set this to 0.

`0{key}0x0{value}0x0{ttl}`

These are the two same create examples from the string queries:
`0s2.8eqYursP2McHeQvHB2bauyE6n3vptOj8M96PxmGAQMDfimeZ31WAzP3hSw5Ixv5Y0x086ebe1a0-11bf-11ed-aa8e-13602e2ad46b0x0`
`0s2.8eqYursP2McHeQvHB2bauyE6n3vptOj8M96PxmGAQMDfimeZ31WAzP3hSw5Ixv5Y0x086ebe1a0-11bf-11ed-aa8e-13602e2ad46b0x0602800`

#### Bitwise fetch

`1s2.8eqYursP2McHeQvHB2bauyE6n3vptOj8M96PxmGAQMDfimeZ31WAzP3hSw5Ixv5Y`
`2s2.8eqYursP2McHeQvHB2bauyE6n3vptOj8M96PxmGAQMDfimeZ31WAzP3hSw5Ixv5Y`
`3s2.8eqYursP2McHeQvHB2bauyE6n3vptOj8M96PxmGAQMDfimeZ31WAzP3hSw5Ixv5Y`

#### Bitwise delete

`4s2.8eqYursP2McHeQvHB2bauyE6n3vptOj8M96PxmGAQMDfimeZ31WAzP3hSw5Ixv5Y`
`5s2.8eqYursP2McHeQvHB2bauyE6n3vptOj8M96PxmGAQMDfimeZ31WAzP3hSw5Ixv5Y`
