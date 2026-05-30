# fflate

High-performance compression and decompression in an 8kB package.

`fflate` (fast flate) is a tiny, ultra-fast, pure JavaScript library for **DEFLATE, GZIP, Zlib, and ZIP** data. It runs in browsers, Node.js, Deno, and other JavaScript runtimes, and is fully interoperable with standard compression tools in both directions.

## Start here

```sh
npm i fflate
```

```js
import { gzipSync, gunzipSync, strToU8, strFromU8 } from 'fflate'

const compressed = gzipSync(strToU8('Hello world!'))
const text = strFromU8(gunzipSync(compressed))
console.log(text) // 'Hello world!'
```

Need something else? Jump to **[What should I use?](#what-should-i-use)** for the full API map, or **[Why fflate?](#why-fflate)** for benchmarks and bundle sizes.

---

## Why fflate?

Built for projects that need speed, small bundles, and broad format support.

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

**Performance & size**

* **Size:** ~8kB minified core (~3kB decompression-only), versus `pako` (45.6kB) and `UZIP.js` (14.2kB). Fully tree-shakeable.
* **Speed:** decompresses up to 40% faster than `pako`, compresses up to 50% faster than `pako`, and runs ~25% faster than `UZIP.js` overall.
* **Multithreading:** in async mode, web workers deliver over 3x the throughput of most single-threaded utilities.
* **Ratio:** often beats the original Zlib C library; in synchronous mode the compressor can out-do native tools like Info-ZIP on both speed and ratio.

---

## Install

```sh
npm i fflate # or yarn add fflate, or pnpm add fflate
```

**Cherry-pick (recommended — smallest bundles):**

```js
import { zipSync, unzipSync, strToU8, strFromU8 } from 'fflate'
```

**CommonJS:**

```js
const fflate = require('fflate')
```

**Full bundle (browsers):**

```js
import * as fflate from 'fflate'
```

---

## What should I use?

| If you want to... | Use this |
| --- | --- |
| Create / read a `.zip` archive | `zipSync()` / `zip()` · `unzipSync()` / `unzip()` |
| Create / read a `.gz` file | `gzipSync()` / `gzip()` · `gunzipSync()` / `gunzip()` |
| Read a `.tar.gz` file | `gunzipSync()` / `gunzip()`, then a TAR parser — `fflate` does not parse TAR |
| Create / read Zlib data | `zlibSync()` / `zlib()` · `unzlibSync()` / `unzlib()` |
| Create / read raw DEFLATE data | `deflateSync()` / `deflate()` · `inflateSync()` / `inflate()` |
| Compress one buffer with a sensible default | `compressSync()` / `compress()` *(outputs GZIP)* |
| Decompress data of unknown format | `decompressSync()` / `decompress()` *(GZIP, Zlib, or DEFLATE)* |
| Process data piece by piece | `Gzip`, `Gunzip`, `Deflate`, `Inflate`, `Zip`, `Unzip` classes |
| Convert strings ↔ bytes | `strToU8()` / `strFromU8()` |

**Sync, async, or streaming?**

| Situation | Use |
| --- | --- |
| Small inputs, scripts, tests, build tools, Node.js CLIs | Sync APIs |
| Browser UI where blocking the main thread matters | Async APIs |
| Large archives or many large files | Async ZIP APIs |
| Data arrives or leaves in chunks | Streaming classes |

> 💡 **Async startup cost.** Async APIs offload to web workers, adding ~50ms overhead on the *first* call of each distinct function (e.g. `unzip` and `zlib` each pay it once). Under ~50kB, sync APIs are usually faster; for larger or multiple files, async wins decisively.

---

## Basic usage

Compression APIs operate on `Uint8Array` (Node.js `Buffer`s work natively).

**Text**

```js
import { gzipSync, gunzipSync, strToU8, strFromU8 } from 'fflate'

const compressed = gzipSync(strToU8('Hello world!'))
const text = strFromU8(gunzipSync(compressed))
console.log(text) // 'Hello world!'
```

**Auto-detect decompression** — use `decompressSync()` when the format (GZIP, Zlib, or raw DEFLATE) is unknown:

```js
import { decompressSync, strFromU8 } from 'fflate'
import { readFileSync } from 'node:fs'

const data = decompressSync(readFileSync('./data.bin'))
console.log(strFromU8(data))
```

**Compression options** — `compressSync()` is the matching high-level compressor; it outputs GZIP by default:

```js
import { compressSync, strToU8 } from 'fflate'

const compressed = compressSync(strToU8('Hello world!'), {
  level: 6, // 0 (none) to 9 (max). Default: 6
  mem: 8    // 0 to 12. Default: 4. Higher may improve speed at the cost of memory
})
```

---

## ZIP archives

Keys are file paths; values are `Uint8Array`s.

**Create**

```js
import { zipSync, strToU8 } from 'fflate'
import { readFileSync, writeFileSync } from 'node:fs'

const zipData = zipSync({
  'hello.txt': strToU8('Hello world!'),
  'images/photo.png': readFileSync('./photo.png'),
  // Nested objects become directories; filenames may use Unicode
  docs: { '你好.txt': strToU8('Hey there!') }
})

writeFileSync('./archive.zip', zipData)
```

**Compression options** — files are DEFLATE-compressed by default, which is wasteful for already-compressed formats (PNG, JPEG, PDF, `.gz`). Pass a `[data, options]` tuple per file, or a global options object as the second argument. Per-file options override globals.

```js
import { zipSync, strToU8 } from 'fflate'
import { readFileSync } from 'node:fs'

const zipData = zipSync({
  'notes.txt': strToU8('lots of compressible text...'),
  'photo.png': [readFileSync('./photo.png'), { level: 0 }], // store, don't recompress

  // ZIP-only: Unix executable permissions (OS flag + attributes)
  'hello.sh': [strToU8('echo hello world'), { os: 3, attrs: 0o755 << 16 }],

  // Directories take options too, applied to the files inside them
  exec: [{
    'run.sh': [strToU8('echo run'), { os: 3, attrs: 0o755 << 16 }]
  }, {
    mtime: new Date('2020-10-20')
  }]
}, {
  level: 6,                     // global default level
  mtime: new Date('1980-01-01') // global default modification time
})
```

**Read** — returns a flat object of full paths → `Uint8Array` (directories are not nested):

```js
import { unzipSync, strFromU8 } from 'fflate'
import { readFileSync, writeFileSync } from 'node:fs'

const files = unzipSync(readFileSync('./archive.zip'))
console.log(strFromU8(files['hello.txt']))
writeFileSync('./photo.png', files['images/photo.png'])
// e.g. { 'nested/directory/structure.txt': Uint8Array }
```

**Skip files (filter)** — avoid decompressing files you don't need:

```js
import { unzipSync } from 'fflate'
import { readFileSync } from 'node:fs'

const files = unzipSync(readFileSync('./archive.zip'), {
  filter: file => file.name.endsWith('.txt') && file.originalSize <= 10_000_000
})
```

**Async** — runs in parallel across threads, up to 3x faster (most noticeably for multiple large files):

```js
import { zip, unzip, strToU8, strFromU8 } from 'fflate'
import { readFileSync } from 'node:fs'

const files = {
  'hello.txt': strToU8('Hello world!'),
  'images/photo.png': readFileSync('./photo.png')
}

const zipData = await new Promise((resolve, reject) =>
  zip(files, (err, data) => (err ? reject(err) : resolve(data))))

const out = await new Promise((resolve, reject) =>
  unzip(zipData, (err, data) => (err ? reject(err) : resolve(data))))
console.log(strFromU8(out['hello.txt']))
```

> ⚠️ `unzip` is parallelized (often faster than `unzipSync`) and is the **only** async function that does not support the `consume` option.

**Cancel** — every async function returns a termination function. Calling it aborts the work and suppresses the callback:

```js
const terminate = unzip(zipData, (err, files) => { /* ... */ })
terminate()
```

---

## GZIP, Zlib, and raw DEFLATE

These formats handle a single stream and store no directory structure.

```js
import {
  gzipSync, gunzipSync,
  zlibSync, unzlibSync,
  deflateSync, inflateSync,
  strToU8
} from 'fflate'

// GZIP — supports filename + mtime metadata
const gz = gzipSync(strToU8('Data'), { filename: 'file.txt', mtime: new Date() })
gunzipSync(gz)

// Zlib
unzlibSync(zlibSync(strToU8('Data')))

// Raw DEFLATE
inflateSync(deflateSync(strToU8('Data')))
```

`mtime` accepts a `Date`, a date string, or a Unix timestamp.

**`.tar.gz`** — a TAR archive wrapped in GZIP. Decompress the GZIP layer, then pass the result to a TAR parser:

```js
import { gunzipSync } from 'fflate'
import { readFileSync } from 'node:fs'

const tarData = gunzipSync(readFileSync('./archive.tar.gz')) // still TAR data
```

---

## Streaming

Use streams when data arrives in chunks or to bound memory.

* **Sync streams** (`Gzip`, `Gunzip`, `Deflate`, `Inflate`) use a `(chunk, final)` handler and throw errors from `push()`.
* **ZIP and async streams** use an `(err, chunk, final)` handler.
* Set the handler in the constructor or later via `stream.ondata`. Pass `true` to `push()` only on the last chunk.

**Compression / decompression**

```js
import { Gzip, Gunzip } from 'fflate' // swap for Decompress to auto-detect the format

const gzip = new Gzip((chunk, final) => {
  // handle chunk; final === true on the last one
})

gzip.push(chunk1)
gzip.push(lastChunk, true)
```

**Text** — chain `EncodeUTF8`/`DecodeUTF8` for text arriving in chunks (chaining = push to the next stream from the previous handler):

```js
import { EncodeUTF8, DecodeUTF8, Gzip, Gunzip } from 'fflate'

const decoder = new DecodeUTF8((text, final) => console.log(text))
const gunzip  = new Gunzip((chunk, final) => decoder.push(chunk, final))
const gzip    = new Gzip((chunk, final) => gunzip.push(chunk, final))
const encoder = new EncodeUTF8((data, final) => gzip.push(data, final))

encoder.push('Hello ')
encoder.push('world!', true)
```

**ZIP creation** — call `zip.add(stream)` before pushing to that stream, and `zip.end()` once every file is finalized:

```js
import { Zip, ZipDeflate, ZipPassThrough, strToU8 } from 'fflate'

const zip = new Zip((err, chunk, final) => {
  if (err) throw err
  // write chunk to destination
})

const textFile = new ZipDeflate('hello.txt', { level: 9 })
zip.add(textFile)
textFile.push(strToU8('Hello world!'), true)

// Already-compressed file: ZipPassThrough stores it without recompressing
const pngFile = new ZipPassThrough('photo.png')
zip.add(pngFile)
pngFile.push(pngData, true)

zip.end()
```

`ZipPassThrough` behaves like `ZipDeflate` with `level: 0` but tree-shakes better. Use `AsyncZipDeflate` to compress files off the main thread; async ZIP streams compress multiple files in parallel. ZIP streams take streams as both input and output, so you can plug in custom algorithms [defined in the ZIP spec](https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT) (section 4.4.5) — [feel free to ask](https://github.com/101arrowz/fflate/discussions) for help.

**ZIP extraction** — for whole archives, prefer `unzip()` / `unzipSync()`. Use `Unzip` for file-by-file processing. `register()` a decompression algorithm before starting any compressed file; only files you `.start()` consume resources.

```js
import { Unzip, UnzipInflate } from 'fflate'

const unzip = new Unzip(file => {
  if (!file.name.endsWith('.txt')) return // skip
  file.ondata = (err, chunk, final) => {
    if (err) throw err
    // process chunk; final === true when this file is done
  }
  file.start()
})

unzip.register(UnzipInflate) // DEFLATE — almost always what ZIPs use
unzip.push(chunk1)
unzip.push(lastChunk, true)  // true marks the end of the archive input
```

`UnzipInflate` handles DEFLATE; register custom algorithms to add formats like BZIP2 or LZMA. Stored (uncompressed) ZIPs need no registration. Use `AsyncUnzipInflate` for off-main-thread DEFLATE. To know when all wanted files finish, count the ones you `start()` and wait for each `final`.

> ⚠️ **Streaming ZIP caveats**
> * `file.size` / `file.originalSize` may be `undefined` until the ZIP metadata is parsed — check before relying on them.
> * To avoid stack-limit errors, keep each pushed chunk under ~5,000 files' worth of metadata (~64kB chunks when files are under 100 bytes; multi-megabyte chunks are fine for larger files).

---

## Async streams

Streaming off the main thread, without blocking the UI.

```js
import { AsyncGzip } from 'fflate'

const gzip = new AsyncGzip({ level: 9, mem: 12, filename: 'hello.txt' })
gzip.ondata = (err, chunk, final) => {
  if (err) return console.error(err)
  console.log(chunk, final)
}

gzip.push(chunk)
gzip.push(lastChunk, true)
gzip.terminate() // free the worker if you abandon the stream early
```

> ⚠️ **How async streams differ**
> * **Buffers are consumed.** Pushed buffers become unusable — clone first if you still need them.
> * **Callbacks are never synchronous** during `push()`. Use sync streams if you truly need that.
> * **Errors are fatal.** After an error the stream is corrupt and must be discarded.

**The `consume` option** — most one-shot async functions accept `{ consume: true }`, letting `fflate` skip copying the input for lower memory and higher speed, at the cost of making the input buffer unusable:

```js
import { zlib } from 'fflate'

zlib(aMassiveFile, { consume: true, level: 9 }, (err, data) => { /* ... */ })
// aMassiveFile is now unusable, but no copy was made
```

Supported by every async utility except `unzip`.

---

## Other environments

**Browser download**

```js
import { zipSync, strToU8 } from 'fflate'

const zipData = zipSync({ 'hello.txt': strToU8('Hello from the browser!') })
const blob = new Blob([zipData], { type: 'application/zip' })

const a = document.createElement('a')
a.href = URL.createObjectURL(blob)
a.download = 'archive.zip'
a.click()
URL.revokeObjectURL(a.href)
```

**CDN** — production apps should prefer npm + tree-shaking. For demos, pin a version and use **either** unpkg or jsDelivr:

```html
<script src="https://unpkg.com/fflate@0.8.3"></script>
<!-- or -->
<script src="https://cdn.jsdelivr.net/npm/fflate@0.8.3/umd/index.js"></script>
<script>
  const compressed = fflate.gzipSync(fflate.strToU8('Hello!'))
</script>
```

UMD CDN builds don't tree-shake, so they include the whole library (~33kB, 12.5kB gzipped).

**Deno & buildless ESM**

```js
// Deno — the @deno-types comment adds typings; the ?dts flag isn't needed
// @deno-types="https://cdn.skypack.dev/fflate@0.8.3/lib/index.d.ts"
import * as fflate from 'https://cdn.skypack.dev/fflate@0.8.3?min'

// Buildless browser ESM
import * as fflate from 'fflate/esm/browser.js'

// Older Node.js where the standard ESM import fails
import * as fflate from 'fflate/esm'
```

---

## Binary strings

Latin-1 binary strings are supported for backwards compatibility, though inefficient (they roughly double size). They can represent binary data that isn't valid UTF-8. Pass `true` as the second argument:

```js
import { compressSync, decompressSync, strFromU8, strToU8 } from 'fflate'

const binaryString = strFromU8(compressSync(strToU8('Hello')), true)
const restored = strFromU8(decompressSync(strToU8(binaryString, true)))
```

---

## Bundle size estimates

Aggregators like Bundlephobia report the whole package as an upper bound. Because `fflate` tree-shakes, you pay only for what you import. The full feature set is ~33kB minified (12.5kB gzipped); `pako` feature parity is only ~10kB (versus ~45kB for `pako` itself).

| Feature | Minified size | Nearest competitor |
| --- | --- | --- |
| Decompression | 3kB | `tiny-inflate` |
| Compression | 5kB | `UZIP.js`, 2.84x larger |
| Async decompression | 4kB (1kB + raw decompression) | N/A |
| Async compression | 6kB (1kB + raw compression) | N/A |
| ZIP decompression | 5kB (2kB + raw decompression) | `UZIP.js`, 2.84x larger |
| ZIP compression | 7kB (2kB + raw compression) | `UZIP.js`, 2.03x larger |
| GZIP/Zlib decompression | 4kB (1kB + raw decompression) | `pako`, 11.4x larger |
| GZIP/Zlib compression | 5kB (1kB + raw compression) | `pako`, 9.12x larger |
| Streaming decompression | 4kB (1kB + raw decompression) | `pako`, 11.4x larger |
| Streaming compression | 5kB (1kB + raw compression) | `pako`, 9.12x larger |

If your bundle grows dramatically after adding `fflate`, please [open an issue](https://github.com/101arrowz/fflate/issues/new).

---

## Architecture

**Why it's fast.** Most JS compression libraries port C line-for-line. `pako` mirrors Zlib closely and is well made, but ignores JS/C differences and weighs ~45kB minified. `tiny-inflate` is tiny (3kB) but ~40% slower than `pako`. `UZIP.js` is faster (up to 40%) and smaller (~14kB) than `pako` with clever innovations, but has inefficiencies and no direct GZIP/Zlib support. `fflate` builds on `UZIP.js`'s ideas, optimizes them, adds direct GZIP/Zlib support, and ships ES modules so bundlers drop unused code — rivaling `tiny-inflate` in size while staying ~25% faster than `UZIP.js` and up to 50% faster than `pako`, with equal or better ratios, across compression, decompression, ZIP, streaming, dictionaries, and async workers.

**Versus native Node.js `zlib`.** For Node-only backends with no browser or ZIP needs, Node's native `zlib` bindings may be faster. Even so, `fflate` is only ~30% slower in decompression and ~10% slower in compression, and often achieves better ratios.

**Versus `CompressionStream`.** The browser `CompressionStream` API does GZIP/Zlib/DEFLATE over native bindings and streams well, but there's no native non-streaming API and it performs poorly on in-memory data: `fflate` is faster even for files dozens of megabytes large, and much faster below 1MB (no marshalling overhead). Even streaming hundreds of megabytes, the native API ranges from 30% faster to 10% slower than `fflate`. Choose `fflate` when you need ZIP support, sync or non-streaming APIs, compression-level control, older-browser support (or the [Compression Streams ponyfill](https://github.com/101arrowz/compression-streams-polyfill)), a small tree-shakeable dependency, or consistent behavior across browser and Node.

---

## Compatibility

* **Core:** requires typed arrays (`Uint8Array`, `Uint16Array`), polyfillable at a performance cost; the most recent browser [without them is from 2011](https://caniuse.com/typedarrays).
* **Async:** requires `Worker`, supported by nearly all browsers that have typed arrays.
* **Transpilation:** ES3-compatible, so you usually won't even need a bundler.

---

## Documentation

* [Full API reference](./docs/README.md)
* [Browser demo](https://101arrowz.github.io/fflate) — try it without installing
* Questions or bugs: [GitHub issues](https://github.com/101arrowz/fflate/issues) · [discussions](https://github.com/101arrowz/fflate/discussions)

---

## Testing

```sh
npm test
```

Tests validate behavior, verify outputs are no more than 5% larger than competitors at maximum compression, and write performance metrics to `test/results`. The CLI's per-test timing is **not** the measured package performance — check the JSON output for accurate benchmarks.

---

## License

[MIT licensed](./LICENSE), with one exemption:

* [SheetJS](https://github.com/SheetJS/) may license source code from this software under the BSD Zero Clause License.
