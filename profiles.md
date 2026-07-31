---
layout: default
title: Processing Profiles
---

# Processing Profiles

Processing profiles define how Horizon matches, analyzes, filters, enriches,
and renders different kinds of content. They replace global score-threshold and
fixed enrichment-field behavior.

## Directory Layout

Profiles live under `profiles/<id>/`:

```text
profiles/
|-- tech-news/
|   |-- profile.json
|   |-- match.md
|   |-- analysis.md
|   `-- enrichment.md
`-- tech-blog/
    |-- profile.json
    |-- match.md
    |-- analysis.md
    `-- enrichment.md
```

- `profile.json` defines the profile contract.
- `match.md` tells automatic routing what content belongs to the profile.
- `analysis.md` defines the first-pass analysis and scoring rubric.
- `enrichment.md` defines how to write the localized output blocks.

## Built-in Profiles

| Profile | Purpose | Output |
| --- | --- | --- |
| `tech-news` | Timely releases, incidents, research results, and technology-industry developments | Compact summary with optional background and community discussion |
| `tech-blog` | Long-form engineering deep dives, tutorials, investigations, retrospectives, and technical arguments | One structured, multi-paragraph `story` |

The blog profile uses larger input budgets, head-middle-tail sampling, no score
filtering, and no AI topic deduplication. For RSS feeds, pair it with a full-text
extractor so the profile receives the article rather than only the feed excerpt:

```json
{
  "name": "NVIDIA CUDA Technical Blog",
  "url": "https://developer.nvidia.com/blog/tag/cuda/feed/",
  "profile": "tech-blog",
  "content_extractor": "trafilatura"
}
```

Install the optional extractor locally with `uv sync --extra trafilatura`, or
build Docker with `--build-arg EXTRAS=trafilatura`. Extraction
failures fall back to the feed-provided content.

Configure discovery in `data/config.json`:

```json
{
  "processing": {
    "profiles_dir": "profiles",
    "default_profile": "tech-news"
  }
}
```

`default_profile` must name a loaded profile. Horizon fails to start if no
profiles are found or the default does not exist.

## Profile Schema

```json
{
  "id": "tech-news",
  "name": "Technology News",
  "display_names": {
    "zh": "科技新闻"
  },
  "match": "match.md",
  "analysis": "analysis.md",
  "filter": {
    "enabled": true,
    "threshold": 8.0
  },
  "content": {
    "analysis_max_chars": 1000,
    "enrichment_max_chars": 8000,
    "sampling": "prefix"
  },
  "topic_dedup": {
    "enabled": true
  },
  "enrichment": {
    "prompt": "enrichment.md",
    "blocks": [
      {
        "id": "summary",
        "type": "section",
        "tools": []
      },
      {
        "id": "background",
        "type": "section",
        "tools": ["web_search"],
        "optional": true
      },
      {
        "id": "community_discussion",
        "type": "section",
        "tools": [],
        "optional": true
      }
    ]
  }
}
```

| Field | Description |
| --- | --- |
| `id` | Unique profile ID. It starts with a lowercase letter and may contain lowercase letters, digits, `_`, and `-`. |
| `name` | Human-readable name used in the matching catalog. |
| `display_names` | Optional language-keyed names used as digest section headings. |
| `match` | Profile-relative path to the matching prompt. |
| `analysis` | Profile-relative path to the analysis prompt. |
| `filter` | Per-profile score-filter configuration. |
| `content` | Input budgets and long-content sampling strategy for AI stages. |
| `topic_dedup` | Whether this profile participates in AI semantic topic deduplication. |
| `enrichment.prompt` | Profile-relative path to the enrichment prompt. |
| `enrichment.blocks` | Contract for localized output blocks. At least one block is required. |

Block IDs use the same format as profile IDs and must be unique within a
profile. The only supported block `type` is `"section"`. Blocks are required by
default; set `optional` to `true` when they may be omitted.

| Block field | Description |
| --- | --- |
| `id` | Unique block ID within the profile. |
| `type` | Must be `"section"`. |
| `tools` | Tools allowed for this block. Declare it on every block; use `[]` when none are allowed. |
| `optional` | Whether output may omit the block. Defaults to `false`. |

Prompt paths cannot escape their profile directory. Unknown fields are rejected
in profile JSON.

## Source Routing

Set `profile` on a source entry to route its items directly:

```json
{
  "sources": {
    "rss": [
      {
        "name": "Example",
        "url": "https://example.com/feed.xml",
        "profile": "tech-news"
      }
    ]
  }
}
```

Routing follows these rules:

1. An explicit profile ID uses that profile and skips AI matching.
2. A missing `profile` or `"auto"` invokes AI matching against every loaded
   profile's `match.md`.
3. An unknown explicit profile ID is an error.
4. If automatic matching fails, Horizon records the failure and uses
   `processing.default_profile`.

All source types support profile routing. For sources with nested entries, put
`profile` on the item-producing configuration, such as a GitHub entry, RSS feed,
Reddit subreddit or user, or OpenBB watchlist. Top-level single configurations,
such as Hacker News, Twitter, OSS Insight, GDELT, and Google News, carry the
field directly.

## Analysis

After routing, Horizon sends the item to the selected profile's `analysis.md`
prompt. The analysis result contains a nullable 0-10 score, a reason, a
one-sentence summary, and tags. The profile owns the rubric, so profiles can
evaluate different content forms by different standards.

## Filtering

Filtering belongs to the profile:

```json
{
  "filter": {
    "enabled": true,
    "threshold": 8.0
  }
}
```

When enabled, `threshold` is required and must be between 0 and 10. Horizon
keeps items whose analysis score is greater than or equal to that threshold.
When disabled, the item bypasses score filtering and `threshold` may be omitted.

The top-level `collection` configuration controls `time_window_hours`. Optional
balanced digest limits such as `category_groups` and `max_items` belong to the
top-level `digest` configuration and run after profile filtering and topic
deduplication.

## Enrichment Blocks And Tools

`enrichment.blocks` defines the exact block IDs available to output.
Required blocks must be present; optional blocks can be omitted when they add no
useful content. Generated output cannot contain unknown or duplicate blocks.

Tools are allowed per block through its `tools` array. The only built-in tool is
`web_search`, and a block may use it only when that block explicitly declares
`"tools": ["web_search"]`. Use an empty array for blocks that need no tools.
Unknown tools are rejected when profiles are initialized.

## Content Selection

Profiles can control how much source content each AI stage receives:

```json
{
  "content": {
    "analysis_max_chars": 16000,
    "enrichment_max_chars": 24000,
    "sampling": "head-middle-tail"
  }
}
```

`sampling` accepts `"prefix"` or `"head-middle-tail"`. Prefix sampling preserves
the compact behavior used by news profiles. Head-middle-tail sampling keeps the
opening, a middle excerpt, and the conclusion of long-form content. Both limits
must be between 500 and 100000 characters.

## Topic Deduplication

AI topic deduplication can be disabled for profiles where different treatments
of the same subject should remain separate:

```json
{
  "topic_dedup": {
    "enabled": false
  }
}
```

This does not disable conservative cross-source URL deduplication. Items with
the same normalized URL and requested Profile are still merged before analysis;
the same URL routed to different Profiles remains separate.

Search-backed statements cite tool results through source references. Horizon
rejects references that were not returned by a tool call.

## Localized Output

For each language in `ai.languages`, enrichment produces a localized artifact
with:

- a title;
- an optional lead paragraph;
- the profile's required and applicable optional section blocks; and
- cited external sources referenced by those blocks.

The Markdown briefing renders the localized title and lead, each block under
its localized heading, and a sources list when external references were used.
Items are grouped by Profile: the briefing title is H1, localized Profile names
are H2 sections, items are H3 headings, and artifact blocks are H4 headings.
