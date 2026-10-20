# Building a Blazing-Fast Report PDF Environment on Docker/Linux, Callable from C# and Node.js

## The goal

Carving out just the report-PDF-generation component into a Linux container, callable from existing C# or Node.js applications, is an increasingly common need — usually driven by wanting to cut Windows server licensing/ops costs, or to autoscale on Kubernetes.

This article walks through that setup using ACR (Across Report), a Rust-based report engine I develop: running a native renderer in a Docker + Linux environment and calling it from existing C# / Node.js code with minimal changes. The key idea is to **ship neither a browser engine nor a .NET runtime in the container — just a single native binary**.

## Overall architecture

```
┌─────────────────────────┐
│   C# app  /  Node.js app │
└───────────┬─────────────┘
            │ FFI (libaxr_runtime.so)  /  N-API (index.linux-x64-gnu.node)
┌───────────▼─────────────┐
│    Docker container (Linux) │
│  ├ acr_core              │ ← JSON template → RenderCommand
│  ├ acr_renderer_skia     │ ← RenderCommand → PDF/PNG (Skia)
│  └ fonts-vlgothic        │ ← Japanese fonts
└─────────────────────────┘
```

What you bundle into the container depends slightly on the calling language:

- **From C#**: call `libaxr_runtime.so` via P/Invoke (the same FFI interface as Windows' `axr_runtime.dll`)
- **From Node.js**: `require` the `acr_node_native` native addon (built with napi-rs) directly

Either path ultimately generates the PDF/PNG through the same underlying Rust binary.

## Build details that matter

### 1. Building the Rust core

```bash
cargo build --release
```

This builds every crate, including `acr_core` / `acr_renderer_skia`, for Linux. None of the Windows-ARM64-specific workarounds apply here — `skia-safe`'s prebuilt binaries just work on Linux x86_64.

The examples below assume Rust 1.82+ and Node.js 22 (LTS). Node 20 reached end-of-life in April 2026, so use `node:22-bookworm-slim` or later for any new setup.

### 2. Calling from Node.js (napi-rs)

The Node.js-facing native addon lives in the `acr_node_native` crate. It's a `cdylib` built with N-API (`napi` / `napi-derive`), exposing functions you can call directly from JS:

```ts
// index.d.ts (excerpt)
export function renderPdf(templateJson: string, dataJson: string, outputPath: string): number
export function renderPreview(templateJson: string, dataJson: string): Array<PageImage>
```

To build it:

```bash
cd acr_node_native
npm install
npx napi build --platform --release
```

If your workspace customizes `target-dir` (e.g., via `.cargo/config.toml`), `napi build`'s post-processing step can fail to locate the `.node` file. In that case, fall back to:

```bash
cargo build --release -p acr_node_native
# then manually copy/rename the built .so to index.<platform-triple>.node / index.node
```

Baking this manual copy step directly into your `Dockerfile`'s multi-stage build avoids this trap.

One detail to watch: the copy's destination filename. The `index.js` napi-rs generates doesn't look for a fixed `index.node` — it does `require('./index.linux-x64-gnu.node')` (or `index.linux-x64-musl.node` on musl-based images), i.e. it looks for a **platform-triple-qualified filename**. If you build and run on a glibc-based `node:slim` image, you need to copy the binary in as `index.linux-x64-gnu.node`.

### 3. Example Dockerfile (Node.js consumer)

```dockerfile
# ---- build stage ----
FROM rust:1.82-bookworm AS builder
WORKDIR /app
COPY . .
RUN cargo build --release -p acr_node_native

# ---- runtime stage ----
FROM node:22-bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends \
        fonts-vlgothic \
        libfontconfig1 \
        libfreetype6 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=builder /app/target/release/libacr_node_native.so ./index.linux-x64-gnu.node
COPY acr_node_native/index.js acr_node_native/index.d.ts ./
COPY acr_node_native/package.json ./

# COPY your app code here, and call it via require('./index.js')
```

Three things matter here:

- **Separate the build stage from the runtime stage** so the final image doesn't carry the Rust toolchain, keeping it lean
- **Only install Japanese fonts in the runtime stage** (`apt install fonts-vlgothic`) — the build stage doesn't need them
- **`skia-safe` needs `libfontconfig1` / `libfreetype6` at runtime.** These aren't included by default in `node:*-slim` images, so install them explicitly

### 4. Calling from C#

If your C# app is already on .NET (Core) and cross-platform-ready, referencing `libaxr_runtime.so` via P/Invoke just works on Linux, with the exact same calling code as on Windows. The buffer the native side returns isn't managed by the .NET GC, so you must explicitly free it once you're done with it.

One more thing to watch: string marshaling. If a `string` parameter has no `CharSet` or `MarshalAs` specified, Windows marshals it as ANSI by default (Shift-JIS on a Japanese-locale machine) before handing it to the native side. The Rust side expects UTF-8 (it reads the buffer as a `CStr`), so without explicit marshaling, Japanese text inside your JSON can get mangled or fail to parse. Add `[MarshalAs(UnmanagedType.LPUTF8Str)]` to marshal explicitly as UTF-8 (on .NET 7+, `LibraryImport` with `StringMarshalling.Utf8` solves the same problem).

```csharp
using System;
using System.Runtime.InteropServices;

internal static class AcrNative
{
    // Resolves to libaxr_runtime.so on Linux, axr_runtime.dll on Windows
    [DllImport("axr_runtime", CallingConvention = CallingConvention.Cdecl)]
    private static extern IntPtr acr_render_pdf(
        [MarshalAs(UnmanagedType.LPUTF8Str)] string templateJson,
        [MarshalAs(UnmanagedType.LPUTF8Str)] string dataJson,
        out int outLen);

    [DllImport("axr_runtime", CallingConvention = CallingConvention.Cdecl)]
    private static extern void acr_free_buffer(IntPtr ptr, int len);

    public static byte[] RenderPdf(string templateJson, string dataJson)
    {
        IntPtr ptr = acr_render_pdf(templateJson, dataJson, out int len);
        if (ptr == IntPtr.Zero) throw new InvalidOperationException("ACR render failed");

        try
        {
            var buffer = new byte[len];
            Marshal.Copy(ptr, buffer, 0, len);
            return buffer;
        }
        finally
        {
            acr_free_buffer(ptr, len); // always free buffers allocated on the Rust side
        }
    }
}
```

If you're still on .NET Framework, migrating to .NET (Core) / .NET 8+ is a prerequisite — but since the report-rendering call interface is identical to the Windows version, you can "Linux-ify and validate the report engine first, and tackle the rest of the app's .NET migration separately, later."

## Why this setup ends up lighter

Compared to Chrome-based PDF generation (Puppeteer, etc.), this architecture wins on a few fronts:

- No need to ship a full Chromium install (hundreds of MB) in the runtime image
- No "renderer process" concept means none of the process-management/memory-monitoring complexity that scales with page count
- A single self-contained Rust binary means faster container startup and shorter cold starts under autoscaling

> **TODO (fill in before publishing)**: Don't leave "lighter" and "blazing-fast" as qualitative claims — back them with real measurements (runtime image size, container cold-start time, pages generated per second, compared against the Puppeteer setup). A title that says "blazing-fast" without numbers to back it up undercuts the article's credibility as a technical piece.

## Summary

- Carving out just report-PDF generation into a Linux container, callable natively from C# or Node.js, is achievable via a multi-stage build plus bundling Japanese fonts only in the runtime stage
- With two call paths available — a napi-rs addon for Node.js, and a shared `.so`/`.dll` FFI interface for C# — the same engine can serve callers regardless of language
- Not depending on a browser engine gives you an edge on both image size and startup speed

---

**Note / consultation**: I offer consulting on migrating existing report systems (e.g., moving off ActiveReports) using ACR (Across Report), as well as reducing server costs by moving off Puppeteer-based setups.
If you're dealing with rising license renewal costs, or want to port your system cheaply to Linux / cloud environments, feel free to reach out via [Across Systems contact](https://acrossreport.com).
