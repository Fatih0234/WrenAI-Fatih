# WrenAI with Google Gemini Integration

This repository contains a custom configuration for running WrenAI with Google Gemini 2.0 Flash instead of OpenAI models.

## Changes from Default Setup

### Model Configuration

**LLM Model:**
- Changed from: OpenAI GPT models
- Changed to: `gemini/gemini-2.0-flash` (stable version)
- Why: The experimental `gemini-2.0-flash-exp` has strict rate limits (10 requests/min)

**Embedding Model:**
- Changed from: `text-embedding-3-large` (3072 dimensions)
- Changed to: `gemini/text-embedding-004` (768 dimensions)
- Critical: This requires Qdrant index recreation

### Configuration Files

1. **`docker/config.gemini.yaml`** - Main Gemini configuration
   - Gemini 2.0 Flash for LLM tasks
   - Gemini 2.0 Flash with JSON mode for chart generation
   - Gemini text-embedding-004 for embeddings
   - Qdrant with 768-dimensional vectors

2. **`docker/config.yaml.openai.backup`** - Original OpenAI setup
   - Backup of the working OpenAI configuration
   - Uses text-embedding-3-large (3072 dimensions)

3. **`docker/config.yaml`** - Active configuration (gitignored)
   - Copy `config.gemini.yaml` to `config.yaml` to use Gemini
   - This file is gitignored to prevent committing sensitive settings

## Setup Instructions

### 1. Prerequisites

- Docker and Docker Compose installed
- Google AI Studio API key (from https://aistudio.google.com/apikey)
- (Optional) Langfuse account for observability (https://cloud.langfuse.com)

### 2. Environment Variables

Create or update `docker/.env` with:

```bash
# Google Gemini API Key (Required)
GEMINI_API_KEY=your_gemini_api_key_here

# Langfuse Integration (Optional - for observability)
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
```

### 3. Activate Gemini Configuration

```bash
# Copy the Gemini config to the active config file
cp docker/config.gemini.yaml docker/config.yaml

# Important: Clear Qdrant volumes when switching embedding models
docker compose -f docker/docker-compose.yaml down
docker volume rm wrenai_data

# Start services
docker compose -f docker/docker-compose.yaml up -d
```

### 4. Verify Setup

Check that services started successfully:

```bash
# View logs
docker compose -f docker/docker-compose.yaml logs -f wren-ai-service

# Look for these log entries:
# LANGFUSE_ENABLE: True
# LANGFUSE_HOST: https://cloud.langfuse.com
# Service version: X.X.X
```

### 5. Re-index Your Data

After switching to Gemini embeddings, you need to re-deploy your MDL (semantic model) to trigger re-indexing:

1. Open WrenAI UI (http://localhost:3000)
2. Go to your data model
3. Click "Deploy" to re-index with new embeddings

## Why Qdrant Volume Must Be Cleared

**Embedding Dimension Mismatch:**
- OpenAI `text-embedding-3-large`: 3072 dimensions
- Gemini `text-embedding-004`: 768 dimensions

Qdrant stores vectors with a fixed dimension per collection. When you switch embedding models, the old 3072-dimensional vectors are incompatible with the new 768-dimensional embeddings.

**Setting `recreate_index: true` alone is NOT enough** - it only recreates the index structure on startup, not the actual embedded data. You must:

1. Clear the Qdrant volume: `docker volume rm wrenai_data`
2. Restart services with `recreate_index: true`
3. Re-deploy your MDL to trigger data re-indexing

## Troubleshooting

### Rate Limit Errors

**Error:** `RESOURCE_EXHAUSTED` - quota limit exceeded

**Solution:** You're using the experimental model. Switch to stable:
- Change `gemini/gemini-2.0-flash-exp` → `gemini/gemini-2.0-flash`

### Chart Generation Stuck at "User Intent Recognized"

**Cause:** Embedding dimension mismatch between old (OpenAI) and new (Gemini) embeddings in Qdrant

**Solution:**
```bash
docker compose -f docker/docker-compose.yaml down
docker volume rm wrenai_data
docker compose -f docker/docker-compose.yaml up -d
```

Then re-deploy your MDL in the UI.

### SQL Generation Timeout: "Request timed out: 30 seconds"

**Cause:** The default `engine_timeout: 30` is too short for complex SQL validation

**Why it happens:**
- Gemini generates complex SQL with CTEs (Common Table Expressions)
- SQL validation runs a dry-run execution against your database
- Complex queries with 6+ table joins take >30 seconds to validate
- The timeout occurs during validation, NOT during Gemini API calls

**Solution:** Increase `engine_timeout` in your config file (already done in this repository):
```yaml
settings:
  engine_timeout: 120  # Changed from 30 to 120 seconds
```

Then restart the service:
```bash
docker compose -f docker/docker-compose.yaml restart wren-ai-service
```

### Langfuse Traces Not Appearing

**Check:**
1. Environment variables are set: `LANGFUSE_PUBLIC_KEY` and `LANGFUSE_SECRET_KEY`
2. Config has `langfuse_enable: true`
3. API keys are valid in Langfuse dashboard
4. Network connectivity to `cloud.langfuse.com`

**Verify in logs:**
```bash
docker compose -f docker/docker-compose.yaml logs wren-ai-service | grep LANGFUSE
# Should show:
# LANGFUSE_ENABLE: True
# LANGFUSE_HOST: https://cloud.langfuse.com
```

## Switching Back to OpenAI

If you need to revert to OpenAI:

```bash
# Restore OpenAI config
cp docker/config.yaml.openai.backup docker/config.yaml

# Clear Qdrant (required - different embedding dimensions)
docker compose -f docker/docker-compose.yaml down
docker volume rm wrenai_data

# Update .env to use OPENAI_API_KEY instead of GEMINI_API_KEY
# Then restart
docker compose -f docker/docker-compose.yaml up -d
```

## Configuration Comparison

| Feature | OpenAI | Gemini |
|---------|--------|--------|
| **LLM Model** | gpt-4.1-nano-2025-04-14 | gemini-2.0-flash |
| **Embedding Model** | text-embedding-3-large | text-embedding-004 |
| **Embedding Dimension** | 3072 | 768 |
| **Context Window** | 1M tokens | 1M tokens |
| **JSON Mode** | ✓ Supported | ✓ Supported |
| **Rate Limits** | Higher (paid tier) | Lower (free tier) |
| **Cost** | Per token pricing | Free tier available |
| **Engine Timeout** | 30s (default) | 120s (optimized for complex SQL) |

## Key Differences in Behavior

1. **Gemini 2.0 Flash** is faster but may have slightly different response patterns than GPT-4
2. **768-dim embeddings** are smaller and faster but may have slightly different semantic capture than 3072-dim
3. **Rate limits** are more restrictive on free tier Gemini (avoid `-exp` models)

## Files in This Repository

- `GEMINI_SETUP.md` - This file
- `SETUP_GITHUB.md` - Instructions for GitHub repository setup
- `docker/config.gemini.yaml` - Gemini configuration
- `docker/config.yaml.openai.backup` - OpenAI configuration backup
- `docker/.env` - Environment variables (gitignored - contains API keys)
- `docker/config.yaml` - Active config (gitignored - copy from either gemini or openai backup)

## Maintaining This Setup

### Pulling Updates from Upstream WrenAI

```bash
# Fetch latest from original repository
git fetch upstream

# Merge into your main branch
git checkout main
git merge upstream/main

# Resolve any conflicts with your config files
# Push to your repository
git push origin main
```

### Keeping Track of Your Changes

```bash
# Work on the gemini-integration branch
git checkout gemini-integration

# Make changes to config files
# Stage and commit
git add docker/config.gemini.yaml
git commit -m "Update: description of changes"

# Push to your repository
git push origin gemini-integration
```

## Support and Resources

- **WrenAI Documentation:** https://docs.getwren.ai
- **Google AI Studio:** https://aistudio.google.com
- **Gemini API Documentation:** https://ai.google.dev/gemini-api/docs
- **Langfuse Documentation:** https://langfuse.com/docs
- **Original WrenAI Repository:** https://github.com/Canner/WrenAI

## License

This is a fork/customization of WrenAI. Please refer to the original WrenAI repository for license information.
