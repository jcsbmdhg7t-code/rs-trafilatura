# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`rs-trafilatura` is a Rust port of the Python [trafilatura](https://github.com/adbar/trafilatura) web content extraction library. It strips boilerplate and extracts clean article content from HTML, with an XGBoost-based page type classifier (7 types), a per-type extraction profile system, and an ML extraction quality predictor.

## Commands

```bash
# Build
cargo build

# Test all
cargo test

# Test a single test file
cargo test --test extraction_test

# Test a single test by name (substring match)
cargo test test_forum_by_domain

# Lint (strict — matches CI for library code)
cargo clippy --lib

# Lint for tests (relaxed — allows unwrap/expect)
cargo clippy --tests -- -A clippy::expect_used -A clippy::unwrap_used

# Format check (CI)
cargo fmt --all -- --check

# Auto-format
cargo fmt

# Docs
cargo doc --no-deps

# Benchmarks
cargo bench

# CLI binaries
cargo run --bin extract_stdin                    # reads HTML from stdin, outputs JSON
cargo run --bin extract_stdin -- --url https://example.com --page-type product

# Examples
cargo run --example basic
cargo run --example markdown_output
cargo run --example metadata
```

## Lint Rules

Enforced in `Cargo.toml [lints]`:
- `unsafe_code = "forbid"` — no unsafe code anywhere
- `clippy::unwrap_used = "deny"` and `clippy::expect_used = "deny"` in library code
- `clippy::pedantic` is enabled with several explicit allows (see `Cargo.toml`)
- Tests are run with `-A clippy::expect_used -A clippy::unwrap_used` (CI pattern)

## Architecture

### Public API (`src/lib.rs`)

Four entry points, all returning `Result<ExtractResult>`:
- `extract(html: &str)` — uses default `Options`
- `extract_with_options(html: &str, options: &Options)`
- `extract_bytes(html: &[u8])` — auto-detects charset from meta tags
- `extract_bytes_with_options(html: &[u8], options: &Options)`

The `spider` feature flag enables `spider_integration::extract_page()` for use with the spider-rs web crawler.

### Core Types

**`Options`** (`src/options.rs`): 28-field struct controlling extraction behavior. Key fields: `favor_precision`/`favor_recall` (override `min_score` to 5000/500), `output_markdown`, `include_comments/tables/images/links`, `page_type` (override classification), `url` (for hostname + URL-based classification). All fields are `pub` with `..Options::default()` construction pattern.

**`ExtractResult`** (`src/result.rs`): Output struct with `content_text`, `content_html`, `content_markdown` (when `output_markdown` is set), `comments_text/html`, `images: Vec<ImageData>`, `metadata: Metadata`, `classification_confidence: Option<f64>`, `extraction_quality: f64`, `warnings: Vec<String>`.

**`PageType`** (`src/page_type/mod.rs`): 7-variant enum — `Article`, `Forum`, `Product`, `Category`, `Listing`, `Documentation`, `Service`. Note: `PageType::Category` serializes to the string `"collection"` (not `"category"`) via `as_str()`.

### Extraction Pipeline (`src/extract.rs`)

The main function `extract_content()` (4199 lines) orchestrates:

1. **HTML parsing** via `dom_query::Document`
2. **Metadata extraction** — JSON-LD, Open Graph, Dublin Core, DOM fallback (`src/metadata/`)
3. **3-stage page type classification**:
   - Stage 1: URL heuristics (`classify_url()`) — fast path for obvious types
   - Stage 2: HTML signals (`extract_html_signals()` + `refine_with_html_signals()`) — JSON-LD `@type`, OG tags, product grids, pagination
   - Stage 3: ML XGBoost classifier (`src/page_type/ml.rs`, 200 trees, 181 features) — final authority
4. **Document backup** (`doc_backup`) before cleaning — used by fallback extraction to recover content inside `<form>` tags and other elements removed during cleaning
5. **Structured data pre-extraction** — JSON-LD `articleBody` and Discourse `data-preloaded` (before cleaning removes `<script>` tags)
6. **Profile selection** — `PageType::extraction_profile()` returns an `ExtractionProfile` with type-specific `content_selectors`, `boilerplate_selectors`, `preserve_tags`, flags like `comments_are_content`, `aggregate_sections`, `collect_repeated_items`
7. **`doc_cleaning`** (`src/html_processing.rs`) — removes boilerplate per profile
8. **Content node selection** — profile selectors → `selector::content::find_content()` → `<article>`/`<main>` → heuristic → bottom-up Readability scorer
9. **Fallback cascade** when extraction is insufficient:
   - `try_fallback_extraction()` — compares DOM result with `compare_external_extraction()` using `candidate_is_usable()` heuristics (`src/extractor/fallback.rs`)
   - `try_multi_candidate_merge()` — for service pages, collects non-overlapping high-scoring nodes
   - `try_collect_repeated_items()` — for listing pages, extracts repeated sibling article/li elements
   - `extract_collection_description()` — for category pages, prepends SEO description text
   - JSON-LD `articleBody`/Discourse preference when DOM result is inferior
10. **Comment extraction** (`src/extractor/comments.rs`) — when `include_comments = true`
11. **Image extraction** — when `include_images = true`
12. **Markdown generation** via `quick_html2md` — when `output_markdown = true`
13. **Extraction quality score** — heuristic (ratio, length, paragraph structure, link density, boilerplate keywords) + 27-feature XGBoost ML predictor via `web-page-classifier` crate

### Key Thread-Local State

`COMMENTS_ARE_CONTENT` (in `src/extract.rs`) is a `thread_local! Cell<bool>` set to `true` during forum extraction. It prevents comment-class elements from being treated as boilerplate — since on forums, comments *are* the content. Reset to `false` after extraction.

### Module Map

| Path | Purpose |
|------|---------|
| `src/extractor/pipeline.rs` | `handle_text_elem()` — routes elements to per-tag handlers |
| `src/extractor/handlers.rs` | Handlers for paragraphs, titles, lists, tables, images, formatting |
| `src/extractor/pruning.rs` | `prune_unwanted_sections()`, `strip_non_potential_tags()` |
| `src/extractor/fallback.rs` | `baseline()`, `compare_external_extraction()`, `candidate_is_usable()` |
| `src/extractor/state.rs` | `ExtractionState` — tracks processed nodes and potential tags |
| `src/extractor/tags.rs` | Tag name constants and `is_xml_*` predicates |
| `src/selector/content.rs` | `find_content()` and `CONTENT_RULES` for content node detection |
| `src/selector/discard.rs` | Boilerplate class/id patterns |
| `src/metadata/json_ld.rs` | JSON-LD structured data parsing |
| `src/metadata/meta_tags.rs` | OG/Twitter/Dublin Core meta tag extraction |
| `src/metadata/dom_extraction.rs` | DOM fallback for title/author/date |
| `src/page_type/ml.rs` | XGBoost page type classifier (200 trees, 181 features) |
| `src/dom.rs` | DOM abstraction over `dom_query` |
| `src/etree.rs` | Element-tree utilities (tail text, element creation) |
| `src/html_processing.rs` | `doc_cleaning`, `doc_cleaning_with_profile` |
| `src/patterns.rs` | Compile-time `Regex` constants |
| `src/link_density.rs` | Link density tests for content/table selection |
| `src/encoding.rs` | Charset detection + UTF-8 transcoding via `encoding_rs` |
| `src/lru.rs` | LRU cache used by deduplication |
| `src/scoring.rs` | F-score calculation (for benchmarking) |

### DOM Library

`dom_query` (`Document`, `Selection`) is the primary DOM library. `tendril::StrTendril` is used for zero-copy text operations. The internal `dom.rs` wrapper provides helpers like `text_content()`, `inner_html()`, `outer_html()`, `tag_name()`, `clone_document()`.

### Dependencies of Note

- `web-page-classifier` — provides both page type and extraction quality XGBoost models
- `quick_html2md` — HTML-to-Markdown conversion for `output_markdown`
- `html-cleaning` — HTML sanitization
- `dom_query` — DOM querying/manipulation
- `spider` — optional feature for spider-rs integration

## Test Layout

Integration tests in `tests/` cover specific extraction scenarios by name (e.g., `boilerplate_test.rs`, `comments_test.rs`, `precision_test.rs`, `recall_test.rs`). The `tests/integration/` subdirectory has `edge_cases.rs`, `feature_combinations.rs`, and `real_world_articles.rs`.

Inline `#[cfg(test)]` modules exist within `src/options.rs` and `src/page_type/mod.rs` for unit-level tests.
