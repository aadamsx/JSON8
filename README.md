JSON8 Patch
===========

[![build status](https://img.shields.io/travis/JSON8/patch.svg?style=flat-square)](https://travis-ci.org/JSON8/patch)

JSON Patch [RFC 6902](http://tools.ietf.org/html/rfc6902) implementation for JavaScript.

See [jsonpatch.com](http://jsonpatch.com) for more information about JSON Patch.

JSON8 Patch passes the entire [json-patch-tests](https://github.com/json-patch/json-patch-tests) suite; see [Tests](#tests)

* [Getting started](#getting-started)
* [Methods](#methods)
  * [apply](#apply)
  * [patch](#patch)
  * [revert](#revert)
  * [diff](#diff)
  * [Operations](#operations)
    * [add](#add)
    * [remove](#remove)
    * [replace](#replace)
    * [move](#move)
    * [copy](#copy)
    * [test](#test)
  * [Extra operations](#extra-operations)
    * [get](#get)
    * [has](#has)
  * [Patch size](#patch-size)
    * [pack](#pack)
    * [unpack](#unpack)
* [Tests](#tests)
* [Contributing](#contributing)

# Getting started

```npm install json8-patch```

----

```javascript
var ooPatch = require('json8-patch');
```

or

```xml
<script src="node_modules/json8-patch/JSON8Patch.js"></script>
```
```javascript
var ooPatch = window.JSON8Patch
```

For performance concerns JSON8 Patch mutates documents; if you want it to work on its own shallow copy of your document use:

```javascript

var oo = require('json8')
doc = oo.clone(doc)
```

See [clone](https://github.com/JSON8/JSON8#ooclone).

[↑](#json8-pointer)

# Methods

## apply

```javascript
doc = ooPatch.apply(doc, [
  { "op": "add",     "path": "/tags/0",        "value": "family"          },
  { "op": "remove",  "path": "/height"                                    },
  { "op": "replace", "path": "/age",           "value": "26"              },
  { "op": "move",    "from": "/address/city",  "path": "/address/country" },
  { "op": "copy",    "from": "/starred",       "to":    "/bookmarked"     },
  { "op": "test",    "path": "/starred",       "value": true              }
]);
```

```ooPatch.apply``` returns a document because the JSON Patch specification states that an operation can replace the original document.

The operation is atomic, if any operation fails, the document will be restored to its original state and an error will be thrown.

[↑](#json8-patch)

## patch

Alias for [apply](#apply) method.

[↑](#json8-patch)

## revert

If the [patch](#patch) or [apply](#apply) method is called with a third argument ```{reversible: true}``` it will return an array of the form ```[doc, revert]```.

The revert object can be used to revert a patch on a document.

```javascript
// apply the patch with the reversible option
var patchResult = ooPatch.apply(doc, patch, {reversible: true});
doc = patchResult[0];

// revert the patch
doc = ooPatch.revert(doc, patchResult[1]);
// doc is strictly identical to the original
```

[↑](#json8-patch)

## diff

Returns a diff in the form of a JSON Patch for 2 JSON values.

```javascript
ooPatch.diff(true, false)
// [{"op": "replace", "path": "", "value": "false"}]

ooPatch.diff([], [])
// []

ooPatch.diff({}, {"foo": "bar"})
// [{"op": "add", "path": "/foo", "value": "bar"}]
```

[↑](#json8-patch)

## Operations

```add```, ```copy```, ```replace```, ```move```, ```remove```, ```test``` operations return an array of the form ```[document, previous, idx]```

The first argument is the patched document.

The second argument is the previous value at the specified destination if any, undefined otherwise.

The third argument is used internaly and can be ignored. It is the index of the new element. For ```add``` operation only using the JSON Pointer '-' token to push an item at the end of an array.

### add
```javascript
doc = ooPatch.add(doc, '/foo', 'foo')[0]
```

[↑](#json8-patch)

### remove
```javascript
doc = ooPatch.remove(doc, '/foo')[0];
```

[↑](#json8-patch)

### replace
```javascript
doc = ooPatch.replace(doc, '/foo', 'foo')[0];
```

[↑](#json8-patch)

### move
```javascript
doc = ooPatch.move(doc, '/foo', '/bar')[0];
```

[↑](#json8-patch)

### copy
```javascript
doc = ooPatch.copy(doc, '/foo', '/bar')[0];
```

[↑](#json8-patch)

### test
```javascript
doc = ooPatch.test(doc, '/foo', 'bar')[0];
```

[↑](#json8-patch)

## Extra operations

Those are not part of the standard and are only provided for convenience.

### get
```javascript
ooPatch.get(doc, '/foo');
// returns value at /foo
```

[↑](#json8-patch)

### has
```javascript
ooPatch.has(doc, '/foo');
// returns true if there is a value at /foo
```

[↑](#json8-patch)

## Patch size

Per specification patches are pretty verbose. JSON8 provides [pack](#patch) and [unpack](#unpack) methods to reduce the size of patches and save memory/space/bandwidth.

Size (in bytes) comparaison for the following patch file

```json
[
  {"op": "add", "path": "/a/b/c", "value": ["foo", "bar"]},
  {"op": "remove", "path": "/a/b/c"},
  {"op": "replace", "path": "/a/b/c", "value": 42},
  {"op": "move", "from": "/a/b/c", "path": "/a/b/d"},
  {"op": "copy", "from": "/a/b/c", "path": "/a/b/e"},
  {"op": "test", "path": "/a/b/c", "value": "foo"}
]
```

|     format    | size (in bytes) |
|:-------------:|:---------------:|
| unpacked      |       313       |
| unpacked gzip |       148       |
| packed        |       151       |
| packed gzip   |        99       |

In pratice I'd recommand to use pack/unpack if

* data compression cannot be used on the transport of the patch
* keeping a large amount of patches in memory/on disk

[↑](#json8-patch)

### pack

```javascript
var patch = [
  {"op": "add", "path": "/a/b/c", "value": ["foo", "bar"]},
  {"op": "remove", "path": "/a/b/c"},
  {"op": "replace", "path": "/a/b/c", "value": 42},
  {"op": "move", "from": "/a/b/c", "path": "/a/b/d"},
  {"op": "copy", "from": "/a/b/c", "path": "/a/b/e"},
  {"op": "test", "path": "/a/b/c", "value": "foo"}
];

var packed = ooPatch.pack(patch);
```

Here is what packed looks like

```json
[
  [0, "/a/b/c", ["foo", "bar"]],
  [1, "/a/b/c"],
  [2, "/a/b/c", 42],
  [3, "/a/b/d", "/a/b/c"],
  [4, "/a/b/e", "/a/b/c"],
  [5, "/a/b/c", "foo"],
]
```

[↑](#json8-patch)

### unpack

```javascript
var patch = ooPatch.unpack(packed);
// [{...}, {...}, ...]
```

[↑](#json8-patch)

###

# Tests

```
git submodule update --init --recursive
npm install -g mocha babel browserify
npm test
```

[↑](#json8-patch)

# Contributing

See [CONTRIBUTING.md](https://github.com/JSON8/patch/blob/master/CONTRIBUTING.md)

[↑](#json8-patch)
