# Setting Up Your GitHub Repository for WrenAI

You cloned the original WrenAI repository directly. Here's how to track your changes on GitHub:

## Step 1: Create a New Repository on GitHub

1. Go to https://github.com/new
2. Create a new repository (e.g., `WrenAI-custom` or `WrenAI-gemini`)
3. **DO NOT** initialize with README, .gitignore, or license
4. Keep it public or private as you prefer
5. Copy the repository URL (will be something like `https://github.com/YOUR_USERNAME/WrenAI-custom.git`)

## Step 2: Update Your Local Repository

Run these commands in your WrenAI directory:

```bash
# Add your new repository as a remote (keeping original as upstream)
git remote rename origin upstream
git remote add origin https://github.com/YOUR_USERNAME/WrenAI-custom.git

# Verify remotes are set correctly
git remote -v
# Should show:
# origin    https://github.com/YOUR_USERNAME/WrenAI-custom.git (fetch)
# origin    https://github.com/YOUR_USERNAME/WrenAI-custom.git (push)
# upstream  https://github.com/Canner/WrenAI.git (fetch)
# upstream  https://github.com/Canner/WrenAI.git (push)
```

## Step 3: Create a Branch for Your Changes

```bash
# Create a new branch for your Gemini integration
git checkout -b gemini-integration

# Stage your configuration changes
git add docker/config.yaml docker/config.gemini.yaml docker/config.yaml.openai.backup

# Commit your changes
git commit -m "Configure WrenAI to use Google Gemini 2.0 Flash

- Switch from gemini-2.0-flash-exp to stable gemini-2.0-flash
- Update embedding model to gemini/text-embedding-004 (768-dim)
- Set recreate_index: true for Qdrant re-indexing
- Add backup of original OpenAI config
- Configure Langfuse integration

Fixes rate limit issues with experimental Gemini model."
```

## Step 4: Push to Your GitHub Repository

```bash
# Push your branch to your repository
git push -u origin gemini-integration

# Or if you want to push to main branch:
git checkout main
git merge gemini-integration
git push -u origin main
```

## Step 5: Keep Synced with Upstream (Optional)

To pull updates from the original WrenAI repository:

```bash
# Fetch latest changes from upstream
git fetch upstream

# Merge upstream changes into your main branch
git checkout main
git merge upstream/main

# Push updates to your repository
git push origin main
```

## Alternative: Use GitHub's Import Feature

If you prefer, you can also:

1. Go to https://github.com/new/import
2. Enter the original URL: `https://github.com/Canner/WrenAI.git`
3. Name your repository
4. Let GitHub import it
5. Then update your local remote:

```bash
git remote set-url origin https://github.com/YOUR_USERNAME/WrenAI-custom.git
git push origin main
```

## What's Changed in Your Setup

Your configuration changes:
- `docker/config.yaml` - Gemini 2.0 Flash configuration
- `docker/config.gemini.yaml` - Backup Gemini config
- `docker/config.yaml.openai.backup` - Original OpenAI config

These are currently untracked. Once you push to your GitHub repo, you can track all future changes.

## Benefits of This Setup

- **origin**: Your repository (where you push changes)
- **upstream**: Original WrenAI repository (to pull updates)
- You can maintain your custom Gemini configuration
- You can still get updates from the original project
- You have a backup of your work on GitHub
