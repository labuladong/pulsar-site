---
id: client-libraries-python
title: Pulsar Python client
sidebar_label: "Python"
---

````mdx-code-block
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
````

You can use a Pulsar Python client to create producers, consumers, and readers. For Pulsar features that Python clients support, see [Client Feature Matrix](https://docs.google.com/spreadsheets/d/1YHYTkIXR8-Ql103u-IMI18TXLlGStK8uJjDsOOA0T20/edit#gid=1784579914).

## Installation

Use [pip](https://pip.pypa.io/) to install the latest version:

```bash
pip install 'pulsar-client==@pulsar:version_number@'
```

You can install optional components alongside the client library:

```bash
# avro serialization
pip install 'pulsar-client[avro]==@pulsar:version_number@'

# functions runtime
pip install 'pulsar-client[functions]==@pulsar:version_number@'

# all optional components
pip install 'pulsar-client[all]==@pulsar:version_number@'
```

Installation via PyPi is available for the following Python versions:

| Platform                       | Supported Python versions |
|:-------------------------------|:--------------------------|
| macOS (>= 11.0)                | 3.7, 3.8, 3.9 and 3.10    |
| Linux (including Alpine Linux) | 3.7, 3.8, 3.9 and 3.10    |

## Connection URLs

To connect to Pulsar using client libraries, you need to specify a [Pulsar protocol](developing-binary-protocol.md) URL.

You can assign Pulsar protocol URLs to specific clusters and use the `pulsar` scheme. The following is an example of `localhost` with the default port `6650`:

```http
pulsar://localhost:6650
```

If you have multiple brokers, separate `IP:port` by commas:

```http
pulsar://localhost:6550,localhost:6651,localhost:6652
```

If you use [TLS](security-tls-authentication.md) authentication, add `+ssl` in the scheme:

```http
pulsar+ssl://pulsar.us-west.example.com:6651
```

## API reference

All the methods in producer, consumer, and reader of Pulsar Python clients are thread-safe. See the [API docs](@pulsar:apidoc:python@) for more details.

## Release notes

For the changelog of Pulsar Python clients, see [release notes](/release-notes/#python).

## Examples

You can find a variety of Python code examples for the `pulsar-client` library.

### Producer example

The following example creates a Python producer for the `my-topic` topic and sends 10 messages on that topic:

```python
import pulsar

client = pulsar.Client('pulsar://localhost:6650')

producer = client.create_producer('my-topic')

for i in range(10):
    producer.send(('Hello-%d' % i).encode('utf-8'))

client.close()
```

### Consumer example

The following example creates a consumer with the `my-subscription` subscription name on the `my-topic` topic, receives incoming messages, prints the content and ID of messages that arrive, and acknowledges each message to the Pulsar broker.

```python
import pulsar

client = pulsar.Client('pulsar://localhost:6650')

consumer = client.subscribe('my-topic', 'my-subscription')

while True:
    msg = consumer.receive()
    try:
        print("Received message '{}' id='{}'".format(msg.data(), msg.message_id()))
        # Acknowledge successful processing of the message
        consumer.acknowledge(msg)
    except Exception:
        # Message failed to be processed
        consumer.negative_acknowledge(msg)

client.close()
```

This example shows how to configure negative acknowledgment.

```python
from pulsar import Client, schema
client = Client('pulsar://localhost:6650')
consumer = client.subscribe('negative_acks','test',schema=schema.StringSchema())
producer = client.create_producer('negative_acks',schema=schema.StringSchema())
for i in range(10):
    print('send msg "hello-%d"' % i)
    producer.send_async('hello-%d' % i, callback=None)
producer.flush()
for i in range(10):
    msg = consumer.receive()
    consumer.negative_acknowledge(msg)
    print('receive and nack msg "%s"' % msg.data())
for i in range(10):
    msg = consumer.receive()
    consumer.acknowledge(msg)
    print('receive and ack msg "%s"' % msg.data())
try:
    # No more messages expected
    msg = consumer.receive(100)
except:
    print("no more msg")
    pass
```

### Reader interface example

You can use the Pulsar Python API to use the Pulsar [reader interface](concepts-clients.md#reader-interface). Here's an example:

```python
# MessageId taken from a previously fetched message
msg_id = msg.message_id()

reader = client.create_reader('my-topic', msg_id)

while True:
    msg = reader.read_next()
    print("Received message '{}' id='{}'".format(msg.data(), msg.message_id()))
    # No acknowledgment
```

### Multi-topic subscriptions

In addition to subscribing a consumer to a single Pulsar topic, you can also subscribe to multiple topics simultaneously. To use multi-topic subscriptions, you can supply a regular expression (regex) or a `List` of topics. If you select topics via regex, all topics must be within the same Pulsar namespace.

The following is an example:

```python
import re
consumer = client.subscribe(re.compile('persistent://public/default/topic-*'), 'my-subscription')
while True:
    msg = consumer.receive()
    try:
        print("Received message '{}' id='{}'".format(msg.data(), msg.message_id()))
        # Acknowledge successful processing of the message
        consumer.acknowledge(msg)
    except Exception:
        # Message failed to be processed
        consumer.negative_acknowledge(msg)
client.close()
```

### Create a Python client with multiple advertised listeners
To ensure clients in both internal and external networks can connect to a Pulsar cluster, Pulsar introduces [advertisedListeners](concepts-multiple-advertised-listeners.md).

The following example creates a Python client using multiple advertised listeners:

```python
import pulsar

client = pulsar.Client('pulsar://localhost:6650', listener_name='external')
```

## Schema

### Supported schema types

You can use different built-in schema types in Pulsar. All the definitions are in the `pulsar.schema` package.

| Schema | Notes |
| ------ | ----- |
| `BytesSchema` | Get the raw payload as a `bytes` object. No serialization/deserialization are performed. This is the default schema mode |
| `StringSchema` | Encode/decode payload as a UTF-8 string. Uses `str` objects |
| `JsonSchema` | Require record definition. Serializes the record into standard JSON payload |
| `AvroSchema` | Require record definition. Serializes in AVRO format |

### Schema definition reference

The schema definition is done through a class that inherits from `pulsar.schema.Record`.

This class has a number of fields that can be of either `pulsar.schema.Field` type or another nested `Record`. All the fields are specified in the `pulsar.schema` package. The fields are matching the AVRO field types.

| Field Type | Python Type | Notes |
| ---------- | ----------- | ----- |
| `Boolean`  | `bool`      |       |
| `Integer`  | `int`       |       |
| `Long`     | `int`       |       |
| `Float`    | `float`     |       |
| `Double`   | `float`     |       |
| `Bytes`    | `bytes`     |       |
| `String`   | `str`       |       |
| `Array`    | `list`      | Need to specify record type for items. |
| `Map`      | `dict`      | Key is always `String`. Need to specify value type. |

Additionally, any Python `Enum` type can be used as a valid field type.

#### Fields parameters

When adding a field, you can use these parameters in the constructor.

| Argument   | Default | Notes |
| ---------- | --------| ----- |
| `default`  | `None`  | Set a default value for the field, such as `a = Integer(default=5)`. |
| `required` | `False` | Mark the field as "required". It is set in the schema accordingly. |

#### Schema definition examples

##### Simple definition

```python
class Example(Record):
    a = String()
    b = Integer()
    c = Array(String())
    i = Map(String())
```

##### Using enums

```python
from enum import Enum

class Color(Enum):
    red = 1
    green = 2
    blue = 3

class Example(Record):
    name = String()
    color = Color
```

##### Complex types

```python
class MySubRecord(Record):
    x = Integer()
    y = Long()
    z = String()

class Example(Record):
    a = String()
    sub = MySubRecord()
```

##### Set namespace for Avro schema

Set the namespace for the Avro Record schema using the special field `_avro_namespace`.

```python
class NamespaceDemo(Record):
   _avro_namespace = 'xxx.xxx.xxx'
   x = String()
   y = Integer()
```

The schema definition is like this.

```json
{
  "name": "NamespaceDemo", "namespace": "xxx.xxx.xxx", "type": "record", "fields": [
    {"name": "x", "type": ["null", "string"]},
    {"name": "y", "type": ["null", "int"]}
  ]
}
```

### Declare and validate schema

Before the producer is created, the Pulsar broker validates that the existing topic schema is the correct type and that the format is compatible with the schema definition of a class. If the format of the topic schema is incompatible with the schema definition, an exception occurs in the producer creation.

Once a producer is created with a certain schema definition, it only accepts objects that are instances of the declared schema class.

Similarly, for a consumer or reader, the consumer returns an object (which is an instance of the schema record class) rather than raw bytes.

**Example**

```python
consumer = client.subscribe(
                  topic='my-topic',
                  subscription_name='my-subscription',
                  schema=AvroSchema(Example) )

while True:
    msg = consumer.receive()
    ex = msg.value()
    try:
        print("Received message a={} b={} c={}".format(ex.a, ex.b, ex.c))
        # Acknowledge successful processing of the message
        consumer.acknowledge(msg)
    except Exception:
        # Message failed to be processed
        consumer.negative_acknowledge(msg)
```

For more code examples, see [Schema - Get started](schema-get-started.md).

## End-to-end encryption

Pulsar encryption allows applications to encrypt messages at producers and decrypt messages at consumers. See [Get started](security-encryption.md#get-started) for more details.