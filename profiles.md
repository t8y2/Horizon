---
layout: default
title: Processing Profiles
---

# Processing Profiles

Processing profiles define how Horizon matches, analyzes, enriches, and renders
different kinds of content. User preferences such as score thresholds and topic
deduplication live in the runtime configuration so profiles remain stable.

## Directory Layout

Profiles live under `profiles/<id>/`:

```text
profiles/
|-- finance-news/
|   |-- profile.json
|   |-- match.md
|   |-- analysis.md
|   `-- enrichment.md
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
| `finance-news` | Macroeconomics, markets, company finance, and economically material policy | Concise summary, necessary background, and optional direct impact |
| `tech-news` | Timely releases, incidents, research results, and technology-industry developments | Compact summary and background with optional impact and community discussion |
| `tech-blog` | Long-form engineering deep dives, tutorials, investigations, retrospectives, and technical arguments | Required background, solution, and takeaway sections |

The blog profile uses larger input budgets and head-middle-tail sampling. For RSS
feeds, pair it with a full-text extractor so the profile receives the article
rather than only the feed excerpt:

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

## Contributing a Profile

A profile is a reusable editorial policy for a content domain, not a user
account, source list, or collection of personal interests. It tells Horizon:

1. which items belong to the domain (`match.md`),
2. how to evaluate and score them (`analysis.md`), and
3. which content blocks to produce and how to write them (`profile.json` and
   `enrichment.md`).

To contribute a built-in profile, add `profiles/<id>/` with those four files and
open a focused pull request. No Python changes are normally required.

Before submitting, check that the profile:

- covers a clear content domain that is useful to more than one source list or
  individual user,
- has a meaningful routing, evaluation, or output difference from existing
  profiles,
- states both what belongs and what does not belong in `match.md`,
- defines a concrete 0-10 rubric in `analysis.md`,
- keeps generated blocks specific, non-overlapping, and grounded in supplied
  content or declared tools,
- contains no credentials, private sources, or user-specific thresholds, and
- passes `uv run pytest tests/test_profiles.py tests/test_prompting.py -q`.

Built-in profiles are loaded as automatic-routing candidates, so maintainers may
ask contributors to narrow ambiguous matching rules or clarify overlap before a
profile is merged.

Configure discovery in `data/config.json`:

```json
{
  "processing": {
    "profiles_dir": "profiles",
    "default_profile": "tech-news",
    "profile_settings": {
      "tech-news": {
        "threshold": 7.0,
        "topic_dedup": true
      },
      "tech-blog": {
        "threshold": 4.0,
        "topic_dedup": false
      }
    }
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
  "content": {
    "analysis_max_chars": 1000,
    "enrichment_max_chars": 8000,
    "sampling": "prefix"
  },
  "enrichment": {
    "prompt": "enrichment.md",
    "blocks": [
      {
        "id": "summary",
        "type": "section",
        "tools": [],
        "primary": true
      },
      {
        "id": "background",
        "type": "section",
        "tools": ["web_search"]
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
| `content` | Input budgets and long-content sampling strategy for AI stages. |
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
| `primary` | Render this required block directly below the item title without a block heading. At most one block may be primary. Defaults to `false`. |

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

Set `profile` to an array to restrict automatic matching to a candidate subset:

```json
{
  "channel": "zaihuapd",
  "profile": ["tech-news", "finance-news"]
}
```

Routing follows these rules:

1. An explicit profile ID uses that profile and skips AI matching.
2. A missing `profile` or `"auto"` invokes AI matching against every loaded
   profile's `match.md`.
3. A non-empty profile array invokes AI matching only against those candidates.
4. Unknown, duplicate, blank, or `"auto"` entries in a candidate array are errors.
5. If candidate matching fails, Horizon uses `processing.default_profile` when
   it is a candidate, otherwise the first candidate. Unrestricted matching falls
   back to `processing.default_profile`.

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

Filtering is a user preference configured by profile ID under
`processing.profile_settings`:

```json
{
  "processing": {
    "profile_settings": {
      "tech-news": {
        "threshold": 8.0
      }
    }
  }
}
```

`threshold` must be between 0 and 10. Horizon keeps items whose analysis score
is greater than or equal to that threshold. Set it to `null` or omit settings for
a profile to bypass score filtering. An MCP threshold supplied for a single
operation takes precedence over these configured values.

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

Tool planning receives each block's required or optional status. For required
blocks with tools, it uses a tool unless the source already provides enough
evidence; tool failures do not make the block optional.

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

AI topic deduplication is also a runtime preference. Disable it for profiles
where different treatments of the same subject should remain separate:

```json
{
  "processing": {
    "profile_settings": {
      "tech-blog": {
        "topic_dedup": false
      }
    }
  }
}
```

`topic_dedup` defaults to `true` when it or the profile's settings are omitted.

This does not disable conservative cross-source URL deduplication. Items with
the same normalized URL and requested Profile are still merged before analysis;
the same URL routed to different Profiles remains separate.

Search-backed statements cite tool results through source references. Horizon
rejects references that were not returned by a tool call.

## Localized Output

For each language in `ai.languages`, enrichment produces a localized artifact
with:

- a title;
- the profile's required and applicable optional section blocks; and
- cited external sources referenced by those blocks.

Artifacts generated for `zh` are normalized to Simplified Chinese before they
are stored and rendered, including older artifacts read during rendering.

The Markdown briefing renders a block marked `primary` directly below the item
title and before the source line, without a redundant block heading. Profiles
without a primary block show the source first and then render every block under
its bold localized title on the same line as its content. External references
follow the blocks when used. Items
are grouped by Profile: the briefing title is H1, localized Profile names are H2
sections, and items are H3 headings. Set `digest.profile_order` to control the H2
section order; a non-empty list must contain every loaded Profile exactly once.
