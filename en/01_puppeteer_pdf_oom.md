# Fixing Out-of-Memory Errors in Puppeteer PDF Generation

## What this is about

Many teams generate invoices, delivery slips, and other business documents as PDFs from a web system using Puppeteer (Headless Chrome). It's an easy way to lay out documents in HTML/CSS, but once you scale up, you tend to hit a wall:

- Memory balloons and the process gets OOM-killed once page counts reach the hundreds or thousands
- Docker container memory limits (512MB–1GB) get hit almost immediately
- Per-page generation time stays reasonable, but total time degrades worse than linearly

This article first breaks down the structural reasons Puppeteer runs into OOM, then introduces an alternative: skipping the browser engine entirely. I'll walk through a real example from ACR (Across Report), a report engine I develop in Rust.

## Why Puppeteer runs out of memory

When generating PDFs for large numbers of pages with Puppeteer / Headless Chrome, memory bloat usually comes down to three causes.

### 1. The renderer process heap isn't designed for reuse

A Chrome renderer process is fundamentally designed around "open a page, then close it." If you keep calling `page.pdf()` repeatedly on the same `page` object for hundreds of report pages in a row, the V8 heap and DOM tree caches accumulate without being released. Get the timing of `page.close()` / `browser.close()` wrong, and zombie Chrome processes linger around your Node.js process too.

### 2. Fonts and images get reloaded per page

In HTML/CSS-based rendering, a Japanese font specified via `@font-face` (especially one that isn't subsetted) can get re-parsed on every page transition. If the same font gets re-expanded for every single report page, both memory usage and rendering time degrade proportionally to page count.

### 3. The `page.pdf()` output buffer accumulates in-process

When combining many pages into a single PDF, if the implementation holds each page's PDF byte sequence in memory before concatenating, the final PDF size translates directly into the Node.js process's memory footprint.

## Mitigations (if you stick with Puppeteer)

- Restart the `browser` every N pages and treat the process as disposable
- Reference OS-installed fonts via `font-family` directly instead of embedding Base64 fonts
- Isolate generation into worker processes so an OOM doesn't take down the main process (tune `--max-old-space-size` alongside this)
- Give Docker memory limits enough headroom for Chrome's peak memory (typically 3–5x normal usage)

These are workarounds, though — they don't address the root cause, which is repurposing a browser engine for report rendering in the first place. If your report volume and page counts are only going to grow, it's worth considering a different architecture.

## An alternative: a report renderer that skips the browser

ACR (Across Report), which I develop, converts a JSON template into an intermediate rendering model (`RenderCommand`) instead of HTML/CSS, and renders PDF/PNG directly via Google Skia. Written in Rust, it bypasses Chrome entirely, which means:

- There's no concept of a "renderer process," so the V8-heap-accumulation problem simply doesn't exist
- Fonts are loaded once at process startup and cached; everything after that is a reference
- The generated PDF is written page-by-page as a stream, so final file size doesn't directly map to process memory

That said, "not using a browser" doesn't automatically mean "safe" — the same class of bug can creep in depending on the implementation. Here's a real case I ran into.

## Case study: rebuilding a font "per page" caused the same kind of bloat

ACR's watermark drawing routine, `draw_watermark()`, originally resolved a fresh `Typeface` via `FontMgr::new()` on every page. This is structurally the exact same problem as Puppeteer's font re-parsing issue.

The result: the PDF output library (Skia's `SkPDF`) ended up embedding a separate font subset per page instead of sharing one. On a 1,394-page sample report, this produced:

| | Before fix | After fix |
|---|---|---|
| PDF size | ~300MB | ~10.3MB |
| Generation time | ~28s | ~2.5s |

The fix was simple: build the font once outside the page loop, and pass a reference into `draw_watermark()` instead of rebuilding it (applying the same pattern already used for `font_cache`).

What this case illustrates is a general principle: in any code generating large numbers of report pages, be suspicious of "whatever we're doing for every single page." Whether in Puppeteer or ACR, most OOM and bloat issues trace back to this pattern — initialization that should happen once, but is instead repeated inside the loop.

## Summary

- Heavy or crashing PDF generation for large page counts with Puppeteer is, to some degree, a structural consequence of how Chrome's renderer process works
- Workarounds (browser restarts, worker isolation, memory limit tuning) can get you by, but only up to a point
- If you want to fix it at the root, a native renderer that bypasses the browser engine is one option
- Even native implementations can fall into the same trap of redundant per-page initialization, though, so the profiling mindset doesn't change

---

**Note / consultation**: I offer consulting on migrating existing report systems (e.g., moving off ActiveReports) using ACR (Across Report), as well as reducing server costs by moving off Puppeteer-based setups.
If you're dealing with rising license renewal costs, or want to port your system cheaply to Linux / cloud environments, feel free to reach out via [Across Systems contact](https://acrossreport.com).
