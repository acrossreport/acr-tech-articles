# Migrating Off ActiveReports to the Web and Linux While Keeping Your Existing Assets

## Background: three pain points of ActiveReports estates

Plenty of .NET Framework-era business systems still rely on ActiveReports (GrapeCity / MESCIUS) for report output. After years of running in production, these setups tend to accumulate the same three frustrations:

1. **License renewal costs keep growing** — server and runtime license renewal fees no longer feel proportionate to the value the reporting feature alone provides
2. **Locked into Windows / .NET Framework** — recent ActiveReports versions do support .NET 6+ and Linux, but in long-running projects it's typically the web app *hosting* ActiveReports that's still stuck on .NET Framework, which keeps the whole app — not just the report engine — off Linux containers and cloud-native setups
3. **Dozens or hundreds of designer assets (report layouts) already exist** — rebuilding them all from scratch isn't realistic; "keep the existing assets" is a hard requirement

When all three show up together, neither "full replacement" nor "do nothing" is an easy choice, and projects tend to stall.

## The migration approach: aim for "semantic compatibility," not "replacement"

ActiveReports layout files (.rdlx, .rpx, etc.) are XML under the hood, but reading them with "full compatibility" — reproducing output down to the pixel — isn't realistic. Parsing the XML schema itself has little value; the real cost is reproducing everything ActiveReports' rendering engine does internally: layout calculation, page-break decisions, and format resolution.

Instead, ACR (Across Report), which I develop, takes the approach of **reproducing ActiveReports' concepts and behavior at the JSON template level**. Concretely, this means baking in the following compatibility at the implementation level:

### Section structure compatibility

ACR inherits the `Detail` / `PageHeader` / `PageFooter` section concepts from ActiveReports' section reports (`.rpx`) directly, including per-section controls like `NewPage AFTER` (force a page break after a section outputs), reproduced with the same meaning.

### Date format and font specification compatibility

Format tokens from `.NET`, like `yyyy/MM/dd` and `HH:mm:ss`, are interpreted as-is, so existing template format strings can effectively be ported unchanged. Font resolution also follows ActiveReports' own priority order between the `Font` / `FontName` fields and a CSS-style `Style.font-family`.

### Barcode type compatibility

ActiveReports' own barcode type codes (e.g., the legacy code `18`) are mapped onto current types like `Code128B`, absorbing the difference so data-side changes stay minimal.

None of this aims for pixel-perfect, zero-deviation reproduction. The goal is **letting you carry over existing report definitions, data, and operational know-how without a rewrite**.

## Why Linux becomes possible

ACR is a native renderer implemented in Rust + Google Skia, with no dependency on .NET Framework or the Windows API.

- The rendering engine itself (`acr_core` / `acr_renderer_skia`) builds as the same binary structure on Windows, macOS, and Linux
- On Linux, installing Japanese fonts (VL Gothic, etc.) via `apt install fonts-vlgothic` gets you working report output, but VL Gothic's character widths and glyph shapes differ subtly from MS Mincho / MS Gothic, so the result isn't pixel-identical to Windows. This is the single most common place migrations get stuck — don't skip the per-report visual check described under "Validate with parallel operation" below; watch specifically for shifted rule lines, table borders, and line-wrap differences
- Existing .NET / C# assets can call in via FFI (`axr_runtime.dll` / `libaxr_runtime.so`), so you don't need to rewrite the calling code all at once

```c
// FFI interface (shared between axr_runtime.dll and libaxr_runtime.so)
unsigned char* acr_render_pdf(const char* template_json, const char* data_json, int* out_len);
void acr_free_buffer(unsigned char* ptr, int len);
```

In other words, this isn't "throw away the C# app" — it's "swap out just the report-rendering layer for a lightweight Rust/Skia engine, while the calling side keeps using it via FFI, unchanged." Since the same FFI interface works on both Windows and Linux, once you eventually move the main web app into a Linux container, the report portion can be validated and switched over ahead of time, independently.

## A typical migration path

1. **Inventory**: enumerate existing ActiveReports templates. Check section structure, barcode types, and any special formatting (justified/equal-width spacing, ruby annotations, vertical text, etc.)
2. **Convert to JSON templates**: starting with the highest-priority reports, convert ActiveReports definitions into ACR's JSON template format (section-by-section, field-by-field; currently this is a manual process rather than an automated converter tool)
3. **Validate with parallel operation**: run the same data through both the existing ActiveReports output and the new ACR output, then compare visually via the WYSIWYG preview and/or automated diffing
4. **Swap the calling side**: on the C# side, this is just a switch to the FFI call — the web frontend and business logic stay untouched
5. **Move to Linux**: once a report has been validated, move its rendering piece into a Linux container

## Summary

- Migrating off ActiveReports doesn't have to be an all-or-nothing choice between a full rewrite and doing nothing
- By moving to an engine that semantically inherits ActiveReports' *behavior* — section structure, date formats, barcode types — you can migrate while keeping your template assets intact
- Because only the rendering layer needs to be swapped via FFI, moving from Windows to Linux can happen incrementally too

---

**Note / consultation**: I offer consulting on migrating existing report systems (e.g., moving off ActiveReports) using ACR (Across Report), as well as reducing server costs by moving off Puppeteer-based setups.
If you're dealing with rising license renewal costs, or want to port your system cheaply to Linux / cloud environments, feel free to reach out via [Across Systems contact](https://acrossreport.com).
