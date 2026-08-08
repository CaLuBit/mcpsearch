# MCPSearch Quick Reference

## Live Services

```
Frontend:  https://mcpsearch.com
API:       https://api.mcpsearch.com
CLI:       npm install -g @mcpsearch/cli
```

---

## Common Commands

### Local Development
```bash
# Install deps
pnpm install

# Build shared (required first)
pnpm --filter @mcpsearch/shared build

# Run API locally (port 4000)
pnpm --filter @mcpsearch/api dev

# Run Web locally (port 3000)
pnpm --filter @mcpsearch/web dev

# Run all
pnpm dev
```

### Deploy API
```bash
# Build & deploy to App Runner
./infrastructure/deploy-apprunner.sh

# Or trigger redeployment
aws apprunner start-deployment \
  --service-arn arn:aws:apprunner:us-east-1:127813366915:service/mcpsearch-api/0d9a4424649a4ba28e51886998d257d4
```

### Ingest MCP Packages
```bash
# Ingest from npm and GitHub
pnpm --filter @mcpsearch/worker seed --sources=npm,github --limit=100
```

### Publish CLI
```bash
cd packages/cli
pnpm build
npm publish --access public
```

---

## AWS Resources

| Resource | Name/ID |
|----------|---------|
| App Runner | `mcpsearch-api` |
| DynamoDB Table | `mcpsearch-packages` |
| ECR Repo | `mcpsearch-api` |
| Amplify App | `d1c2vsrknw7lwy` |
| IAM Role | `mcpsearch-api-role` |
| Route53 Zone | `mcpsearch.com` |

---

## API Endpoints

```bash
# Health
curl https://api.mcpsearch.com/health

# List packages
curl "https://api.mcpsearch.com/v1/packages?limit=10"

# Popular
curl https://api.mcpsearch.com/v1/packages/popular

# Search
curl "https://api.mcpsearch.com/v1/search?q=filesystem"

# Single package (encode @ as %40)
curl https://api.mcpsearch.com/v1/packages/mcp-hello-world
curl "https://api.mcpsearch.com/v1/packages/%40supabase%2Fmcp-server-supabase"
```

---

## Key Files

| Purpose | File |
|---------|------|
| API Routes | `apps/api/src/routes/packages.ts` |
| DynamoDB Repo | `apps/api/src/repositories/package-repository.ts` |
| Ingestion | `apps/worker/src/handlers/mcp-ingestion.ts` |
| Amplify Config | `apps/web/amplify.yml` |
| CLI Config | `packages/cli/src/core/config-manager.ts` |
| Package Types | `packages/shared/src/types/package.ts` |

---

## Environment Variables

### API (App Runner)
```
NODE_ENV=production
DYNAMODB_TABLE_NAME=mcpsearch-packages
AWS_REGION=us-east-1
CORS_ORIGINS=https://mcpsearch.com,https://www.mcpsearch.com
```

### Worker (.env)
```
DYNAMODB_TABLE=mcpsearch-packages
AWS_REGION=us-east-1
# DYNAMODB_ENDPOINT commented out for real AWS
```

---

## Troubleshooting

**API returns seed data?**
→ Check `USE_SEED_DATA` is not set, `NODE_ENV=production`

**Amplify build fails?**
→ Check `.npmrc` exists with `node-linker=hoisted`

**DynamoDB connection refused?**
→ Check `DYNAMODB_ENDPOINT` is NOT set (or commented out)

**CLI can't reach API?**
→ Check `packages/cli/src/core/config-manager.ts` has `https://api.mcpsearch.com`

---

## What's Done vs Pending

### Done
- [x] Frontend deployed (Amplify + CloudFront)
- [x] API deployed (App Runner)
- [x] Custom domains (mcpsearch.com, api.mcpsearch.com)
- [x] DynamoDB with 285 real MCP packages
- [x] CLI published to npm
- [x] Basic search working

### Pending
- [ ] OpenSearch for better search
- [ ] Redis caching
- [ ] User auth (Cognito)
- [ ] Package publishing flow
- [ ] S3 for package storage
- [ ] CI/CD pipeline
