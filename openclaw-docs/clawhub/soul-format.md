> ## Documentation Index
> Fetch the complete documentation index at: https://docs.openclaw.ai/llms.txt
> Use this file to discover all available pages before exploring further.

# Soul format

# Soul format

## On disk

A soul is a single file:

* `SOUL.md` (or `soul.md`)

For now, onlycrabs.ai rejects any extra files.

## `SOUL.md`

* Markdown with optional YAML frontmatter.
* The server extracts metadata from frontmatter during publish.
* `description` is used as the soul summary in the UI/search.

## Limits

* Total bundle size: 50MB.
* Embedding text includes `SOUL.md` only.

## Slugs

* Derived from folder name by default.
* Must be lowercase and URL-safe: `^[a-z0-9][a-z0-9-]*$`.

## Versioning + tags

* Each publish creates a new version (semver).
* Tags are string pointers to a version; `latest` is commonly used.
