# 🔒 Remove Cloudflare Tunnel & Ngrok from Shopify Remix App

**⚠️ CRITICAL: For Security Compliance - Complete Tunnel Removal Guide**

This guide removes ALL tunnel dependencies from your Shopify Remix app to prevent security violations that could result in laptop lockdown/data loss.

## 🚨 Why This Matters

- **Security Risk**: Cloudflare tunnels and ngrok expose your local environment to the internet
- **Compliance Issues**: Many organizations ban third-party tunneling services
- **Data Protection**: Prevents accidental exposure of sensitive development data

---

## 📋 Step-by-Step Removal Process

### Step 1: Modify package.json

**REMOVE** the Cloudflare plugin from `trustedDependencies`:

```json
{
  "scripts": {
    "build": "remix vite:build",
    "dev": "npm exec remix vite:dev",
    "dev:secure": "npm exec remix vite:dev --host 0.0.0.0 --port 3000",
    "dev:shopify": "shopify app dev",
    "config:link": "shopify app config link",
    "generate": "shopify app generate",
    "deploy": "shopify app deploy",
    "config:use": "shopify app config use",
    "env": "shopify app env",
    "start": "remix-serve ./build/server/index.js",
    "docker-start": "npm run setup && npm run start",
    "setup": "prisma generate && prisma migrate deploy",
    "lint": "eslint --cache --cache-location ./node_modules/.cache/eslint .",
    "shopify": "shopify",
    "prisma": "prisma",
    "graphql-codegen": "graphql-codegen",
    "vite": "vite"
  },
  // ... other sections remain the same ...
  "trustedDependencies": [
    // ❌ REMOVE THIS LINE: "@shopify/plugin-cloudflare"
  ],
  // ... rest remains the same ...
}
```

### Step 2: Update shopify.web.toml

Modify the dev command to use only Remix:

```toml
name = "remix"
roles = ["frontend", "backend"]
webhooks_path = "/webhooks/app/uninstalled"

[commands]
predev = "npx prisma generate"
dev = "npx prisma migrate deploy && npm exec remix vite:dev --host 0.0.0.0 --port 3000"
```

### Step 3: Configure shopify.app.toml for Production URLs

**NEVER use localhost or tunnel URLs in production config:**

```toml
client_id = "your_actual_client_id"
name = "your-app-name"
application_url = "https://your-production-domain.com/"
embedded = true

[auth]
redirect_urls = [ "https://your-production-domain.com/api/auth" ]

# ... rest of your config ...
```

### Step 4: Remove Tunnel Dependencies

**Uninstall any tunnel-related packages:**

```bash
# Remove cloudflare plugin if installed
npm uninstall @shopify/plugin-cloudflare

# Remove ngrok if installed
npm uninstall ngrok

# Clean install to remove any cached tunnel dependencies
rm -rf node_modules package-lock.json
npm install
```

### Step 5: Create Secure Development Script

Create `scripts/dev-secure.sh`:

```bash
#!/bin/bash
echo "🔒 Starting SECURE development mode (NO TUNNELS)"
echo "⚠️  Ensure shopify.app.toml points to production/staging URLs"
echo "🌐 Local server will run on: http://localhost:3000"
echo ""

# Ensure no tunnel processes are running
pkill -f "cloudflare" 2>/dev/null || true
pkill -f "ngrok" 2>/dev/null || true

# Start secure development
npm exec remix vite:dev --host 0.0.0.0 --port 3000
```

Make it executable:
```bash
chmod +x scripts/dev-secure.sh
```

---

## 🛡️ Development Workflow (Tunnel-Free)

### For Local Development:
```bash
# ✅ SAFE: Use this for local development
npm run dev:secure
# OR
./scripts/dev-secure.sh
```

### ❌ NEVER RUN THESE COMMANDS:
```bash
# ❌ DANGEROUS: These create tunnels
npm run dev                    # This runs 'shopify app dev'
shopify app dev               # Creates Cloudflare tunnel
npx shopify app dev           # Creates Cloudflare tunnel
```

---

## 🌐 Production Deployment Options

Since you cannot use tunnels, choose one of these approaches:

### Option A: Staging Environment (Recommended)
1. **Deploy to staging server** (Render, Fly.io, Railway, Vercel)
2. **Use subdomain**: `https://staging-yourapp.yourcompany.com`
3. **Update Shopify Partner Dashboard** with staging URLs
4. **Develop locally, deploy to staging for testing**

### Option B: Corporate Reverse Proxy
1. **Use company's reverse proxy** with TLS certificates
2. **Configure internal domain**: `https://dev-yourapp.internal.company.com`
3. **Ensure proxy allows Shopify webhook traffic**

### Option C: Production-Ready Local Setup
1. **Install mkcert** for local HTTPS: `brew install mkcert`
2. **Create certificates**: `mkcert localhost 127.0.0.1`
3. **Configure Vite for HTTPS**:
   ```javascript
   // vite.config.ts
   export default {
     server: {
       https: {
         key: fs.readFileSync('localhost-key.pem'),
         cert: fs.readFileSync('localhost.pem'),
       },
       port: 3000,
       host: '0.0.0.0'
     }
   }
   ```

---

## 📝 Shopify Partner Dashboard Configuration

**Update these URLs in your Shopify Partner Dashboard:**

1. **App URL**: `https://your-production-domain.com/`
2. **Allowed redirect URLs**: `https://your-production-domain.com/api/auth`
3. **Webhook endpoints**: `https://your-production-domain.com/webhooks/*`

**⚠️ Important**: Never use `localhost`, `127.0.0.1`, or tunnel URLs in production configuration.

---

## 🔍 Verification Checklist

Before you start development, verify tunnel removal:

- [ ] `package.json` has no `@shopify/plugin-cloudflare` in `trustedDependencies`
- [ ] `npm run dev` points to secure command (not `shopify app dev`)
- [ ] `shopify.app.toml` uses production URLs (not example.com or localhost)
- [ ] No tunnel-related packages in `node_modules`
- [ ] Secure development script works without tunnel creation

**Test with:**
```bash
# Should show your secure dev command, not shopify app dev
npm run dev

# Should not find any tunnel dependencies
grep -r "cloudflare\|ngrok\|tunnel" package.json node_modules/ 2>/dev/null || echo "✅ Clean!"
```

---

## 🚨 Emergency Procedures

**If you accidentally run a tunnel command:**

1. **Immediately stop the process**: `Ctrl+C`
2. **Kill any tunnel processes**:
   ```bash
   pkill -f "cloudflare"
   pkill -f "ngrok"
   pkill -f "tunnel"
   ```
3. **Check for active tunnels**: `ps aux | grep -E "(cloudflare|ngrok|tunnel)"`
4. **Report to security team if required by your organization**

---

## ✅ Final Security Notes

- **Never commit tunnel URLs** to version control
- **Always use HTTPS** for production URLs
- **Keep staging environment URLs** in a secure location
- **Regularly audit dependencies** for tunnel packages
- **Train team members** on this secure workflow

**Remember**: One tunnel violation can result in laptop lockdown and data loss. This guide ensures complete compliance with security policies.
