# Safely Confirming an AI-Generated Report Design Without Printing It

## What this is about

"Have AI write the report layout" is becoming a common pattern. Ask ChatGPT or
Claude to "build me a JSON template for this form," and you get something
usable back almost immediately.

Two problems stand in the way of using this in production, though.

1. **AI's output is probabilistic** — the same prompt can produce a slightly
   different result each time. Deploying it straight to production without
   checking is risky.
2. **Many tools let the preview drift from the actual printed result.** You end
   up having to print and verify by hand, and every time you ask AI to revise,
   you're back at the printer.

This article shows how ACR (Across Report), a report engine I develop, solves
both problems — with working code.

## Problem 1: if preview doesn't match print, iterating with AI doesn't work

When you have AI write a report's JSON and go back and forth refining it, the
ideal loop looks like this:

```
AI writes a definition → check the preview → something's off, AI fixes it →
preview again → confirm
```

This loop only works if "the preview matches the printed result" holds. If a
layout looks fine in preview but ruled lines shift or a barcode becomes
unreadable once actually printed, you're back to verifying on paper every
time — and the whole point of iterating with AI evaporates.

ACR renders PDF/PNG directly via Google Skia, so the preview (HTML/on-screen)
and the final output (PDF) both reference the same intermediate render model
(`RenderCommand`). That keeps preview and print in sync without needing
physical hardware, which is what lets the AI iteration loop close entirely on
a screen.

## Problem 2: how do you prove a confirmed definition wasn't altered afterward?

Say a human reviews an AI-written definition and confirms it's good. After
that, though:

- Has the JSON file been copied somewhere else and silently edited?
- Does rendering it in production produce the same result it did at
  confirmation time?
- Did a font or engine version change alter the appearance?

Without a way to check these after the fact, the act of "confirming" it
doesn't actually mean much.

### Implementation: hashing the template, the data, and the render output

I added an opt-in Cargo feature called `provenance` to ACR's native engine
(`acr_dll`, written in Rust). Enabling it adds two new functions without
touching any existing FFI exports.

```rust
// Cargo.toml
[features]
provenance = ["acr_core/provenance"]
```

```bash
cargo build --release -p acr_dll --features provenance
```

```c
// New C ABI (existing acr_render_pdf etc. are untouched)
char* acr_compute_render_manifest(const char* template_json, const char* data_json);
void  acr_free_manifest_string(char* ptr);
```

Calling `acr_compute_render_manifest` returns JSON like this:

```json
{
  "template_sha256": "4c0592dc635ba2dd9aea6379189a4b7845414b3496d5d367e4cb2c055a5b7516",
  "data_sha256": "bbc579eb2b157e56080fa61972a7a63054e323400a7ef731d0012948f19c32bc",
  "acr_version": "2.0.0",
  "render_sha256": "4f53cda18c2baa0c0354bb5f9a3ecbe5ed12ab4d8e11ba873c2f11161202b945",
  "fonts_sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
}
```

Two design points matter here.

**1. Template/data hashing is insensitive to key order and whitespace**

Hashing the raw input JSON string directly would treat two JSON documents with
identical content — but different key order or whitespace — as different
inputs. Instead, the input is parsed into a `serde_json::Value` and
re-serialized before hashing (by default, `serde_json` keeps object keys in a
`BTreeMap`, so the output always has a stable key order).

```rust
pub fn canonical_json_sha256(json_str: &str) -> Result<String> {
    let value: serde_json::Value = serde_json::from_str(json_str)?;
    let canonical = serde_json::to_vec(&value)?;
    Ok(sha256_hex(&canonical))
}
```

**2. The render hash targets the intermediate render commands, not the final PDF**

The final PDF byte stream can change for reasons unrelated to content — an
embedded creation timestamp, or subtle rasterization differences across
OS/font environments. That makes it useless for verifying "same input always
produces the same result." Instead, the hash targets the intermediate render
commands (Rust's `Vec<Vec<RenderCommand>>`) right before Skia draws them. Given
the same inputs (template and data), this hash is identical regardless of OS
or timing.

### How verification works

There's no dedicated "verify" function in the engine. Verification is just the
caller's job: compare a **saved manifest** against the result of calling
`acr_compute_render_manifest` again with the **same template and data**.

- Only `template_sha256` differs → the form itself was swapped out
- Only `data_sha256` differs → the data content changed
- Inputs are identical but `render_sha256` differs → the engine or font
  environment changed

### Unaffected by watermark/license state

`render_sha256` is computed at the intermediate render-command stage, *before*
Skia actually draws anything. Whether a watermark gets applied happens later,
in the drawing step, so verifying on an unlicensed machine produces the same
hash as on a licensed one. This cleanly separates "was the confirmed content
correct?" from "does the output carry a watermark?"

### What was actually verified

Using a real template and real data (a report with 433 detail rows), the
following were confirmed with tests:

- Same input → byte-identical manifest (determinism)
- Same content, different formatting → identical `data_sha256` (canonicalization
  works)
- Changing the number of detail rows → `render_sha256` changes while
  `template_sha256` stays the same

## Summary

- A workflow where AI writes a report design and a human iterates on it only
  works without physical hardware if "preview = print result" actually holds
- Making "we confirmed this" verifiable after the fact requires hashing the
  template, the data, the render output, and the engine environment
  separately, then re-rendering and comparing later
- Hashing the intermediate render commands rather than the final PDF keeps the
  reproducibility check immune to environment differences and embedded
  metadata
