# CLAUDE.md — Docs Repo

This repo is a [Mintlify](https://mintlify.com) documentation site for Craft/CraftFlow. It has three sections: Help Center, API Reference (driven by an OpenAPI spec), and Security.

---

## Repo structure

```
docs/
├── docs.json                  # Site config: navigation, theme, colors
├── api/
│   ├── openapi.json           # OpenAPI 3.1 spec — source of truth for API Reference
│   ├── introduction.mdx       # Non-spec pages, listed explicitly in navigation
│   ├── authentication.mdx
│   └── endpoints/             # Empty — endpoint pages are auto-generated from the spec
├── help/                      # Help Center pages
├── security/                  # Security pages
├── logo/
└── images/
```

---

## How API Reference navigation works

Mintlify auto-generates sidebar entries for every operation in `api/openapi.json` because of the `openapi` field on the API Reference tab:

```json
{
  "tab": "API Reference",
  "openapi": "api/openapi.json",
  "groups": [
    {
      "group": "Overview",
      "pages": ["api/introduction", "api/authentication"]
    }
  ]
}
```

Operations are grouped in the sidebar by their `tags` field in the spec. The tag names become the group headings. The `api/endpoints/` directory is intentionally empty — do not create MDX files there.

---

## Adding a new API endpoint

1. Add the operation to `api/openapi.json` — set `summary`, `tags`, parameters, request/response schemas, and an example.
2. That's it. Mintlify auto-generates the page.

No MDX file needed. No `docs.json` update needed.

To add a new **group** (e.g. "Webhooks"), just use a new tag value in the spec and operations will appear under that heading automatically.

---

## The one rule that must never be broken

**Do not create MDX files in `api/endpoints/` and do not add endpoint groups to `docs.json`.**

Doing either will cause every endpoint to appear twice in the sidebar. Mintlify auto-generates entries for all spec operations regardless — any explicit MDX file or navigation group for the same operation produces a duplicate.

We learned this through multiple production incidents. The root cause every time: two sources of truth for the same pages.

**Corollary: never remove `"openapi": "api/openapi.json"` from the API Reference tab.**

This has happened when editing `docs.json` for unrelated reasons (e.g. adding Help Center nav groups) — the tab-level `openapi` field gets accidentally dropped, and at the same time explicit endpoint groups get re-added. The result: endpoint pages 404 because the MDX files don't exist and auto-generation is disabled. Always verify the API Reference tab still has `"openapi": "api/openapi.json"` after any `docs.json` edit.

---

## Adding non-spec pages (guides, authentication docs, etc.)

For pages that aren't in the OpenAPI spec (like `api/introduction.mdx` and `api/authentication.mdx`), create the MDX file and add it to the "Overview" group in `docs.json` manually:

```json
{
  "group": "Overview",
  "pages": [
    "api/introduction",
    "api/authentication",
    "api/your-new-page"
  ]
}
```

---

## Local development

```bash
npm i -g mint   # install CLI once
mint dev        # preview at http://localhost:3000
mint validate   # check for build errors
```

---

## Known Mintlify gotchas

| Symptom | Cause | Fix |
|---|---|---|
| Endpoints appear twice in sidebar | MDX files in `api/endpoints/` exist alongside spec auto-generation | Delete the MDX files; let the spec drive everything |
| Endpoints appear twice in sidebar | Root-level `openapi` in `docs.json` AND tab-level `openapi` | Only ever put `openapi` on the tab, not at root level |
| API pages are blank in production | `openapi` not declared on the tab | Ensure `"openapi": "api/openapi.json"` is on the API Reference tab in `docs.json` |
| Endpoint pages 404 | `openapi` removed from tab + explicit endpoint groups re-added pointing to deleted MDX files | Remove the endpoint groups; restore `"openapi": "api/openapi.json"` on the tab |
| New endpoint missing from sidebar | Operation lacks a `tags` value in the spec | Add the appropriate tag to the operation |
| CI fails instantly (~6s) | Invalid `docs.json` schema | The `pages` array only accepts file paths — `"GET /path"` strings are rejected by the validator even though the Mintlify docs suggest that format |
