---
description: Fast deployment using Docker layer caching for small code changes
---

# Quick Deploy (with Docker cache)

Fast deployment using Docker layer caching. Use this for small code changes when dependencies haven't changed.

**WARNING**: If you changed mix.exs, mix.lock, package.json, or package-lock.json, use `/deploy` instead.

## Usage

```
/quick-deploy
```

## Instructions

1. **Commit any changes**
   ```bash
   git add -A && git commit -m "Quick update" && git push origin main
   ```

2. **Build with cache**
   ```bash
   ./deploy/build.sh
   ```

3. **Deploy**
   ```bash
   ./deploy/deploy.sh
   ```

4. **Report status** with production URL

## When to Use

✅ Use `/quick-deploy` when:
- Only source code changed (no dependency updates)
- Small bug fixes or copy changes
- CSS/styling tweaks
- Configuration changes

❌ Use `/deploy` instead when:
- Dependencies changed (package.json, mix.exs, requirements.txt)
- Docker build context changed
- Infrastructure changes needed
- First deployment to a new environment

## Customization

Copy this skill to your project and update:

1. **Build command** - Replace `./deploy/build.sh` with your build script
2. **Deploy command** - Replace `./deploy/deploy.sh` with your deploy script
3. **Dependency files** - Update the WARNING list for your stack:
   - Node.js: `package.json`, `package-lock.json`
   - Elixir: `mix.exs`, `mix.lock`
   - Python: `requirements.txt`, `Pipfile.lock`
   - Go: `go.mod`, `go.sum`
