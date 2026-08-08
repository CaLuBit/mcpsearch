# MCPSearch Deployment Status

> Last updated: March 2026

## Current Architecture

```
                          ┌─────────────────┐
                          │    Route 53     │
                          │  mcpsearch.com  │
                          └────────┬────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
    ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
    │   CloudFront    │  │   App Runner    │  │      npm        │
    │ (d3gp1f...net)  │  │ api.mcpsearch   │  │  @mcpsearch/cli │
    └────────┬────────┘  └────────┬────────┘  └─────────────────┘
             │                    │
             ▼                    ▼
    ┌─────────────────┐  ┌─────────────────┐
    │  AWS Amplify    │  │    DynamoDB     │
    │   Next.js SSR   │  │ mcpsearch-pkgs  │
    │  (d1c2vsr...)   │  │  (285 items)    │
    └─────────────────┘  └─────────────────┘
             │
             ▼
    ┌─────────────────┐
    │      ECR        │
    │ mcpsearch-api   │
    └─────────────────┘
```

**What's Deployed:**
- Frontend: Amplify (Next.js SSR) → CloudFront → mcpsearch.com
- API: App Runner → api.mcpsearch.com
- Database: DynamoDB (mcpsearch-packages)
- CLI: Published to npm

**Not Yet Deployed:**
- OpenSearch (search uses DynamoDB queries)
- ElastiCache/Redis (no caching)
- Cognito (no auth)
- S3 (no package storage)

---

## Live URLs

| Service | URL | Status |
|---------|-----|--------|
| Frontend | https://mcpsearch.com | Live |
| Frontend (www) | https://www.mcpsearch.com | Live |
| API | https://api.mcpsearch.com | Live |
| CLI | `npm install -g @mcpsearch/cli` | Published v0.1.0 |

---

## AWS Infrastructure

### Account & Region
- **AWS Account ID**: 127813366915
- **Region**: us-east-1
- **IAM User**: junibark

### Services Deployed

#### App Runner (API)
- **Service Name**: mcpsearch-api
- **Service ARN**: `arn:aws:apprunner:us-east-1:127813366915:service/mcpsearch-api/0d9a4424649a4ba28e51886998d257d4`
- **Service URL**: `2bxybmyptk.us-east-1.awsapprunner.com`
- **Custom Domain**: api.mcpsearch.com
- **Instance Role**: `arn:aws:iam::127813366915:role/mcpsearch-api-role`
- **Image**: `127813366915.dkr.ecr.us-east-1.amazonaws.com/mcpsearch-api:latest`

**Environment Variables**:
```
NODE_ENV=production
LOG_LEVEL=info
AWS_REGION=us-east-1
DYNAMODB_TABLE_NAME=mcpsearch-packages
CORS_ORIGINS=https://mcpsearch.com,https://www.mcpsearch.com,http://localhost:3000
RATE_LIMIT_ENABLED=true
RATE_LIMIT_MAX_REQUESTS=100
RATE_LIMIT_WINDOW_MS=60000
```

#### DynamoDB
- **Table Name**: `mcpsearch-packages`
- **Primary Key**: `PK` (String) + `SK` (String)
- **GSI1**: `GSI1PK` + `GSI1SK` (for category queries)
- **GSI2**: `GSI2PK` + `GSI2SK` (for publisher queries)
- **Items**: ~285 packages

**Key Patterns**:
- Package: `PK=PKG#<packageId>`, `SK=META`
- Version: `PK=PKG#<packageId>`, `SK=VERSION#<version>`
- Category GSI: `GSI1PK=CATEGORY#<category>`, `GSI1SK=PKG#<packageId>`
- Publisher GSI: `GSI2PK=PUBLISHER#<publisherId>`, `GSI2SK=PKG#<packageId>`

#### ECR Repository
- **Repository**: `mcpsearch-api`
- **URI**: `127813366915.dkr.ecr.us-east-1.amazonaws.com/mcpsearch-api`

#### AWS Amplify (Frontend)
- **App Name**: mcpsearch-web
- **App ID**: `d1c2vsrknw7lwy`
- **Default Domain**: `d1c2vsrknw7lwy.amplifyapp.com`
- **Custom Domains**: mcpsearch.com, www.mcpsearch.com
- **Build Config**: `apps/web/amplify.yml`
- **CloudFront**: `d3gp1f0u0f3jj4.cloudfront.net`
- **Branch**: main (auto-deploy enabled)

#### Route53 (DNS)
- **Hosted Zone**: mcpsearch.com

**DNS Records**:
| Name | Type | Value |
|------|------|-------|
| mcpsearch.com | A (Alias) | d3gp1f0u0f3jj4.cloudfront.net |
| www.mcpsearch.com | CNAME | d3gp1f0u0f3jj4.cloudfront.net |
| api.mcpsearch.com | CNAME | 2bxybmyptk.us-east-1.awsapprunner.com |
| ACM validation records | CNAME | *.acm-validations.aws |

#### IAM Roles
- **mcpsearch-api-role**: App Runner instance role with DynamoDB access
  - Policy: `DynamoDBAccess` (GetItem, PutItem, UpdateItem, DeleteItem, Query, Scan, BatchGetItem, BatchWriteItem)

---

## Package Data

### MCP Packages Ingested
- **Total**: 285 packages
- **Sources**: npm (231), GitHub modelcontextprotocol/servers (1)
- **Ingestion Script**: `pnpm --filter @mcpsearch/worker seed`

**Notable Packages**:
- @modelcontextprotocol/server-filesystem
- @supabase/mcp-server-supabase
- @heroku/mcp-server
- @sentry/mcp-server
- @notionhq/notion-mcp-server
- @salesforce/mcp
- @azure/mcp
- figma-mcp
- tavily-mcp

### Running Ingestion
```bash
cd /Users/henok/Downloads/mcpsearch

# Update worker .env (already configured)
# DYNAMODB_TABLE=mcpsearch-packages
# DYNAMODB_ENDPOINT commented out (uses real AWS)

# Run ingestion
pnpm --filter @mcpsearch/worker seed --sources=npm,github --limit=100
```

---

## Development Setup

### Prerequisites
- Node.js 20+
- pnpm 9+
- AWS CLI configured with junibark credentials

### Local Development
```bash
# Install dependencies
pnpm install

# Build shared package first
pnpm --filter @mcpsearch/shared build

# Start API locally
pnpm --filter @mcpsearch/api dev

# Start Web locally
pnpm --filter @mcpsearch/web dev

# Start Worker locally
pnpm --filter @mcpsearch/worker dev
```

### Important Config Files

**`.npmrc`** (root - required for Amplify):
```
node-linker=hoisted
shamefully-hoist=true
strict-peer-dependencies=false
auto-install-peers=true
```

**`apps/web/next.config.js`** (required for monorepo):
```javascript
outputFileTracingRoot: path.join(__dirname, '../../'),
```

---

## Deployment Commands

### Deploy API to App Runner
```bash
cd /Users/henok/Downloads/mcpsearch

# Build and push Docker image
./infrastructure/deploy-apprunner.sh

# Or manually:
docker build -f infrastructure/docker/api.Dockerfile -t mcpsearch-api .
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 127813366915.dkr.ecr.us-east-1.amazonaws.com
docker tag mcpsearch-api:latest 127813366915.dkr.ecr.us-east-1.amazonaws.com/mcpsearch-api:latest
docker push 127813366915.dkr.ecr.us-east-1.amazonaws.com/mcpsearch-api:latest

# Trigger App Runner deployment
aws apprunner start-deployment --service-arn arn:aws:apprunner:us-east-1:127813366915:service/mcpsearch-api/0d9a4424649a4ba28e51886998d257d4
```

### Deploy Frontend (Amplify)
- Amplify auto-deploys on push to main branch
- Manual trigger: AWS Console > Amplify > mcpsearch > Redeploy

### Publish CLI to npm
```bash
cd packages/cli
pnpm build
npm publish --access public
```

---

## API Endpoints

### Working Endpoints (Verified)
```bash
# List packages
curl "https://api.mcpsearch.com/v1/packages?limit=10"

# Get popular packages
curl "https://api.mcpsearch.com/v1/packages/popular"

# Get recent packages
curl "https://api.mcpsearch.com/v1/packages/recent"

# Get featured packages
curl "https://api.mcpsearch.com/v1/packages/featured"

# Get single package (URL-encode @ as %40)
curl "https://api.mcpsearch.com/v1/packages/mcp-hello-world"
curl "https://api.mcpsearch.com/v1/packages/%40modelcontextprotocol%2Fserver-filesystem"

# Search packages
curl "https://api.mcpsearch.com/v1/search?q=github"

# Health check
curl "https://api.mcpsearch.com/health"
```

---

## Known Issues & TODOs

### Completed
- [x] Amplify SSR deployment (fixed with `.npmrc` and `outputFileTracingRoot`)
- [x] Custom domain for frontend (mcpsearch.com)
- [x] Custom domain for API (api.mcpsearch.com)
- [x] DynamoDB table creation
- [x] MCP package ingestion (285 packages)
- [x] API connected to DynamoDB
- [x] CLI published to npm

### Pending
- [ ] OpenSearch setup for better search (currently using DynamoDB scan)
- [ ] Redis/ElastiCache for caching
- [ ] Cognito setup for authentication
- [ ] S3 bucket for package storage
- [ ] User registration & publishing flow
- [ ] CI/CD pipeline (CodePipeline)
- [ ] Monitoring & CloudWatch dashboards
- [ ] Security audit & WAF rules

### Notes
- Package names with `@` must be URL-encoded in API calls (`%40`)
- Search currently uses basic DynamoDB queries (falls back to seed data search)
- No authentication/publishing enabled yet

---

## File Locations

### Key Source Files
| File | Purpose |
|------|---------|
| `apps/api/src/routes/packages.ts` | Package API endpoints |
| `apps/api/src/services/package-service.ts` | Package business logic |
| `apps/api/src/repositories/package-repository.ts` | DynamoDB operations |
| `apps/api/src/lib/seed-data.ts` | Seed data fallback |
| `apps/api/src/lib/config.ts` | Environment config |
| `apps/worker/src/handlers/mcp-ingestion.ts` | Package ingestion |
| `apps/worker/src/scripts/seed-mcps.ts` | Ingestion script |
| `apps/web/amplify.yml` | Amplify build config |
| `packages/cli/src/core/config-manager.ts` | CLI config (API URL) |
| `packages/shared/src/types/package.ts` | Package type definitions |

### Environment Files
| File | Purpose |
|------|---------|
| `apps/api/.env` | API local dev config |
| `apps/worker/.env` | Worker config (DynamoDB settings) |
| `apps/web/.env.local` | Web local dev config |

---

## Troubleshooting

### Amplify Build Fails
1. Ensure `.npmrc` exists in root with `node-linker=hoisted`
2. Ensure `outputFileTracingRoot` is set in `next.config.js`
3. Check `amplify.yml` uses `buildPath: /` and `baseDirectory: apps/web/.next`

### DynamoDB Connection Issues
1. Check `DYNAMODB_ENDPOINT` is NOT set (for real AWS)
2. Verify `DYNAMODB_TABLE_NAME=mcpsearch-packages`
3. Ensure IAM role has DynamoDB permissions

### API Returns Seed Data Instead of DynamoDB
1. Check `USE_SEED_DATA` is NOT set to `true`
2. Check `NODE_ENV=production`
3. Verify DynamoDB table exists and has data

### CLI Can't Connect to API
1. Check `packages/cli/src/core/config-manager.ts` has correct API URL
2. Verify https://api.mcpsearch.com is accessible
