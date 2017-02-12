# typeson.js

Preserves types over JSON, BSON or socket.io.

*typeson.js is tiny. 2.6 kb minified. ~1 kb gzipped.*

## Example of how a stringified object can look

```js
{foo: "bar"}                    // {"foo":"bar"} (simple types gives plain JSON)
{foo: new Date()}               // {"foo":1464049031538, "$types":{"foo":"Date"}}
{foo: new Set([new Date()])}    // {"foo":[1464127925971], "$types":{"foo":"Set","foo.0":"Date"}}
{foo: {sub: /bar/i}}            // {"foo":{"sub":{"source":"bar","flags":"i"}}, "$types":{"foo.sub":"RegExp"}}
{foo: new Int8Array(3)}         // {"foo":"AAAA", "$types":{"foo":"Int8Array"}}
new Date()                      // {"$":1464128478593, "$types":{"$":{"":"Date"}}} (special format at root)
```

## Why?

JSON can only contain strings, numbers, booleans, `null`, arrays and objects. If you want to serialize other types over HTTP, WebSocket, `postMessage()` or other channels, this module makes it possible to serialize any type over channels that normally only accept vanilla objects. Typeson adds a metadata property `$types` to the result that maps each non-trivial property to a type name. (In the case of arrays or encoded primitives, a new object will instead be created with a `$` property that can be preserved by JSON.) The type name is a reference to a registered type specification that you need to have the same on both the stringifying and the parsing side.

## Type Registry

[typeson-registry](https://github.com/dfahlander/typeson-registry) contains encapsulation rules for standard JavaScript types such as `Date`, `Error`, `ArrayBuffer`, etc. Pick the types you need, use a preset or write your own.

```js
var typeson = new Typeson().register([
    require('typeson-registry/types/date'),
    require('typeson-registry/types/set'),
    require('typeson-registry/types/regexp'),
    require('typeson-registry/types/typed-arrays')
]);
```
or if you want support for all built-in javascript classes:
```js
var typeson = new Typeson().register([
    require('typeson-registry/presets/builtin')
]);
```
The module `typeson-registry/presets/builtin` is 1.6 kb minizied and gzipped and adds support 32 builtin JavaScript types: *Date, RegExp, NaN, Infinity, -Infinity, Set, Map, ArrayBuffer, DataView, Uint8Array, Int8Array, Uint8ClampedArray, Int16Array, Uint16Array, Int32Array, Uint32Array, Float32Array, Float64Array, Error, SyntaxError, TypeError, RangeError, ReferenceError, EvalError, URIError, InternalError, Intl.Collator, Intl.DateTimeFormat, Intl.NumberFormat, Object String, Object Number and Object Boolean*.

## Compatibility

- Node
- Browser
- Worker
- ES5

## Features

- Can stringify custom and standard ES5 / ES6 classes.
- Produces standard JSON with an additional `$types` property in case it is needed (or a new object if representing a primitive or array at root).
- Resolves cyclic references, such as lists of objects where each object has a reference to the list
- You can register (almost) any type to be stringifiable (serializable) with your typeson instance.
- Output will be identical to that of `JSON.stringify()` in case your object doesnt contain special types or cyclic references.
- Type specs may encapsulate its type in other registered types. For example, `ImageData` is encapsulated as `{array: Uint8ClampedArray, width: number, height: number}`, expecting another spec to convert the `Uint8ClampedArray`. With the [builtin](https://github.com/dfahlander/typeson-registry/blob/master/presets/builtin.js) preset this means it's gonna be converted to base64, but with the [socketio](https://github.com/dfahlander/typeson-registry/blob/master/presets/socketio.js) preset, its gonna be converted to an `ArrayBuffer` that is left as-is and streamed binary over the WebSocket channel!

## Limitations

Since typeson has a synchronous API, it cannot encapsulate and revive async types such as `Blob`, `File` or `Observable`. Encapsulating an async object requires to be able to emit streamed content asynchronically. Remoting libraries could however complement typeson with a streaming channel that handles the emitting of stream content. For example, a remoting library could define a typeson rule that encapsulates an [Observable](https://github.com/zenparsing/es-observable) to an id (string or number for example), then starts subscribing to it and emitting the chunks to the peer as they arrive. The peer could revive the id to an observable that when subscribed to, will listen to the channel for chunks destinated to the encapsulated ID.

## Usage

```
npm install typeson
```

```js
// Require typeson. It's an UMD module so you could also use requirejs or plain script tags.
var Typeson = require('typeson');

var typeson = new Typeson().register({
    Date: [
        x => x instanceof Date, // test function
        d => d.getTime(), // encapsulator function
        number => new Date(number) // reviver function
    ],
    Error: [
        x => x instanceof Error, // tester
        e => ({name: e.name, message: e.message}), // encapsulator
        data => {
          // reviver
          var e = new Error (data.message);
          e.name = data.name;
          return e;
        }
    ],
    SimpleClass: SimpleClass // Default rules apply. See "register (typeSpec)"
});

function SimpleClass (foo) {
    this.foo = foo;
}

// Encapsulate to a JSON friendly format:
var jsonFriendly = typeson.encapsulate({
    date: new Date(),
    e: new Error("Oops"),
    c: new SimpleClass("bar")
});
// Stringify using good old JSON.stringify()
var json = JSON.stringify(jsonFriendly, null, 2);
/*
{
  "date": 1464049031538,
  "e": {
    "name": "Error",
    "message": "Oops"
  },
  "c": {
    "foo": "bar"
  },
  "$types": {
    "date": "Date",
    "e": "Error",
    "c": "SimpleClass"
  }
}
*/

// Parse using good old JSON.parse()
var parsed = JSON.parse(json);
// Revive back again:
var revived = typeson.revive(parsed);

```
*The above sample separates Typeson.encapsulate() from JSON.stringify(). Could also have used Typeson.stringify().*

## Environment/Format support

### Use with socket.io

Socket.io can stream `ArrayBuffer`s as real binary data. This is more efficient than encapsulating it in base64/JSON. Typeson can leave certain types, like `ArrayBuffer`, untouched, and leave the stringification / binarization part to other libs (use `Typeson.encapsulate()` and not `Typeson.stringify()`).

What socket.io doesn't do though, is preserve `Date`s, `Error`s or your custom types.

So to get the best of two worlds:

- Register preset 'typeson-registry/presets/socketio' as well as your custom types.
- Use `Typeson.encapsulate()` to generate an object ready for socket-io `emit()`
- Use `Typeson.revive()` to revive the encapsulated object at the other end.

```js
var Typeson = require('typeson'),
    presetSocketIo = require('typeson-registry/presets/socketio.js');

var TSON = new Typeson()
    .register(presetSocketIo)
    .register({
        CustomClass: [
            x => x instanceof CustomClass,
            c => {foo: c.foo, bar: c.bar},
            o => new CustomClass(o.foo, o.bar)
        ]
    });

var array = new Float64Array(65536);
array.fill(42, 0, 65536);

var data = {
    date: new Date(),
    error: new SyntaxError("Ooops!"),
    array: array,
    custom: new CustomClass("foo", "bar")
};

socket.emit('myEvent', TSON.encapsulate(data));
```

The `encapsulate()` method will not stringify but just traverse the object and return a simpler structure where certain properties are replaced with a substitute. The resulting object will also have a `$types` property containing the type metadata.

Packing it up at the other end:

```js
socket.on('myEvent', function (data) {
    var revived = TSON.revive(data);
    // Here we have a true Date, SyntaxError, Float64Array and Custom to play with.
});
```
*NOTE: Both peers must have the same types registered.*

### Use with [BSON](https://www.npmjs.com/package/bson)

The BSON format can serialize object over a binary channel. It supports just the standard JSON types plus `Date`, `Error` and optionally `Function`. You can use Typeson to encapsulate and revive other types as well with BSON as bearer. Use it the same way as shown above with socket.io.

### Use with Worker.postMessage()

Web Workers have the `onmessage` and `postMessage()` communication channel that has built-in support for transferring structures using the [structured clone algorithm](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm). It supports `Date`, `ArrayBuffer`and many other standard types, but not `Error`s or your own custom classes. To support `Error` and custom types over web worker channels, register just the types that are needed (`Error`s and your custom types), and then use `Typeson.encapsulate()` before posting a message, and `Typeson.revive()` in the `onmessage` callback.

## API

### constructor ([options])

```js
new Typeson([options]);
```

Creates an instance of Typeson, on which you may configure additional types to support, or call `encapsulate()`, `revive()`, `stringify()` or `parse()` on.

#### Arguments

##### options (optional):

```
{
    cyclic?: boolean, // Default true
}
```

###### cyclic

Whether or not to support cyclic references. Defaults to `true` unless explicitely set to `false`. If this property is `false`, the parsing algorithm becomes a little faster and in case a single object occurs on multiple properties, it will be duplicated in the output (as `JSON.stringify()` would do). If this property is `true`, several instances of same object will only occur once in the generated JSON and other references will just contain a pointer to the single reference.

#### Sample

```js
var Typeson = require('typeson');
var typeson = new Typeson()
    .register (require('typeson-registry/presets/builtin'));

var tson = typeson.stringify(complexObject);
console.log(tson);
var obj = typeson.parse(tson);

```

### Properties

#### types

A map between type identifier and type-rules. Same structure as passed to `register()`. Use this property if you want to create a new Typeson containing all types from another Typeson.

##### Sample

```js
var commonTypeson = new Typeson().register([
    require('typeson-registry/presets/builtin')
]);

var myTypeson = new Typeson().register([
    commonTypeson.types, // Derive from commonTypeson
    myOwnSpecificTypes // Add your extra types
]);
```

### Methods

#### stringify (obj, [replacer], [space])

*Arguments identical to those of JSON.stringify()*

Generates JSON based on the given `obj`. If the supplied `obj` has special types or cyclic references, the produced JSON will contain a `$types` property on the root upon which type info relies.

##### Sample
```js
var TSON = new Typeson().register(require('typeson-registry/types/date'));
TSON.stringify ({date: new Date()});
```
Output:
```js
{"date": 1463667643065, "$types": {"date": "Date"}}
```

#### parse (obj, [reviver])

*Arguments identical to those of JSON.parse()*

Parses Typeson-genereted JSON back into the original complex structure again.

##### Sample

```js
var TSON = new Typeson().register(require('typeson-registry/types/date'));
TSON.parse ('{"date": 1463667643065, "$types": {"date": "Date"}}');
```

#### encapsulate (obj)

Encapsulates an object but leaves the stringification part to you. Pass your encapsulated object further to socket.io, `postMessage()`, BSON or IndexedDB.

##### Sample

```js
var encapsulated = typeson.encapsulate(new Date());
var revived = typeson.revive(encapsulated);
assert (revived instanceof Date);
```

#### revive (obj)

Revives an encapsulated object. See `encapsulate()`.

#### register (typeSpec)

##### typeSpec

An object that maps a type-name to a specification of how to test, encapsulate and revive that type.

`{TypeName => constructor-function | [tester, encapsulator, reviver]}` or an array of such structure.

###### constructor-function

A class (constructor function) that would use default test, encapsulation and revival rules, which is:

- `test`: check if x.constructor === constructor-function.
- `encapsulate`: copy all enumerable own props into a vanilla object
- `revive`: Use `Object.create()` to revive the correct type, and copy all props into it.

###### tester (obj : any, stateObj : {ownKeys: boolean}) : boolean

Function that tests whether an instance is of your type and returns a truthy value if it is.

If the context is iteration over non-"own" integer string properties of an array (i.e.,
an absent (`undefined`) item in a sparse array), `ownKeys` will be set to `false`.
Otherwise, when iterating an object or array, it will be set to `true`. The default
for the `stateObj` is just an empty object.

If you wish to have exceptions thrown upon encountering a certain type of
value, you may leverage the tester to do so.

###### encapsulator (obj: YourType, stateObj : {ownKeys: boolean}) : Object

Function that maps your instance to a JSON-serializable object. For the `stateObj`,
see `tester`. In a property context (for arrays or objects), returning `undefined`
will prevent the addition of the property.

###### reviver (obj: Object) : YourType

Function that maps your JSON-serializable object into a real instance of your type.
In a property context (for arrays or objects), returning `undefined`
will prevent the addition of the property. To explicitly add `undefined`, see
`Typeson.Undefined`.

##### Sample

```js
var typeson = new Typeson();

function CustomType(foo) {
    this.foo = foo;
}

typeson.register({
  // simple style - provide just a constructor function.
  // This style works for any trivial js class without hidden closures.
  CustomType: CustomType,

  // Date is native and hides it's internal state.
  // We must define encapsulator and reviver that always works.
  Date: [
    x => x instanceof Date, // tester
    date => date.getTime(), // encapsulator
    obj => new Date(obj)    // reviver
  ],

  RegExp: [
    x = x instanceof RegExp,
    re => [re.source, re.flags],
    a => new RegExp (a[0], a[1])
  ]
});

console.log(typeson.stringify({
    ct: new CustomType("hello"),
    d: new Date(),
    r:/foo/gi
}));
// {"ct":{"foo":"hello"},"d":1464049031538,"r":["foo","gi"],$types:{"ct":"CustomType","d":"Date","r":"RegExp"}}
```

### `Typeson.Undefined` class

During encapsulation, `undefined` will not be set for property values,
of objects or arrays (including sparse ones and replaced values)
(`undefined` will be converted to `null` if stringified
anyways). During revival, however, since `undefined` is also used in
this context to indicate a value will not be added, if you wish to
have an explicit `undefined` added, you can return
`new Typeson.Undefined()` to ensure a value is set explicitly to
`undefined`.

This distinction is used by the `undefined` type in `typeson-registry`
to allow reconstruction of explicit `undefined` values (and its
`sparseUndefined` type will ensure that sparse arrays can be
reconstructed).

[typeson-registry](https://github.com/dfahlander/typeson-registry) contains ready-to-use types to register with your Typeson instance.
