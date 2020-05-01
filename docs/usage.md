---
id: usage
title: Usage
sidebar_label: Usage
---

Typical usage consists of [uploading one or more schemas](#uploading-schemas) to the registry, [encoding
data](#encoding-data) using the registered schemas, and/or [decoding encoded data](#decoding-data) by getting
the schemas from the registry.

## Creating the registry client

```js
const { SchemaRegistry } = require('@kafkajs/confluent-schema-registry')

const registry = new SchemaRegistry({ host: 'http://localhost:8081' })
```

For more configuration options, [see configuration](#configuration).

## Uploading schemas

Schemas can be defined in either `AVSC` or `AVDL` format, and are read using
`readAVSCAsync` and `avdlToAVSCAsync` respectively.

Once read, the schemas can be registered with the schema registry using
`registry.register(schema)`, which resolves to an object containing the
schema id. This schema id is later [used when encoding](#encoding-data).

```js
const { readAVSCAsync, avdlToAVSCAsync } = require('@kafkajs/confluent-schema-registry')

// From an avsc file
const schema = await readAVSCAsync('path/to/schema.avsc')
const { id } = await registry.register(schema) // { id: 2 }

// From an avdl file
const schema = await avdlToAVSCAsync('path/to/protocol.avdl')
const { id } = await registry.register(schema) // { id: 3 }
```

### Compatibility

The [compatibility](https://docs.confluent.io/current/schema-registry/avro.html#compatibility-types) of the schema will be whatever the global default is (typically `BACKWARD`).
It's possible to override this for the specific subject by setting it like so:

```js
const {
  COMPATIBILITY: { NONE },
} = require('@kafkajs/confluent-schema-registry')
await registry.register(schema, { compatibility: NONE })
```

**NOTE:**
If the subject already has an overridden compatibility setting and it's different,
the client will throw and error (`ConfluentSchemaRegistryCompatibilityError`)

### Overriding subject

Each schema is registered under a [subject](https://docs.confluent.io/current/schema-registry/serializer-formatter.html#sr-avro-subject-name-strategy).
By default, this subject is generated by concatenating the schema namespace and the schema name
with a separator. For example, the following schema would get the subject `com.example.Simple`:

```avdl
@namespace("com.example")
protocol SimpleProto {
  record Simple {
    string foo;
  }
}
```

`registry.register` accepts a `subject` option to override the subject entirely:

```js
await registry.register(schema, { subject: 'my-fixed-subject' })
```

If you just want to change the separator used when automatically creating the subject, use
the `separator` option:

```js
// This would result in "com.example-Simple"
await registry.register(schema, { separator: '-' })
```

## Encoding data

To encode data, call `registry.encode` with the schema id and the payload to encode.

```js
const payload = { full_name: 'John Doe' }
await registry.encode(id, payload)
```

## Decoding data

The encoded payload contains the schema id of the schema used to decode it,
so to decode, simply call `registry.decode` with the encoded payload. The
corresponding schema will be downloaded from the registry if needed in order
to decode the payload.

```js
const payload = await registry.decode(buffer)
// { full_name: 'John Doe' }
```

## Configuration

### Retry

By default, all `GET` requests will retry three times in case of failure. If you want to tweak this config you can do:

```js
const registry = new SchemaRegistry({
  host: 'http://localhost:8081',
  retry: {
    maxRetryTimeInSecs: 5,
    initialRetryTimeInSecs: 0.1,
    factor: 0.2, // randomization factor
    multiplier: 2, // exponential factor
    retries: 3, // max retries
  },
})
```

### Basic auth

It's also possible to configure basic auth:

```js
const registry = new SchemaRegistry({
  host: 'http://localhost:8081',
  auth: {
    username: '***',
    password: '***',
  },
})
```