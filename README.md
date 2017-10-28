# xxhash-wasm

[![Build Status][travis-badge]][travis]
[![npm][npm-badge]][npm-link]

A WebAssembly implementation of [xxHash][xxhash].

## Installation


### From npm

```sh
npm install --save xxhash-wasm
```

Or with Yarn:

```sh
yarn add xxhash-wasm
```

### From [Unpkg][unpkg]

#### ES Modules

```html
<script type="module">
  import xxhash from "https://unpkg.com/xxhash-wasm/esm/xxhash-wasm.js";
</script>
```

#### UMD build

```html
<script src="https://unpkg.com/xxhash-wasm/umd/xxhash-wasm.js"></script>
```

The global `xxhash` will be available.

## Usage

The WebAssembly is contained in the JavaScript bundle, so you don't need to
manually fetch it and create a new WebAssembly instance.

```javascript
import xxhash from "xxhash-wasm";

// Creates the WebAssembly instance.
xxhash().then(hasher => {
  const input = "The string that is being hashed";
  // 32-bit version
  hasher.h32(input); // ee563564
  // 64-bit version
  hasher.h64(input); // 502b0c5fc4a5704c
});
```

Or with `async`/`await` and destructuring:

```javascript
// Creates the WebAssembly instance.
const { h32, h64 } = await xxhash();

const input = "The string that is being hashed";
// 32-bit version
h32(input); // ee563564
// 64-bit version
h64(input); // 502b0c5fc4a5704c
```

### Node

This was initially meant for the browser, but Node 8 also added support for
WebAssembly, so it can be run in Node as well. The implementation uses
the browser API [`TextEncoder`][textencoder-mdn], which is has been added
recently to Node as [`util.TextEncoder`][textencoder-node], but it is not
a global. To compensate for that, a CommonJS bundle is created which
automatically imports `util.TextEncoder`.

*Note: You will see a warning that it's experimental, but it should work just
fine.*

The `main` field in `package.json` points to the CommonJS bundle, so you can
require it as usual.

```javascript
const xxhash = require("xxhash-wasm");

// Or explicitly use the cjs bundle
const xxhash = require("xxhash-wasm/cjs/xxhash-wasm");
```

If you want to bundle your application for Node with a module bundler that uses
the `module` field in `package.json`, such as webpack or Rollup, you will need
to explicitly import `xxhash-wasm/cjs/xxhash-wasm` otherwise the browser version
is used.

## API

`const { h32, h64 } = await xxhash()`

Create a WebAssembly instance.

`h32(input: string, [seed: u32]): string`

Generate a 32-bit hash of `input`. The optional `seed` is a `u32` and any number
greater than the maximum (`0xffffffff`) is wrapped, which means that
`0xffffffff + 1 = 0`.

Returns a string of the hash in hexadecimal.

`h64(input: string, [seedHigh: u32, seedLow: u32]): string`

Generate a 64-bit hash of `input`. Because JavaScript doesn't support `u64` the
seed is split into two `u32`, where `seedHigh` represents the first 32-bits of
the `u64` and `seedLow` the remaining 32-bits. For example:

```javascript
// Hex
seed64:   ffffffff22222222
seedhigh: ffffffff
seedLow:          22222222

// Binary
seed64:   1111111111111111111111111111111100100010001000100010001000100010
seedhigh: 11111111111111111111111111111111
seedLow:                                  00100010001000100010001000100010
```

Each individual part of the seed is a `u32` and they are also wrapped
individually for numbers greater than the maximum.

Returns a string of the hash in hexadecimal.

## Comparison to [xxhashjs][xxhashjs]

[`xxhashjs`][xxhashjs] is implemented in pure JavaScript and because JavaScript
is lacking support for 64-bit integers, it uses a workaround with
[`cuint`][cuint]. Not only is that a big performance hit, but it also increases
the bundle size by quite a bit when it's used in the browser.

This library (`xxhash-wasm`) has the big advantage that WebAssembly supports
`u64` and also some instructions (e.g. `rotl`), which would otherwise have
to be emulated. However, The downside is that you have to initialise
a WebAssembly instance, which takes a little over 2 seconds. But once the
instance is created, it can be used without any further overhead. For the
benchmarks below, the instantiation is done before the benchmark and therefore
it's excluded from the results, since it wouldn't make sense to always create
a new WebAssembly instance.

There is still the problem that JavaScript can't represent 64-bit integers and
both the seed and the result of the 64-bit algorithm are `u64`. To work around
this, the seed and the result are split into two `u32`, which are assembled and
disassembled into/from a `u64`. Splitting the seed into two `u32` isn't a big
deal, but the result is more problematic because to assemble the 64-bit hash in
JavaScript, 3 strings have to be created: The hex representation of the first
`u32` and the hex representation of the second `u32`, which are then
concatenated to a 64-bit hex representation. That are 2 additional strings and
this is a notable overhead when the input is small.

### Benchmarks

Benchmarks are using [Benchmark.js][benchmarkjs] with random strings of
different lengths. *Higher is better*

| String length             | xxhashjs 32-bit    | xxhashjs 64-bit    | xxhash-wasm 32-bit        | xxhash-wasm 64-bit     |
| ------------------------: | ------------------ | ------------------ | ------------------------- | ---------------------- |
| 1 byte                    | 683,014 ops/sec    | 12,048 ops/sec     | ***1,414,409 ops/sec***   | 899,967 ops/sec        |
| 10 bytes                  | 577,761 ops/sec    | 12,073 ops/sec     | ***1,336,927 ops/sec***   | 902,045 ops/sec        |
| 100 bytes                 | 379,348 ops/sec    | 10,242 ops/sec     | ***1,051,487 ops/sec***   | 748,235 ops/sec        |
| 1,000 bytes               | 88,732 ops/sec     | 7,755 ops/sec      | ***451,814 ops/sec***     | 444,694 ops/sec        |
| 10,000 bytes              | 11,754 ops/sec     | 1,694 ops/sec      | 66,484 ops/sec            | ***80,590 ops/sec***   |
| 100,000 bytes             | 721 ops/sec        | 174 ops/sec        | 6,915 ops/sec             | ***8,935 ops/sec***    |
| 1,000,000 bytes           | 55,38 ops/sec      | 15,98 ops/sec      | 757 ops/sec               | ***972 ops/sec***      |
| 10,000,000 bytes          | 5.98 ops/sec       | 1.77 ops/sec       | 71.19 ops/sec             | ***92.02 ops/sec***    |
| 100,000,000 bytes         | 0.63 ops/sec*      | 0.19 ops/sec*      | 7.23 ops/sec              | ***9.26 ops/sec***     |

`*` = Runs out of memory with the default heap size.

`xxhash-wasm` outperforms `xxhashjs` significantly, the 32-bit is up to 11 times
faster (increases as the size of the input grows), and the 64-bit is up to 74
times faster (decreases as the size of the input grows).

The 64-bit version is the faster algorithm, but it only becomes faster at about
1kB because of the above mentioned limitations of JavaScript numbers. For
`xxhashjs` the 64-bit is strictly worse.

### Bundle size

Both libraries can be used in the browser and they provide a UMD bundle. The
bundles are self-contained, that means they can be included and used without
having to add any other dependencies. The table shows the bundle size of the
minified versions. *Lower is better*.

|                | xxhashjs   | xxhash-wasm   |
| -------------- | ---------- | ------------- |
| Bundle size    | 41.5kB     | ***3.9kB***   |
| Gzipped Size   | 10.3kB     | ***1.3kB***   |

[benchmarkjs]: https://benchmarkjs.com/
[cuint]: https://github.com/pierrec/js-cuint
[npm-badge]: https://img.shields.io/npm/v/xxhash-wasm.svg?style=flat-square
[npm-link]: https://www.npmjs.com/package/xxhash-wasm
[textencoder-mdn]: https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder/TextEncoder
[textencoder-node]: https://nodejs.org/api/util.html#util_class_util_textencoder
[travis]: https://travis-ci.org/jungomi/xxhash-wasm
[travis-badge]: https://img.shields.io/travis/jungomi/xxhash-wasm/master.svg?style=flat-square
[unpkg]: https://unpkg.com/
[xxhash]: https://github.com/Cyan4973/xxHash
[xxhashjs]: https://github.com/pierrec/js-xxhash
