# CLAUDE.md — Docs Repo

This repo is a [Mintlify](https://mintlify.com) documentation site for Craft/CraftFlow. It contains a Help Center, an API Reference (driven by an OpenAPI spec), and a Security section.

---

## Repo structure

```
docs/
├── docs.json                  # Site-wide config: navigation, theme, colors
├── api/
│   ├── openapi.json           # OpenAPI 3.1 spec — source of truth for API Reference
│   ├── introduction.mdx
│   ├── authentication.mdx
│   └── endpoints/
│       ├── list-users.mdx
│       ├── get-user.mdx
│       └── ...                # One MDX file per API endpoint
├── help/                      # Help Center pages
├── security/                  # Security pages
├── logo/
└── images/
```

---

## Critical: OpenAPI configuration rules

We learned these rules the hard way through multiple production incidents. Follow them exactly.

### Rule 1: `openapi` belongs at the root of `docs.json`, not on the tab

```json
// CORRECT — root level
{
  "$schema": "https://mintlify.com/docs.json",
  "openapi": "api/openapi.json",
  ...
}

// WRONG — tab level (causes every endpoint to appear twice in sidebar)
{
  "navigation": {
    "tabs": [
      {
        "tab": "API Reference",
        "openapi": "api/openapi.json",  // <-- DO NOT PUT IT HERE
        ...
      }
    ]
  }
}
```

**Why:** A tab-level `openapi` field causes Mintlify to auto-generate a sidebar entry for every operation in the spec. Combined with our explicit MDX file navigation, each endpoint appears twice. Root-level `openapi` registers the spec globally (so production builds bundle it) without triggering auto-generation.

### Rule 2: MDX endpoint files use short-form `openapi` frontmatter

Each endpoint file in `api/endpoints/` has minimal frontmatter:

```mdx
---
title: "List Users"
openapi: "GET /users/"
---
```

The short-form `"GET /path/"` works because the spec is declared at the root level of `docs.json`. Do not use the long-form `"api/openapi.json GET /users/"` — it's not necessary and was used in failed intermediate attempts.

### Rule 3: Navigation uses MDX file paths, not operation references

```json
// CORRECT — MDX file paths in pages array
{
  "group": "Users",
  "pages": [
    "api/endpoints/list-users",
    "api/endpoints/get-user"
  ]
}

// WRONG — operation references fail Mintlify's CI schema validation
{
  "group": "Users",
  "pages": [
    "GET /users/",           // fails CI
    "api/openapi.json GET /users/"  // also fails CI
  ]
}
```

The Mintlify docs suggest `"GET /path"` as a valid pages format, but their own CI schema validator rejects it. Use MDX file paths.

---

## Adding a new API endpoint

1. **Update `api/openapi.json`** — add the new path, operation, schema, and example to the spec.

2. **Create an MDX file** at `api/endpoints/{slug}.mdx`:
   ```mdx
   ---
   title: "Get Widget"
   openapi: "GET /widgets/{id}/"
   ---
   ```
   The title and `openapi` value are all that's needed. Mintlify renders the full endpoint page from the spec.

3. **Add to `docs.json` navigation** — add the MDX file path to the appropriate group:
   ```json
   {
     "group": "Widgets",
     "pages": [
       "api/endpoints/list-widgets",
       "api/endpoints/get-widget"
     ]
   }
   ```

4. **Add the group** if it doesn't exist yet — add it inside the `"API Reference"` tab's `groups` array.

---

## Production build vs local dev

Mintlify's local dev server (`mint dev`) resolves spec references at runtime, so blank pages or missing endpoints often don't appear locally. The production builder is stricter:

- **Blank API pages in production** happen when the spec isn't bundled. The root-level `"openapi": "api/openapi.json"` in `docs.json` ensures the spec is always included in production builds.
- **Always test** by checking production after a deploy, not just local dev.

---

## CI validation

Mintlify runs schema validation on `docs.json` when a PR is opened. Key points:

- The `pages` array only accepts file path strings — operation references like `"GET /users/"` are rejected
- The `openapi` field value must be a valid path or URL string
- CI takes ~40s for a valid config; it fails in ~6s for schema errors

---

## Local development

```bash
npm i -g mint   # install CLI once
mint dev        # preview at http://localhost:3000
mint validate   # check for build errors
```

---

## When things break

| Symptom | Cause | Fix |
|---|---|---|
| API pages are blank in production | Spec not bundled | Ensure `"openapi": "api/openapi.json"` is at root of `docs.json` |
| Endpoints appear twice in sidebar | `openapi` is on the tab (not root) | Move `openapi` to root level, remove from tab |
| CI fails instantly (~6s) | Invalid `docs.json` schema | Check `pages` arrays don't contain operation references |
| New endpoint page is 404 | MDX file not in `docs.json` navigation | Add the file path to the correct group in `docs.json` |
| Endpoint page renders blank locally | Check MDX frontmatter `openapi` value matches spec path exactly | |
