# MailBeast Public API docs (Mintlify)

Mintlify docs site for the public `/v1` API. The reference pages are
**auto-generated from `openapi.json`**, which is exported from the NestJS
backend's live spec - controllers/DTOs are the single source of truth.

## Regenerate the spec

The spec is produced by the backend's public OpenAPI document
(`SwaggerModule.createDocument(..., { include: [PublicApiModule] })`, served at
`/v1/docs-json`). With the API running locally:

```bash
cd ../mailbeast-ai-backend && npm run openapi:public   # → ../api-docs/openapi.json
```

Re-run this whenever a `/v1` controller or DTO changes. Every public endpoint
must carry `@ApiTags`, `@ApiOperation`, `@ApiResponse`, `@ApiBearerAuth('ApiKey')`
and typed request/response DTOs so the reference renders richly.

## Preview locally

```bash
npx mint@latest dev --port 3333   # → http://localhost:3333
```

## Structure

- `docs.json` - Mintlify config (theme, nav tabs, OpenAPI reference).
- `openapi.json` - generated spec (committed so the hosted build has it).
- `introduction.mdx` / `authentication.mdx` / `quickstart.mdx` - Guides tab.

## Deploy

Connect this folder to the Mintlify dashboard; it auto-deploys on push.
