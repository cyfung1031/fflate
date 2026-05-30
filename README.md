# fflate

High performance compression and decompression in an 8kB package.

`fflate` (short for "fast flate") is a fast, small, pure JavaScript library for DEFLATE, GZIP, Zlib, and ZIP data. It works in browsers, Node.js, Deno, and other JavaScript runtimes. Data compressed with `fflate` can be decompressed by other standard tools, and data produced by other standard tools can be decompressed with `fflate`.

If you are new to `fflate`, start with [What should I use?](#what-should-i-use) and copy the closest example.

## Why fflate?

`fflate` is designed for projects that care about speed, bundle size, and broad format support.

| Feature | `pako` | `tiny-inflate` | `UZIP.js` | `fflate` |
| --- | --- | --- | --- | --- |
| Decompression | ✅ | ✅ | ✅ | ✅ |
| Compression | ✅ | ❌ | ✅ | ✅ |
| ZIP archives | ❌ | ❌ | ✅ | ✅ |
| GZIP | ✅ | ❌ | ❌ | ✅ |
| Zlib | ✅ | ❌ | ❌ | ✅ |
| Streaming APIs | ✅ | ❌ | ❌ | ✅ |
| Async / worker APIs | ❌ | ❌ | ❌ | ✅ |
| Streaming ZIP APIs | ❌ | ❌ | ❌ | ✅ |
| Files up to 4GB | ✅ | ❌ | ❌ | ✅ |
| Recovers cleanly from errors | ✅ | ❌ | ❌ | ✅ |
| Dictionary support | ✅ | ❌ | ❌ | ✅ |
| ES modules | ❌ | ❌ | ❌ | ✅ |

`fflate` is often faster than `pako`, smaller than most alternatives, and can be tree-shaken so you only ship the features you use.

In benchmarks, `fflate` decompresses up to 25% faster than `pako` and `UZIP.js`, and compresses up to 50% faster than `pako`. The core build is about 8kB minified (3kB for decompression only), compared with roughly 45.6kB for `pako` and 14.2kB for `UZIP.js`. Compression ratios are often better than the original Zlib C library, and in synchronous mode the compressor can beat native tools such as Info-ZIP on both speed and ratio. In asynchronous mode, `fflate` can use multiple threads to reach over 3x the throughput of most other utilities.

## Install

```sh
npm i fflate # or yarn add fflate, or pnpm add fflate
```

Import only what you need:

```js
import { zipSync, unzipSync, strToU8, strFromU8 } from 'fflate'
```

For CommonJS:

```js
const fflate = require('fflate')
```

Avoid importing the whole library in browser bundles unless you need most of it:

```js
import * as fflate from 'fflate'
```

## What should I use?

| I want to... | Use this |
| --- | --- |
| Create a `.zip` archive | `zipSync()` or `zip()` |
| Read a `.zip` archive | `unzipSync()` or `unzip()` |
| Create a `.gz` file | `gzipSync()` or `gzip()` |
| Read a `.gz` file | `gunzipSync()` or `gunzip()` |
| Create Zlib data | `zlibSync()` or `zlib()` |
| Read Zlib data | `unzlibSync()` or `unzlib()` |
| Create raw DEFLATE data | `deflateSync()` or `deflate()` |
| Read raw DEFLATE data | `inflateSync()` or `inflate()` |
| Compress one buffer with a sensible default | `compressSync()` or `compress()` |
| Decompress data that may be GZIP, Zlib, or DEFLATE | `decompressSync()` or `decompress()` |
| Compress or decompress data piece by piece | `Gzip`, `Gunzip`, `Deflate`, `Inflate`, `Zip`, or `Unzip` |
| Convert strings to bytes | `strToU8()` |
| Convert bytes to strings | `strFromU8()` |

### Sync, async, or streaming?

| Situation | Recommendation |
| --- | --- |
| Small inputs, scripts, tests, build tools, or Node.js CLI tools | Use sync APIs |
| Browser UI where blocking the main thread matters | Use async APIs |
| Large archives or many large files | Use async ZIP APIs |
| Data arrives or leaves in chunks | Use streaming classes |

Async APIs use workers where possible. They avoid blocking the main thread, but worker startup has overhead, so sync APIs are usually better for small inputs. The overhead is roughly 50ms per asynchronous function: it applies only on the first call of a given function, but each distinct async function (for example `unzip` and `zlib`) pays its own startup cost. For payloads under about 50kB the async APIs are usually slower; for larger or multiple files they are dramatically better.

## Basic usage

Compression APIs work with `Uint8Array`. Node.js `Buffer`s work because they are `Uint8Array`s.

Use `strToU8()` before compressing text:

```js
import { gzipSync, strToU8 } from 'fflate'

const compressed = gzipSync(strToU8('Hello world!'))
```

Use `strFromU8()` after decompressing text:

```js
import { gunzipSync, strFromU8 } from 'fflate'

const text = strFromU8(gunzipSync(compressed))
console.log(text)
```

If the input might be GZIP, Zlib, or raw DEFLATE, use `decompressSync()`:

```js
import { decompressSync, strFromU8 } from 'fflate'
import { readFileSync } from 'node:fs'

const compressed = readFileSync('./data.bin')
const data = decompressSync(compressed)

console.log(strFromU8(data))
```

`compressSync()` is the matching high-level compressor. It produces GZIP-wrapped data by default:

```js
import { compressSync, strToU8 } from 'fflate'

const compressed = compressSync(strToU8('Hello world!'), {
  level: 6,
  mem: 8
})
```

`level` ranges from `0` for no compression to `9` for maximum compression, and defaults to `6`. `mem` ranges from `0` to `12` and defaults to `4`; higher values may improve speed at the cost of memory.

## ZIP archives

ZIP archives can contain many files and directories.

### Create a ZIP

```js
import { zipSync, strToU8 } from 'fflate'
import { readFileSync, writeFileSync } from 'node:fs'

const zipData = zipSync({
  'hello.txt': strToU8('Hello world!'),
  'images/photo.png': readFileSync('./photo.png')
})

writeFileSync('./archive.zip', zipData)
```

Object keys are paths inside the ZIP archive. Values are `Uint8Array`s.

You can also use nested objects for directories, and filenames may contain Unicode:

```js
const zipData = zipSync({
  docs: {
    'hello.txt': strToU8('Hello world!'),
    '你好.txt': strToU8('Hey there!')
  }
})
```

### ZIP compression options

By default, files in a ZIP are DEFLATE-compressed. That is often wasteful for data that is already compressed, such as PNG, JPEG, PDF, `.gz`, and many media formats. Use `level: 0` to store those files without recompressing them.

Pass options per file as a `[data, options]` tuple, or globally as the second argument. File-specific options take precedence over the global defaults:

```js
import { zipSync, strToU8 } from 'fflate'
import { readFileSync } from 'node:fs'

const zipData = zipSync({
  'notes.txt': strToU8('lots of compressible text...'),
  'photo.png': [readFileSync('./photo.png'), { level: 0 }]
}, {
  level: 6,
  mtime: new Date('1980-01-01')
})
```

Some ZIP-specific options are useful for executable files:

```js
import { zipSync, strToU8 } from 'fflate'

const zipData = zipSync({
  'hello.sh': [strToU8('echo hello world'), {
    os: 3,
    attrs: 0o755 << 16
  }]
})
```

Directories accept options too, which apply to the files they contain:

```js
const zipData = zipSync({
  exec: [{
    'hello.sh': [strToU8('echo hello world'), { os: 3, attrs: 0o755 << 16 }]
  }, {
    mtime: new Date('2020-10-20')
  }]
})
```

### Read a ZIP

```js
import { unzipSync, strFromU8 } from 'fflate'
import { readFileSync, writeFileSync } from 'node:fs'

const files = unzipSync(readFileSync('./archive.zip'))

console.log(strFromU8(files['hello.txt']))
writeFileSync('./photo.png', files['images/photo.png'])
```

The result is an object where each key is a path in the ZIP archive and each value is a `Uint8Array`. Directory structures are returned as full paths, not nested objects (for example, `{ 'nested/directory/structure.txt': Uint8Array }`).

### Skip files while reading a ZIP

Use `filter` when you only need some files. Filtering avoids the work of decompressing files you do not want:

```js
import { unzipSync, strFromU8 } from 'fflate'
import { readFileSync } from 'node:fs'

const files = unzipSync(readFileSync('./archive.zip'), {
  filter(file) {
    return file.name.endsWith('.txt') && file.originalSize <= 10_000_000
  }
})

for (const [name, data] of Object.entries(files)) {
  console.log(name, strFromU8(data))
}
```

### Create a ZIP asynchronously

The async ZIP APIs run in parallel across multiple threads, so they are much faster (up to 3x) for larger archives. The effect is most significant for multiple large files and less so for many small ones.

```js
import { zip, strToU8 } from 'fflate'
import { readFileSync, writeFileSync } from 'node:fs'

const files = {
  'hello.txt': strToU8('Hello world!'),
  'images/photo.png': readFileSync('./photo.png')
}

const zipData = await new Promise((resolve, reject) => {
  zip(files, (err, data) => {
    if (err) reject(err)
    else resolve(data)
  })
})

writeFileSync('./archive.zip', zipData)
```

### Read a ZIP asynchronously

```js
import { unzip, strFromU8 } from 'fflate'
import { readFileSync } from 'node:fs'

const zipData = readFileSync('./archive.zip')

const files = await new Promise((resolve, reject) => {
  unzip(zipData, (err, data) => {
    if (err) reject(err)
    else resolve(data)
  })
})

console.log(strFromU8(files['hello.txt']))
```

`unzip` is parallelized, so it is often much faster than `unzipSync`. It is the only async function that does not support the `consume` option described below.

### Cancel an async operation

Every async function returns a termination function. Call it to cancel the work. The callback will not be called.

```js
import { unzip } from 'fflate'

const terminate = unzip(zipData, (err, files) => {
  if (err) throw err
  console.log(files)
})

terminate()
```

## GZIP, Zlib, and DEFLATE

GZIP, Zlib, and raw DEFLATE compress one stream of data. They do not store multiple files or directory structures.

### GZIP

```js
import { gzipSync, gunzipSync, strToU8, strFromU8 } from 'fflate'

const compressed = gzipSync(strToU8('Hello world!'), {
  filename: 'hello.txt',
  mtime: new Date()
})

const text = strFromU8(gunzipSync(compressed))
console.log(text)
```

`mtime` can be a `Date`, a date string, or a Unix timestamp.

### Zlib

```js
import { zlibSync, unzlibSync, strToU8, strFromU8 } from 'fflate'

const compressed = zlibSync(strToU8('Hello world!'))
const text = strFromU8(unzlibSync(compressed))
```

### Raw DEFLATE

```js
import { deflateSync, inflateSync, strToU8, strFromU8 } from 'fflate'

const compressed = deflateSync(strToU8('Hello world!'))
const text = strFromU8(inflateSync(compressed))
```

### `.tar.gz`

A `.tar.gz` file is a TAR archive compressed with GZIP. `fflate` can decompress the GZIP layer:

```js
import { gunzipSync } from 'fflate'
import { readFileSync } from 'node:fs'

const tarData = gunzipSync(readFileSync('./archive.tar.gz'))
```

The result is still TAR data. Use a TAR parser to list or extract files from it.

## Streaming

Use streaming APIs when data arrives in chunks or when you do not want to keep everything in memory at once.

Synchronous streams such as `Gzip`, `Gunzip`, `Deflate`, and `Inflate` call handlers with `(chunk, final)` and throw errors from `push()`. ZIP streams and async streams call handlers with `(err, chunk, final)`. You can pass the data handler in the constructor or attach it later as `stream.ondata`.

### Streaming GZIP compression

```js
import { Gzip } from 'fflate'

const chunks = []

const gzip = new Gzip((chunk, final) => {
  chunks.push(chunk)

  if (final) {
    console.log('gzip complete')
  }
})

gzip.push(chunk1)
gzip.push(chunk2)
gzip.push(lastChunk, true)
```

Pass `true` to `push()` only for the last chunk.

### Streaming GZIP decompression

```js
import { Gunzip } from 'fflate'

const chunks = []

const gunzip = new Gunzip((chunk, final) => {
  chunks.push(chunk)

  if (final) {
    console.log('gunzip complete')
  }
})

gunzip.push(chunk1)
gunzip.push(chunk2)
gunzip.push(lastChunk, true)
```

### Streaming unknown compressed data

`Decompress` auto-detects GZIP, Zlib, or raw DEFLATE data.

```js
import { Decompress } from 'fflate'

const chunks = []

const decompressor = new Decompress((chunk, final) => {
  chunks.push(chunk)

  if (final) {
    console.log('decompression complete')
  }
})

decompressor.push(chunk1)
decompressor.push(lastChunk, true)
```

### Streaming text

Use `EncodeUTF8` and `DecodeUTF8` when text itself arrives in chunks. Chaining streams together is done by pushing to the next stream from inside the handler of the previous one.

```js
import { EncodeUTF8, DecodeUTF8, Gzip, Gunzip } from 'fflate'

const gzip = new Gzip((chunk, final) => {
  gunzip.push(chunk, final)
})

const gunzip = new Gunzip((chunk, final) => {
  decoder.push(chunk, final)
})

const decoder = new DecodeUTF8((text, final) => {
  console.log(text)

  if (final) {
    console.log('done')
  }
})

const encoder = new EncodeUTF8((data, final) => {
  gzip.push(data, final)
})

encoder.push('Hello ')
encoder.push('world!', true)
```

### Streaming ZIP creation

```js
import { Zip, ZipDeflate, ZipPassThrough, strToU8 } from 'fflate'

const chunks = []

const zip = new Zip((err, chunk, final) => {
  if (err) throw err

  chunks.push(chunk)

  if (final) {
    console.log('zip complete')
  }
})

const textFile = new ZipDeflate('hello.txt', { level: 9 })
zip.add(textFile)
textFile.push(strToU8('Hello world!'), true)

const pngFile = new ZipPassThrough('photo.png')
zip.add(pngFile)
pngFile.push(pngData, true)

zip.end()
```

Always call `zip.add()` before pushing to a ZIP file stream, and call `zip.end()` when finished so the ZIP is valid. Use `ZipPassThrough` for already-compressed files (it behaves like `ZipDeflate` with `level: 0` but allows for better tree-shaking), and use `AsyncZipDeflate` for large files if you want compression to run off the main thread. Async ZIP streams automatically use multicore compression, so different files can be compressed in parallel.

ZIP streams are highly extensible: they take streams as both input and output, and you can plug in custom compression or decompression algorithms from other libraries as long as they [are defined in the ZIP spec](https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT) (see section 4.4.5). If you would like more info on custom compressors, [feel free to ask](https://github.com/101arrowz/fflate/discussions).

### Streaming ZIP extraction

For whole-archive extraction, prefer `unzip()` or `unzipSync()`. `Unzip` is for streaming file-by-file processing: each file stream's `final` value marks that file complete, and passing `true` to the last `unzip.push(chunk, true)` tells the parser that the archive input is complete. If you need to know when all selected files are done, count the files you start and wait for each selected file stream's `final` callback.

```js
import { Unzip, UnzipInflate } from 'fflate'

const unzip = new Unzip(file => {
  if (!file.name.endsWith('.txt')) {
    return
  }

  const chunks = []

  file.ondata = (err, chunk, final) => {
    if (err) throw err

    chunks.push(chunk)

    if (final) {
      console.log(file.name, concat(chunks))
    }
  }

  file.start()
})

unzip.register(UnzipInflate)

unzip.push(chunk1)
unzip.push(chunk2)
unzip.push(lastChunk, true)

function concat(chunks) {
  let length = 0

  for (const chunk of chunks) {
    length += chunk.length
  }

  const result = new Uint8Array(length)
  let offset = 0

  for (const chunk of chunks) {
    result.set(chunk, offset)
    offset += chunk.length
  }

  return result
}
```

You must `register()` a decompression algorithm before starting any compressed file. `UnzipInflate` handles DEFLATE, which is almost always what ZIP files use; registering additional algorithms is how you add support for formats such as BZIP2 or LZMA. If your ZIP files are stored uncompressed, registration is not needed. Register `AsyncUnzipInflate` instead of `UnzipInflate` if you want DEFLATE work to happen off the main thread. Only files whose stream you `start()` will emit data, so skipping files you do not need improves performance.

When streaming ZIP input, file sizes may be missing until the ZIP metadata is available. Check that `file.size` or `file.originalSize` is not `undefined` before relying on it.

To avoid stack limit errors, try to keep each pushed chunk under about 5,000 files' worth of metadata. If files are a few kB each, multi-megabyte chunks are fine; if files are mostly under 100 bytes, keep chunks around 64kB.

## Async streams

Async streams are useful when you need streaming behavior without blocking the main thread.

```js
import { AsyncGzip } from 'fflate'

const gzip = new AsyncGzip({ level: 9, mem: 12, filename: 'hello.txt' })

gzip.ondata = (err, chunk, final) => {
  if (err) {
    console.error(err)
    return
  }

  console.log(chunk, final)
}

gzip.push(chunk)
gzip.push(lastChunk, true)
```

Async streams consume pushed buffers internally and render them unusable. If you still need a buffer after pushing it, clone it first.

Because async streams run off the main thread, their callbacks are never invoked synchronously during `push()`. If you absolutely need synchronous callback behavior, use synchronous streams instead. After an error, an async stream becomes corrupt and must be discarded; you cannot keep pushing chunks to it.

Terminate an async stream when you no longer need it:

```js
gzip.terminate()
```

### The `consume` option

Most one-shot async functions accept a `consume` option. When `true`, the input buffer is rendered unusable, but performance improves and memory usage drops significantly because the data does not need to be copied:

```js
import { zlib } from 'fflate'

zlib(aMassiveFile, { consume: true, level: 9 }, (err, data) => {
  // aMassiveFile can no longer be used, but this saved a copy
})
```

`unzip` is the only async function that does not support `consume`.

## Browser usage

Create a downloadable ZIP in the browser:

```js
import { zipSync, strToU8 } from 'fflate'

const zipData = zipSync({
  'hello.txt': strToU8('Hello from the browser!')
})

const blob = new Blob([zipData], { type: 'application/zip' })
const url = URL.createObjectURL(blob)

const a = document.createElement('a')
a.href = url
a.download = 'archive.zip'
a.click()

URL.revokeObjectURL(url)
```

## CDN usage

For production apps, npm plus tree-shaking is recommended. For quick demos, use a pinned CDN URL. Use either unpkg or jsDelivr, not both.

```html
<script src="https://unpkg.com/fflate@0.8.3"></script>
<script>
  const data = fflate.strToU8('Hello world!')
  const compressed = fflate.gzipSync(data)
  const decompressed = fflate.gunzipSync(compressed)

  console.log(fflate.strFromU8(decompressed))
</script>
```

jsDelivr is also supported:

```html
<script src="https://cdn.jsdelivr.net/npm/fflate@0.8.3/umd/index.js"></script>
```

Tree-shaking is not available from the UMD CDN builds, so they include the whole library (about 33kB, or 12.5kB gzipped).

## Deno and buildless ESM

For Deno (the `@deno-types` comment adds TypeScript typings; the `?dts` Skypack flag is not needed):

```js
// @deno-types="https://cdn.skypack.dev/fflate@0.8.3/lib/index.d.ts"
import * as fflate from 'https://cdn.skypack.dev/fflate@0.8.3?min'
```

For buildless browser ESM:

```js
import * as fflate from 'fflate/esm/browser.js'
```

For older Node.js setups where the standard ESM import does not work:

```js
import * as fflate from 'fflate/esm'
```

## Binary strings

Binary strings are inefficient and usually not recommended (they tend to double file size), but they are supported for compatibility. Pass `true` as the second argument to `strFromU8()` or `strToU8()` for Latin-1 style binary strings, which can represent binary data that is not necessarily valid UTF-8.

```js
import { compressSync, decompressSync, strFromU8, strToU8 } from 'fflate'

const data = strToU8('Hello world!')
const binaryString = strFromU8(compressSync(data), true)

const restored = decompressSync(strToU8(binaryString, true))
console.log(strFromU8(restored))
```

## Bundle size estimates

Bundle size reports that include the whole package (such as Bundlephobia) are upper bounds. `fflate` is designed to be tree-shaken, so your actual bundle depends on which APIs you import. The maximum possible bundle is about 33kB (12.5kB gzipped) if you use every feature, but feature parity with `pako` is only around 10kB, compared with about 45kB for `pako` itself.

| Feature | Approximate minified size | Nearest competitor |
| --- | --- | --- |
| Decompression | 3kB | `tiny-inflate` |
| Compression | 5kB | `UZIP.js`, 2.84x larger |
| Async decompression | 4kB | N/A |
| Async compression | 6kB | N/A |
| ZIP decompression | 5kB | `UZIP.js`, 2.84x larger |
| ZIP compression | 7kB | `UZIP.js`, 2.03x larger |
| GZIP/Zlib decompression | 4kB | `pako`, 11.4x larger |
| GZIP/Zlib compression | 5kB | `pako`, 9.12x larger |
| Streaming decompression | 4kB | `pako`, 11.4x larger |
| Streaming compression | 5kB | `pako`, 9.12x larger |

The full feature set is about 33kB minified, or about 12.5kB gzipped. If your bundle size increases dramatically after adding `fflate`, please [create an issue](https://github.com/101arrowz/fflate/issues/new).

## What makes fflate fast?

Many JavaScript compression libraries are ports of native libraries. The most popular, `pako`, is essentially Zlib rewritten nearly line-for-line in JavaScript; it is well made, but it does not account for the differences between JavaScript and C, so it is suboptimal for performance, and even minified it weighs about 45kB. Small decompression-only libraries like `tiny-inflate` are appealing at 3kB but tend to run about 40% slower than `pako`. `UZIP.js` is both faster (up to 25%) and smaller (about 14kB) than `pako` and includes many clever innovations, but it has some inefficiencies and does not support GZIP or Zlib directly.

`fflate` is written and optimized for JavaScript while still producing standards-compatible output. It builds on the ideas from `UZIP.js`, optimizes them, adds direct GZIP and Zlib support, and uses ES modules so bundlers can remove unused code. That lets it rival even `tiny-inflate` in size while staying about 25% faster than `UZIP.js` and up to 50% faster than `pako`, with equal or better compression ratios, all while supporting compression, decompression, ZIP archives, streaming, dictionaries, and async workers.

JavaScript cannot fully match a native program. If you only target Node.js and do not need browser support or ZIP utilities, Node's native `zlib` bindings may be faster for some workloads. Even so, `fflate` is only around 30% slower than Zlib in decompression and about 10% slower in compression, and it can still achieve better compression ratios. `fflate` is most useful when you need a portable JavaScript implementation, small browser bundles, ZIP support, or consistent APIs across runtimes.

## What about CompressionStream?

The browser `CompressionStream` API can compress and decompress GZIP, Zlib, and DEFLATE data without a third-party dependency, and it wraps native Zlib bindings for good streaming performance. It is a good fit when you want a native streaming API and do not need older browser support.

However, browsers offer no native non-streaming compression API, and `CompressionStream` has surprisingly poor performance on data already loaded into memory; `fflate` tends to be faster even for files dozens of megabytes large, and much faster for files under a megabyte because it avoids marshalling overhead. Even when streaming hundreds of megabytes, the native API typically performs between 30% faster and 10% slower than `fflate`.

`fflate` is still useful when you need:

- ZIP support
- sync APIs
- non-streaming APIs
- compression level control
- older browser support
- consistent browser and Node.js behavior
- a small, tree-shakeable dependency

There is also an `fflate`-based [Compression Streams ponyfill](https://github.com/101arrowz/compression-streams-polyfill) for older browsers.

## Browser support

`fflate` uses typed arrays such as `Uint8Array` and `Uint16Array`. Typed arrays can be polyfilled at the cost of performance, but the most recent browser that lacks them [is from 2011](https://caniuse.com/typedarrays), so it is rarely worth bothering.

Async APIs use `Worker` where available. The vast majority of browsers that support typed arrays also support workers.

Other than those features, `fflate` is ES3-compatible, so you usually will not even need a bundler to use it.

## Documentation

See the [API documentation](./docs/README.md) for the full reference.

Try the [browser demo](https://101arrowz.github.io/fflate) without installing anything.

For questions, bug reports, or feature requests, use [GitHub issues](https://github.com/101arrowz/fflate/issues) or [GitHub discussions](https://github.com/101arrowz/fflate/discussions).

## Testing

Run the test suite with:

```sh
npm test
```

The tests validate behavior, ensure the outputs are no more than 5% larger than competitors at maximum compression, and write performance metrics to `test/results`.

The time shown by the CLI while tests are running is not the same as the measured package performance. Check the JSON output for accurate benchmark data.

## License

This software is [MIT licensed](./LICENSE), with special exemptions for projects and organizations as noted below:

- [SheetJS](https://github.com/SheetJS/) is exempt from MIT licensing and may license any source code from this software under the BSD Zero Clause License.
