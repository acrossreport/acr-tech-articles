# Building a Blazing-Fast Report PDF Environment on Docker/Linux, Callable from C# and Node.js

## The goal

Carving out just the report-PDF-generation component into a Linux container, callable from existing C# or Node.js applications, is an increasingly common need — usually driven by wanting to cut Windows server licensing/ops costs, or to autoscale on Kubernetes.

This article walks through that setup using ACR (Across Report), a Rust-based report engine I develop: running a native renderer in a Docker + Linux environment and calling it from existing C# / Node.js code with minimal changes. The key idea is to **ship neither a browser engine nor a .NET runtime in the container — just a single native binary**.

## Overall architecture

```
┌─────────────────────────┐
│   C# app  /  Node.js app │
└───────────┬─────────────┘
            │ FFI (axr_layout.so)  /  N-API (acr_node_native.so)
┌───────────▼─────────────┐
│    Docker container (Linux) │
│  ├ acr_core              │ ← JSON template → RenderCommand
│  ├ acr_renderer_skia     │ ← RenderCommand → PDF/PNG (Skia)
│  └ fonts-vlgothic        │ ← Japanese fonts
└─────────────────────────┘
```

What you bundle into the container depends slightly on the calling language:

- **From C#**: call `libaxr_layout.so` via P/Invoke (the same FFI interface as Windows' `axr_layout.dll`)
- **From Node.js**: `require` the `acr_node_native` native addon (built with napi-rs) directly

Either path ultimately generates the PDF/PNG through the same underlying Rust binary.

## Build details that matter

### 1. Building the Rust core

```bash
cargo build --release
```

This builds every crate, including `acr_core` / `acr_renderer_skia`, for Linux. None of the Windows-ARM64-specific workarounds apply here — `skia-safe`'s prebuilt binaries just work on Linux x86_64.

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

### 3. Example Dockerfile (Node.js consumer)

```dockerfile
# ---- build stage ----
FROM rust:1.78-bookworm AS builder
WORKDIR /app
COPY . .
RUN cargo build --release -p acr_node_native

# ---- runtime stage ----
FROM node:20-bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends \
        fonts-vlgothic \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=builder /app/target/release/libacr_node_native.so ./index.node
COPY acr_node_native/index.js acr_node_native/index.d.ts ./
COPY acr_node_native/package.json ./

# COPY your app code here, and call it via require('./index.node')
```

Two things matter here:

- **Separate the build stage from the runtime stage** so the final image doesn't carry the Rust toolchain, keeping it lean
- **Only install Japanese fonts in the runtime stage** (`apt install fonts-vlgothic`) — the build stage doesn't need them

### 4. Calling from C#

If your C# app is already on .NET (Core) and cross-platform-ready, referencing `libaxr_layout.so` via P/Invoke just works on Linux, with the exact same calling code as on Windows.

```c
char* render_html(const char* template_json, const char* data_json);
void free_string(char* ptr);
```

If you're still on .NET Framework, migrating to .NET (Core) / .NET 8+ is a prerequisite — but since the report-rendering call interface is identical to the Windows version, you can "Linux-ify and validate the report engine first, and tackle the rest of the app's .NET migration separately, later."

## Why this setup ends up lighter

Compared to Chrome-based PDF generation (Puppeteer, etc.), this architecture wins on a few fronts:

- No need to ship a full Chromium install (hundreds of MB) in the runtime image
- No "renderer process" concept means none of the process-management/memory-monitoring complexity that scales with page count
- A single self-contained Rust binary means faster container startup and shorter cold starts under autoscaling

## Summary

- Carving out just report-PDF generation into a Linux container, callable natively from C# or Node.js, is achievable via a multi-stage build plus bundling Japanese fonts only in the runtime stage
- With two call paths available — a napi-rs addon for Node.js, and a shared `.so`/`.dll` FFI interface for C# — the same engine can serve callers regardless of language
- Not depending on a browser engine gives you an edge on both image size and startup speed

---

**Note / consultation**: I offer consulting on migrating existing report systems (e.g., moving off ActiveReports) using ACR (Across Report), as well as reducing server costs by moving off Puppeteer-based setups.
If you're dealing with rising license renewal costs, or want to port your system cheaply to Linux / cloud environments, feel free to reach out via [Across Systems contact](https://acrossreport.com).
