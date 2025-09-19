# 🔍 Understanding `shopify app dev` vs Manual Development

This guide explains exactly what `shopify app dev` does behind the scenes and how to replicate each step manually without using tunnels.

---

## 🔄 What `shopify app dev` Does (Step-by-Step)

### **Phase 1: Authentication & Linking**
1. **Checks authentication**: Ensures you're logged in (`shopify login`)
2. **Reads project config**: Loads `shopify.app.toml` and `shopify.web.toml`
3. **Verifies app link**: Uses the `client_id` from `shopify.app.toml` to identify your Partner app
4. **Fetches credentials**: Downloads API key/secret from Partner Dashboard using your auth session

### **Phase 2: Environment Setup**
1. **Generates tunnel URL**: Creates Cloudflare tunnel (e.g., `https://abc123.trycloudflare.com`)
2. **Updates Partner Dashboard**: Temporarily sets App URL and Redirect URLs to tunnel
3. **Injects environment variables**:
   ```bash
   SHOPIFY_API_KEY=your_api_key                    # From Partner Dashboard
   SHOPIFY_API_SECRET=your_api_secret              # From Partner Dashboard
   SCOPES=write_products                           # From shopify.app.toml
   SHOPIFY_APP_URL=https://abc123.trycloudflare.com # Generated tunnel
   # Plus many others...
   ```

### **Phase 3: Pre-Development**
1. **Runs predev command**: Executes `predev = "npx prisma generate"` from `shopify.web.toml`
2. **Sets up database**: Ensures Prisma client is ready

### **Phase 4: Development Server**
1. **Starts web server**: Runs `dev = "npx prisma migrate deploy && npm exec remix vite:dev"`
2. **Starts extension servers**: Builds/serves any extensions in parallel
3. **Sets up webhook proxy**: Routes Shopify webhooks through tunnel to your local server

### **Phase 5: Runtime Management**
1. **Monitors file changes**: Restarts servers when config files change
2. **Maintains tunnel**: Keeps Cloudflare tunnel alive
3. **Provides CLI commands**: Interactive commands like opening browser, triggering webhooks

---

## 🚫 Why We Can't Use `shopify app dev`

### **Security Issues**
- **Creates public tunnel**: Exposes your local environment to the internet
- **Automatic URL updates**: Changes Partner Dashboard settings without explicit consent
- **Third-party dependency**: Uses Cloudflare infrastructure outside organizational control

### **Compliance Violations**
- Violates "no tunneling" policies
- Potential data exposure through external services
- Uncontrolled external ingress points

---

## ✅ Manual Alternative: Tunnel-Free Development

### **Step 1: One-Time Setup**

#### **1.1 Link to Partner App**
```bash
# Log in to Shopify Partners
shopify login

# Link this project to your Partner app
npm run config:link
# (Select your organization and app)
```

#### **1.2 Get Your Credentials**
```bash
# See what shopify app dev would inject
npm run env

# Or get from Partner Dashboard:
# Partners → Apps → Your App → App setup → Client credentials
```

#### **1.3 Create Environment File**

**⚠️ CRITICAL SECURITY WARNING:**
If you see a tunnel URL in your `.env` file (like `https://*.trycloudflare.com`), someone has been running `shopify app dev` which creates security violations. Replace it immediately!

```bash
# Create .env in your project root
cat > .env << EOF
SHOPIFY_API_KEY=e3b6594ad7e90f604458beac7439253a
SHOPIFY_API_SECRET=your_secret_from_dashboard
SCOPES=write_products
SHOPIFY_APP_URL=http://localhost:3000  # For local dev (limited OAuth)
# OR for full functionality:
# SHOPIFY_APP_URL=https://your-staging-domain.com
EOF
```

**URL Options:**
- `http://localhost:3000` - Local development (API calls work, OAuth limited)
- `https://your-staging-domain.com` - Full functionality with staging deployment

#### **1.4 Set Up Production/Staging URLs**
**In Partner Dashboard:**
- **App URL**: `https://your-staging-domain.com/`
- **Allowed redirect URLs**: `https://your-staging-domain.com/api/auth`

### **Step 2: Development Workflow**

#### **2.1 Manual Pre-Development** (replaces CLI's predev)
```bash
# Generate Prisma client (same as shopify.web.toml predev)
npx prisma generate
```

#### **2.2 Start Development Server** (replaces CLI's dev command)
```bash
# Run the same command as shopify.web.toml dev
npx prisma migrate deploy && npm exec remix vite:dev

# Or use package.json script
npm run dev
```

### **Step 3: Testing & Webhooks**

#### **3.1 Local Testing**
```bash
# Your app runs on localhost:3000
# Test API endpoints, database operations, etc.
```

#### **3.2 Production Testing**
```bash
# Deploy to staging for OAuth/webhook testing
# Update SHOPIFY_APP_URL to staging domain
```

#### **3.3 Webhook Testing**
```bash
# Use Shopify CLI without starting dev server
shopify app generate webhook
shopify webhook trigger --topic=orders/create
```

---

## 📋 Environment Variables Comparison

### **What `shopify app dev` Injects Automatically**
```bash
SHOPIFY_API_KEY=e3b6594ad7e90f604458beac7439253a
SHOPIFY_API_SECRET=shhh_secret_from_dashboard
SHOPIFY_APP_URL=https://random123.trycloudflare.com
SCOPES=write_products
HOST=random123.trycloudflare.com
# Plus many others for extensions, dev tools, etc.
```

### **What You Need to Set Manually**
```bash
# Required for app functionality
SHOPIFY_API_KEY=e3b6594ad7e90f604458beac7439253a
SHOPIFY_API_SECRET=your_secret_from_dashboard
SCOPES=write_products
SHOPIFY_APP_URL=https://your-staging-domain.com

# Optional for development
SHOP_CUSTOM_DOMAIN=your-dev-store.myshopify.com
```

---

## 🔧 Automation Scripts

### **Create `scripts/setup-env.sh`**
```bash
#!/bin/bash
echo "🔧 Setting up tunnel-free development environment"

# Check if linked to Partner app
if ! grep -q "client_id" shopify.app.toml; then
  echo "❌ No app linked. Run: npm run config:link"
  exit 1
fi

# Get API key from config
API_KEY=$(grep "client_id" shopify.app.toml | cut -d'"' -f2)

# Create .env if it doesn't exist
if [ ! -f .env ]; then
  echo "📝 Creating .env file..."
  cat > .env << EOF
SHOPIFY_API_KEY=$API_KEY
SHOPIFY_API_SECRET=your_secret_here
SCOPES=write_products
SHOPIFY_APP_URL=https://your-staging-domain.com
EOF
  echo "✅ .env created. Update SHOPIFY_API_SECRET and SHOPIFY_APP_URL"
else
  echo "✅ .env already exists"
fi
```

### **Create `scripts/dev-secure.sh`**
```bash
#!/bin/bash
echo "🔒 Starting secure development (NO TUNNELS)"

# Check environment
if [ ! -f .env ]; then
  echo "❌ .env file not found. Run ./scripts/setup-env.sh first"
  exit 1
fi

# Source environment
set -a
source .env
set +a

# Validate required vars
if [[ -z "$SHOPIFY_API_KEY" || -z "$SHOPIFY_API_SECRET" || "$SHOPIFY_API_SECRET" == "your_secret_here" ]]; then
  echo "❌ Update SHOPIFY_API_SECRET in .env file"
  exit 1
fi

echo "✅ Environment loaded"
echo "🔑 API Key: $SHOPIFY_API_KEY"
echo "🌐 App URL: $SHOPIFY_APP_URL"
echo ""

# Run pre-development setup
echo "📦 Generating Prisma client..."
npx prisma generate

# Start development server
echo "🚀 Starting development server..."
npx prisma migrate deploy && npm exec remix vite:dev
```

Make scripts executable:
```bash
chmod +x scripts/setup-env.sh scripts/dev-secure.sh
```

---

## 🚀 Quick Start Commands

### **Initial Setup**
```bash
# 1. Link to Partner app
npm run config:link

# 2. Set up environment
./scripts/setup-env.sh

# 3. Edit .env with your real credentials
nano .env
```

### **Daily Development**
```bash
# Start secure development
./scripts/dev-secure.sh

# Or manually
npm run dev
```

### **Testing & Deployment**
```bash
# Build for production
npm run build

# Deploy to staging
npm run deploy  # or your deployment command
```

---

## 🔍 Verification Checklist

### **Security Compliance**
- [ ] No `shopify app dev` in any scripts
- [ ] No tunnel-related dependencies
- [ ] Environment variables set manually
- [ ] Production URLs configured in Partner Dashboard

### **Functionality**
- [ ] App loads locally on localhost:3000
- [ ] Database operations work
- [ ] API calls succeed with proper authentication
- [ ] Staging environment accessible via HTTPS

### **Team Workflow**
- [ ] Documentation updated for team
- [ ] Scripts tested and working
- [ ] Environment setup documented
- [ ] Deployment process defined

---

## 🆘 Troubleshooting

### **"Missing environment variables"**
```bash
# Check .env file exists and has values
cat .env

# Source environment manually
set -a && source .env && set +a
npm run dev
```

### **"Authentication failed"**
```bash
# Verify API credentials
npm run env  # Shows what CLI would inject

# Check Partner Dashboard credentials match .env
```

### **"App not loading in Shopify Admin"**
- Verify SHOPIFY_APP_URL matches Partner Dashboard App URL
- Ensure staging domain is accessible via HTTPS
- Check redirect URLs are correctly configured

---

## 📚 Related Files

- [`REMOVE-TUNNELS-GUIDE.md`](./REMOVE-TUNNELS-GUIDE.md) - Security compliance guide
- [`README.md`](./README.md) - Original development instructions (with tunnels)
- [`shopify.app.toml`](./shopify.app.toml) - App configuration
- [`shopify.web.toml`](./shopify.web.toml) - Development commands

---

**Remember**: This approach trades convenience for security compliance. The extra setup ensures your development environment meets organizational security requirements while maintaining full Shopify app functionality.
