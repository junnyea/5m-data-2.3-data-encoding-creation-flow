# Reference

- XML Cheat Sheet
  - https://cheatography.com/nqramjets/cheat-sheets/xml-1-0/
- Thrift Guide
  - http://diwakergupta.github.io/thrift-missing-guide/
- Protocol Buffers Language Guide
  - https://protobuf.dev/programming-guides/proto3/
- Avro, Parquet and ORC Overview
  - https://www.upsolver.com/blog/the-file-format-fundamentals-of-big-data
- Avro Guide
  - https://avro.apache.org/docs/1.11.1/getting-started-python/
- Parquet Guide
  - https://parquet.apache.org/docs/
- ORC Guide
  - https://orc.apache.org/docs/
- Arrow Guide
  - https://arrow.apache.org/docs/python/getstarted.html

---

### Foundation

- [Introduction to XML](https://www.guru99.com/xml-tutorials.html)
- [Client-Server Model](https://www.geeksforgeeks.org/client-server-model/)
- [REST API](https://dev.to/thabisegoe/a-simple-introduction-to-rest-and-how-to-get-started-2580)

---

## Thrift, Protobuf, Avro Encoding Comparison

### Thrift

Let's use the following example record (JSON or dictionary-like) to encode:

```json
{
  "userName": "Martin",
  "favoriteNumber": 1337,
  "interests": ["daydreaming", "hacking"]
}
```

We can encode the record in Thrift using the following schema in the `.thrift` file:

```thrift
struct Person {
  1: required string userName,
  2: optional i64 favoriteNumber,
  3: optional list<string> interests
}
```

Thrift comes with a code generation tool that takes a schema definition like the ones shown here, and produces classes that implement the schema in various programming languages. Our code can call this generated code to encode or decode records of the schema.

The data encoded with this schema looks like this:
![thrift_binary_protocol](/assets/thrift_binary_protocol.png)

Each field has a type annotation (to indicate whether it is a string, integer, list, etc.) and, where required, a length indication (length of a string, number of items in a list). The strings that appear in the data (“Martin”, “daydreaming”, “hacking”) are encoded as UTF-8.

There are no field names (userName, favoriteNumber, interests). Instead, the encoded data contains _field tags_, which are numbers (1, 2, and 3). Those are the numbers that appear in the schema definition. Field tags are like aliases for fields—they are a compact way of saying what field we’re talking about, without having to spell out the field name.

### Protocol Buffers (Protobuf)

We can encode the previous example record in Protobuf using the following schema in the `.proto` file:

```protobuf
message Person {
  required string user_name = 1;
  optional int64 favorite_number = 2;
  repeated string interests = 3;
}
```

The data encoded with this schema looks like this:
![protobuf](/assets/protobuf.png)

Protocol Buffers have an interesting aspect regarding its datatype handling. Unlike having a specific list or array datatype, it utilizes a `repeated` marker for fields, which serves as a third option alongside `required` and `optional`.

As depicted in the figure, a repeated field is simply represented by the same field tag appearing multiple times in the record. The advantage of this approach is that converting an optional (single-valued) field into a repeated (multi-valued) field is permissible. When new code reads old data, it interprets it as a list with either zero or one element, depending on whether the field was present. On the other hand, old code reading new data only perceives the last element of the list.

### Apache Avro

We can encode the previous example record in IDL using the following schema in the `.avsc` file:

```avro
record Person {
  string userName;
  union { null, long } favoriteNumber = null;
  array<string> interests;
}
```

The equivalent JSON representation of that schema is as follows:

```json
{
  "type": "record",
  "name": "Person",
  "fields": [
    { "name": "userName", "type": "string" },
    { "name": "favoriteNumber", "type": ["null", "long"], "default": null },
    { "name": "interests", "type": { "type": "array", "items": "string" } }
  ]
}
```

The data encoded with this schema looks like this:
![avro](/assets/avro.png)

First and foremost, it's important to note that the schema lacks tag numbers. When we encode our sample record using this schema, the resulting Avro binary encoding is impressively compact, spanning just _32 bytes_—the most space-efficient among all the encodings we've observed.

Examining the byte sequence, one can readily discern the _absence of field identifiers or datatype markers_. The encoding solely comprises concatenated values. For instance, a string is represented by a length prefix followed by UTF-8 bytes, but there are no explicit indicators within the encoded data to specify that it is, indeed, a string. In fact, it could be interpreted as an integer or any other data type altogether. Similarly, an integer is encoded using a variable-length encoding.

To correctly parse the binary data, you must traverse the fields in the order they appear in the schema and _refer to the schema_ itself to ascertain the datatype of each field. Consequently, the binary data can only be accurately decoded if the code reading the data employs the exact same schema as the code that wrote the data. Any deviation or mismatch in the schema between the reader and the writer would result in incorrectly decoded data.

With Avro, data encoding and decoding are based on two schemas: the `writer's schema` used during data encoding and the `reader's schema` employed during data decoding. These schemas do not necessarily have to be identical but should be compatible. When decoding data, the Avro library compares the writer's and reader's schemas, resolving any discrepancies between them.

The Avro specification ensures that fields in different orders between the writer's and reader's schemas pose no issues during resolution since schema matching occurs based on field names. If the reader's schema lacks a field present in the writer's schema, it is simply ignored. Conversely, if the reader's schema expects a field that the writer's schema does not contain, the missing field is filled in with a default value declared in the reader's schema. This allows for flexible schema evolution while maintaining data compatibility.

