---
description: Deploy application to production server (template)
---

# Deploy Application

Deploy the application to production. **Customize this template for your project.**

## Usage

```
/deploy
```

## Template Instructions

Copy this skill to your project's `.claude/commands/deploy.md` and customize:

1. Update server details (hostname, IP, user)
2. Update build commands for your stack
3. Update service name and logs path
4. Add project-specific checks

## Generic Workflow

When you run `/deploy`, Claude will:

### Step 1: Pre-flight Checks

```bash
# Check for uncommitted changes
git status

# Ensure on main/production branch
git branch --show-current
```

If there are uncommitted changes, commit them with a descriptive message.

### Step 2: Push to Remote

```bash
git push origin main
```

### Step 3: Build (Customize per stack)

**Node.js:**
```bash
npm run build
```

**Docker:**
```bash
docker build -t myapp:latest .
```

**Elixir/Phoenix:**
```bash
MIX_ENV=prod mix release
```

### Step 4: Deploy

**Option A: Docker + SSH**
```bash
docker save myapp:latest | ssh deploy@server 'docker load && docker-compose up -d'
```

**Option B: rsync + systemd**
```bash
rsync -avz --delete ./build/ deploy@server:/var/www/myapp/
ssh deploy@server 'sudo systemctl restart myapp'
```

**Option C: Git pull on server**
```bash
ssh deploy@server 'cd /var/www/myapp && git pull && npm install && npm run build && pm2 restart all'
```

### Step 5: Verify

```bash
# Check service status
ssh deploy@server 'sudo systemctl status myapp'

# Check logs for errors
ssh deploy@server 'sudo journalctl -u myapp -n 50'

# Health check
curl -s https://myapp.com/health | jq
```

## Customization Points

Edit these sections for your project:

```yaml
# Server details
server_host: myapp.com
server_ip: 123.456.789.0
server_user: deploy
app_path: /var/www/myapp

# Service
service_name: myapp
logs_command: sudo journalctl -u myapp -f

# Build
build_command: npm run build
test_command: npm test

# Deploy method: docker | rsync | git-pull
deploy_method: rsync
```

## Example: Node.js App

```markdown
# Deploy MyApp

Deploy to production at myapp.com.

## Steps

1. **Check for uncommitted changes**
   \`\`\`bash
   git status
   \`\`\`

2. **Run tests**
   \`\`\`bash
   npm test
   \`\`\`

3. **Build**
   \`\`\`bash
   npm run build
   \`\`\`

4. **Deploy**
   \`\`\`bash
   rsync -avz --delete ./dist/ deploy@myapp.com:/var/www/myapp/
   ssh deploy@myapp.com 'sudo systemctl restart myapp'
   \`\`\`

5. **Verify**
   Report deployment status and provide URL: https://myapp.com
```

## Quick Reference

| Stack | Build | Deploy |
|-------|-------|--------|
| Node.js | `npm run build` | rsync + pm2/systemd |
| Docker | `docker build` | docker-compose |
| Elixir | `mix release` | rsync + systemd |
| Static | `npm run build` | rsync to nginx/S3 |

## Safety Checks

Always include:
- ✅ Test suite passes
- ✅ No uncommitted changes
- ✅ On correct branch (main/production)
- ✅ Version bumped if needed
- ✅ Environment variables set

## Rollback

Add a rollback command to your custom skill:

```bash
# Keep previous release
ssh deploy@server 'cp -r /var/www/myapp /var/www/myapp.backup'

# Rollback if needed
ssh deploy@server 'rm -rf /var/www/myapp && mv /var/www/myapp.backup /var/www/myapp && sudo systemctl restart myapp'
```
