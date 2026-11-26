<h1 align="center">spark: native memory fork</h1>

<p align="center">
  A drop-in fork of <a href="https://github.com/lucko/spark">spark</a> that finds
  <b>native (off-heap)</b> and <b>Java heap</b> memory leaks.
</p>

---

## Why

Upstream spark profiles CPU time and Java allocation, but it never sees `malloc`.
That hides a whole class of server memory problems:

- `Inflater`/`Deflater` zlib streams that are never `end()`ed
- Netty direct buffers that are never `release()`d
- image codec state that is never `dispose()`d
- memory allocated inside JNI libraries

**Symptom:** RSS climbs for days, the Java heap graph stays flat, and then the
server gets OOM-killed.

## Usage

Replace `spark.jar` with this jar. Commands, permissions and viewer links are the
same as upstream.

```
/spark profiler start --leaks --timeout 3600
```

| Flag             | Detects                         |
|------------------|---------------------------------|
| `--leaks`        | native and Java heap leaks      |
| `--native-leaks` | native (off-heap) leaks only    |
| `--heap-leaks`   | Java heap retention only        |

### Limits

- **Linux or macOS only.** Leak detection needs async-profiler, and fails
  loudly instead of returning a profile with no leak data.
- **No window rotation.** A leak outlives the window it was made in, so leak
  modes record one continuous window. They can't be combined with
  `--only-ticks-over`.
- **`mmap` is not tracked.** Metaspace, the JIT code cache, GC structures and
  `MappedByteBuffer` are mostly invisible. For metaspace leaks from repeated
  `/reload`, use NMT.
- **Not all growth is a leak.** glibc arenas can hold 1 GB+ of RSS without
  leaking, and Netty's pooled arenas grow to peak load and stay there. A
  plateau is normal; steady growth is suspicious.

## How it works

- One async-profiler recording carries execution samples, live-object
  allocations and `malloc`/`free` events together (`alloc=` and `nativemem=`
  run alongside `event=`).
- A native leak is a `malloc` with no matching `free`. spark pairs them by
  address in-process, matching `jfrconv --leak` totals byte for byte, with no
  conversion tool needed on the server.
- Allocations from the last 10% of the recording are dropped. They haven't had
  a fair chance to be freed, and would bury a real slow leak in noise.

Profiles keep the `application/x-spark-sampler` content type, so links open in
the normal spark viewer. It shows the CPU profile and ignores the leak data.
Reading the leak data needs a consumer that knows the extended schema; field
numbers are in [`CHANGES.md`](CHANGES.md).

## Building

Requires JDK 21.

```bash
./gradlew :spark-bukkit:build      # Bukkit / Paper plugin
./gradlew :spark-velocity:build    # Velocity plugin
./gradlew :spark-geyser:build      # Geyser extension
```

- **Bukkit and Paper:** use `spark-<version>-bukkit.jar`. The `spark-paper`
  module is the library Paper bundles in its server jar; it has no `plugin.yml`
  and is not a plugin.
- **Geyser:** `spark-<version>-geyser.jar` is a Geyser extension; put it in
  Geyser's `extensions/` folder. It works on Geyser Standalone and on Geyser
  running as a plugin or mod. Requires Geyser API 2.9.0+.
  - `/spark tps` reports nothing, since Geyser has no tick loop. Everything
    else works, including all three leak modes.
  - The warning that the extension *"loads class one.profiler.Recording from an
    external source"* is expected: that class is async-profiler's JNI helper.

## License

**GPL-3.0-or-later**, same as spark. Original copyright notices are intact, and
[`CHANGES.md`](CHANGES.md) lists every modification (GPLv3 §5(a)).
spark is itself a fork of [WarmRoast](https://github.com/sk89q/WarmRoast) by
sk89q (GPLv3). The bundled async-profiler is Apache-2.0.

Not affiliated with or endorsed by lucko. Report this fork's bugs here, not on
the spark issue tracker. Upstream's README is kept as
[`README.spark.md`](README.spark.md).
