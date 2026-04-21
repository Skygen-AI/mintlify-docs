# Skygen Documentation

Public documentation for Skygen — an AI agent automation platform. Deployed via Mintlify.

## Structure

```
mintlify-docs/
├── docs.json               # Mintlify site config (nav, theme, openapi wiring)
├── index.mdx               # Landing page
├── quickstart.mdx          # 5-minute onboarding
├── errors.mdx              # Stripe-style error reference
├── concepts/               # Core concept explanations
│   ├── devices.mdx
│   ├── chat-sessions.mdx
│   ├── agents.mdx
│   ├── confirmations.mdx
│   ├── cloud-pc.mdx
│   ├── billing.mdx
│   ├── safety.mdx
│   └── mcp-integrations.mdx
├── guides/                 # How-to pages
│   ├── authentication.mdx
│   ├── connecting-integrations.mdx
│   ├── custom-connectors.mdx
│   ├── triggers.mdx
│   ├── knowledge-base.mdx
│   ├── scheduled-tasks.mdx
│   └── screenshots.mdx
├── api-reference/
│   ├── introduction.mdx    # API reference intro
│   └── (auto-generated from openapi.json by Mintlify at build time)
├── openapi.json            # Exported from Backend (DO NOT edit by hand)
└── AUDIT.md                # Backend annotation audit + fix roadmap
```

## Local development

```bash
npm i -g mint
mint dev
```

Opens a local preview at http://localhost:3000. Changes to MDX files hot-reload.

## OpenAPI reference

The API Reference tab is **auto-generated from `openapi.json`**. That file is exported from the FastAPI backend.

### Re-generate locally

```bash
cd ../Backend
poetry run python scripts/export_openapi.py
```

The script writes `../mintlify-docs/openapi.json`.

### CI sync

`Backend/.github/workflows/sync-openapi-docs.yml` runs on every push to `main`/`develop` that touches routers, schemas, or the export script. It exports `openapi.json` and commits it to this repo. A `DOCS_REPO_TOKEN` secret (a PAT with `contents: write` on this repo) must be configured in the Backend repo.

## Annotation audit

See [AUDIT.md](./AUDIT.md) for the prioritized list of Backend files that need annotation work to reach Stripe-level API docs. The short version: we are at ~20% coverage today; target is ≥ 90%.

## Deployment

Mintlify deploys automatically on push to `main` via the GitHub App installed on `skygen-ai/mintlify-docs`. Preview branches get their own URLs for review before merge.

## Writing conventions

Follow the Mintlify skill installed at `.agents/skills/mintlify/SKILL.md`:

- Second-person voice ("you")
- Sentence case headings
- No marketing language ("powerful", "seamless", "robust")
- Code blocks must have language tags
- Internal links are root-relative without extension: `/concepts/devices`
- Add new pages to `docs.json` navigation or they stay hidden

## AI-assisted writing

This repo has 3 Mintlify skills installed (`mintlify`, `mintlify-api`, `mintlify-docs`) via `npx skills add`. They work with Claude Code, Cursor, Copilot, Windsurf, and others. See `.agents/skills/` for the skill content.

## Support

- Docs issues: open a PR or issue on `skygen-ai/mintlify-docs`
- Product support: support@skygen.ai
