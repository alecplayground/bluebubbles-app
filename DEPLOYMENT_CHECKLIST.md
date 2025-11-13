# POA Chat Aggregator - Complete Deployment Checklist

**Project:** BlueBubbles Multi-Server Chat Aggregator
**Target:** Production deployment with 8 BlueBubbles servers
**Last Updated:** November 12, 2025

---

## Table of Contents

1. [Pre-Deployment Setup](#pre-deployment-setup)
2. [Local Development Setup](#local-development-setup)
3. [Application Configuration](#application-configuration)
4. [Building for Production](#building-for-production)
5. [Deployment Options](#deployment-options)
6. [Connecting Your 8 BlueBubbles Servers](#connecting-your-8-bluebubbles-servers)
7. [Post-Deployment Testing](#post-deployment-testing)
8. [Monitoring & Maintenance](#monitoring--maintenance)
9. [Troubleshooting](#troubleshooting)

---

## Pre-Deployment Setup

### Prerequisites Check

- [ ] **Node.js installed** - Version 18.x or higher
  ```bash
  node --version  # Should show v18.x or higher
  ```

- [ ] **npm installed** - Version 9.x or higher
  ```bash
  npm --version  # Should show 9.x or higher
  ```

- [ ] **Git installed** - For version control
  ```bash
  git --version
  ```

- [ ] **Code editor ready** - VS Code, Cursor, or your preferred editor

- [ ] **8 BlueBubbles servers running** - Verify each server is accessible
  - Server 1 URL and password ready
  - Server 2 URL and password ready
  - Server 3 URL and password ready
  - Server 4 URL and password ready
  - Server 5 URL and password ready
  - Server 6 URL and password ready
  - Server 7 URL and password ready
  - Server 8 URL and password ready

- [ ] **Network access verified** - Can reach each BlueBubbles server from your deployment machine
  ```bash
  # Test connectivity to each server
  curl -I https://your-first-server.com/api/v1/ping
  curl -I https://your-second-server.com/api/v1/ping
  # Repeat for all 8 servers
  ```

### Security Preparations

- [ ] **Document all 8 server credentials** - Store securely (use password manager)
  - Create a secure note with format:
    ```
    Server 1: https://server1.example.com | password123
    Server 2: https://server2.example.com | password456
    Server 3: https://server3.example.com | password789
    ... (all 8 servers)
    ```

- [ ] **Choose deployment platform** - Vercel (recommended), Netlify, or self-hosted

- [ ] **Prepare domain name** (optional but recommended)
  - Example: `chat-aggregator.yourdomain.com`
  - DNS records ready to point to deployment

---

## Local Development Setup

### 1. Clone/Copy Application Code

- [ ] **Navigate to application directory**
  ```bash
  cd /home/user/poa-chat-aggregator
  ```

- [ ] **Verify all files are present**
  ```bash
  ls -la
  # Should see: package.json, next.config.ts, tsconfig.json, etc.
  ```

- [ ] **Check critical directories exist**
  ```bash
  ls app/          # Next.js app directory
  ls components/   # React components
  ls lib/          # API client, database, websocket
  ls store/        # Zustand stores
  ls types/        # TypeScript types
  ```

### 2. Install Dependencies

- [ ] **Clean install all dependencies**
  ```bash
  npm clean-install
  ```

- [ ] **Verify installation succeeded**
  ```bash
  npm list --depth=0
  # Should show: next, react, zustand, dexie, socket.io-client, etc.
  ```

- [ ] **Check for security vulnerabilities**
  ```bash
  npm audit
  # Address any critical or high severity issues
  ```

### 3. Initial Build Test

- [ ] **Run development build**
  ```bash
  npm run dev
  ```

- [ ] **Verify app starts** - Should show:
  ```
  ✓ Ready in [X]s
  ○ Local: http://localhost:3000
  ```

- [ ] **Open browser and test** - Navigate to `http://localhost:3000`
  - [ ] Page loads without errors
  - [ ] No console errors in browser DevTools
  - [ ] UI shows "No chats yet - Add a server to get started"

- [ ] **Stop development server** - Press `Ctrl+C`

### 4. Production Build Test

- [ ] **Run production build**
  ```bash
  npm run build
  ```

- [ ] **Verify build succeeds** - Should show:
  ```
  ✓ Compiled successfully
  Route (app)                         Size  First Load JS
  ┌ ○ /                            77.8 kB         191 kB
  ```

- [ ] **Test production build locally**
  ```bash
  npm start
  ```

- [ ] **Verify production app works** - Navigate to `http://localhost:3000`
  - [ ] App loads correctly
  - [ ] No errors in console
  - [ ] UI is responsive

- [ ] **Stop production server** - Press `Ctrl+C`

---

## Application Configuration

### Environment Setup

- [ ] **Create environment file** (if needed for custom config)
  ```bash
  touch .env.local
  ```

- [ ] **Add environment variables** (optional - for advanced config)
  ```bash
  # .env.local
  NEXT_PUBLIC_APP_NAME="POA Chat Aggregator"
  NEXT_PUBLIC_MAX_SERVERS=8
  NEXT_PUBLIC_DEBUG_MODE=false
  ```

### Review Configuration Files

- [ ] **Check `next.config.ts`** - Verify settings
  ```bash
  cat next.config.ts
  ```
  - [ ] React Compiler enabled: `reactCompiler: true`
  - [ ] ESLint ignored during builds: `ignoreDuringBuilds: true`
  - [ ] TypeScript errors NOT ignored: `ignoreBuildErrors: false`

- [ ] **Check `package.json`** - Verify scripts
  ```bash
  cat package.json
  ```
  - [ ] `dev` script uses Turbopack
  - [ ] `build` script uses Turbopack
  - [ ] `start` script available

### Prepare Server Names

- [ ] **Decide on friendly names for your 8 servers**
  - Example naming scheme:
    ```
    Server 1: "Mac Studio"
    Server 2: "MacBook Pro"
    Server 3: "Mac Mini Office"
    Server 4: "iMac Home"
    Server 5: "Mac Pro Lab"
    Server 6: "MacBook Air Travel"
    Server 7: "Mac Mini Backup"
    Server 8: "Mac Studio Secondary"
    ```

- [ ] **Document server names with URLs**
  ```
  Mac Studio: https://studio.example.com
  MacBook Pro: https://macbook.example.com
  Mac Mini Office: https://office-mini.example.com
  iMac Home: https://imac-home.example.com
  Mac Pro Lab: https://lab-pro.example.com
  MacBook Air Travel: https://air-travel.example.com
  Mac Mini Backup: https://backup-mini.example.com
  Mac Studio Secondary: https://studio2.example.com
  ```

---

## Building for Production

### Final Production Build

- [ ] **Clean previous builds**
  ```bash
  rm -rf .next
  rm -rf out
  rm -rf node_modules/.cache
  ```

- [ ] **Run fresh production build**
  ```bash
  npm run build
  ```

- [ ] **Verify build output** - Check bundle sizes
  - [ ] Main route under 200 kB (currently ~191 kB)
  - [ ] No build warnings about bundle size
  - [ ] No TypeScript errors

- [ ] **Test production build one more time**
  ```bash
  npm start
  ```
  - [ ] Navigate to `http://localhost:3000`
  - [ ] Verify everything works
  - [ ] Stop server with `Ctrl+C`

### Optimize for Production (Optional)

- [ ] **Enable source maps** (for debugging) - Add to `next.config.ts`:
  ```typescript
  productionBrowserSourceMaps: true,
  ```

- [ ] **Configure compression** - Already enabled by default in Next.js

- [ ] **Review bundle analyzer** (optional)
  ```bash
  npm install --save-dev @next/bundle-analyzer
  ```

---

## Deployment Options

Choose ONE deployment method below:

### Option A: Vercel (Recommended - Easiest)

#### A1. Vercel Account Setup

- [ ] **Create Vercel account** - Go to https://vercel.com/signup
  - [ ] Sign up with GitHub/GitLab/Email
  - [ ] Verify email address
  - [ ] Complete account setup

#### A2. Install Vercel CLI

- [ ] **Install Vercel CLI globally**
  ```bash
  npm install -g vercel
  ```

- [ ] **Verify installation**
  ```bash
  vercel --version
  ```

- [ ] **Login to Vercel**
  ```bash
  vercel login
  # Follow prompts to authenticate
  ```

#### A3. Deploy to Vercel

- [ ] **Navigate to project directory**
  ```bash
  cd /home/user/poa-chat-aggregator
  ```

- [ ] **Initialize Vercel project**
  ```bash
  vercel
  ```
  - [ ] Confirm project settings
  - [ ] Choose project name (e.g., "poa-chat-aggregator")
  - [ ] Select "No" for overriding build settings
  - [ ] Wait for deployment to complete

- [ ] **Note deployment URL** - Vercel will provide a URL like:
  ```
  https://poa-chat-aggregator-abc123.vercel.app
  ```

- [ ] **Test deployment** - Open the URL in browser
  - [ ] App loads correctly
  - [ ] No console errors
  - [ ] Shows "Add a server to get started"

#### A4. Configure Production Domain (Optional)

- [ ] **Add custom domain in Vercel dashboard**
  - [ ] Go to project settings → Domains
  - [ ] Add your domain (e.g., `chat.yourdomain.com`)
  - [ ] Follow DNS configuration instructions

- [ ] **Verify domain works**
  - [ ] Navigate to custom domain
  - [ ] SSL certificate provisioned automatically
  - [ ] App loads correctly

#### A5. Set Environment Variables (if using)

- [ ] **Add environment variables in Vercel**
  - [ ] Go to project settings → Environment Variables
  - [ ] Add each variable from `.env.local`
  - [ ] Apply to Production, Preview, and Development

- [ ] **Redeploy after adding variables**
  ```bash
  vercel --prod
  ```

---

### Option B: Netlify

#### B1. Netlify Account Setup

- [ ] **Create Netlify account** - Go to https://app.netlify.com/signup
  - [ ] Sign up with GitHub/GitLab/Email
  - [ ] Verify email
  - [ ] Complete setup

#### B2. Install Netlify CLI

- [ ] **Install Netlify CLI**
  ```bash
  npm install -g netlify-cli
  ```

- [ ] **Login to Netlify**
  ```bash
  netlify login
  ```

#### B3. Configure for Netlify

- [ ] **Create `netlify.toml` in project root**
  ```bash
  cd /home/user/poa-chat-aggregator
  touch netlify.toml
  ```

- [ ] **Add Netlify configuration**
  ```toml
  [build]
    command = "npm run build"
    publish = ".next"

  [[plugins]]
    package = "@netlify/plugin-nextjs"
  ```

- [ ] **Install Netlify Next.js plugin**
  ```bash
  npm install --save-dev @netlify/plugin-nextjs
  ```

#### B4. Deploy to Netlify

- [ ] **Initialize Netlify site**
  ```bash
  netlify init
  ```
  - [ ] Create & configure new site
  - [ ] Choose team
  - [ ] Enter site name

- [ ] **Deploy to production**
  ```bash
  netlify deploy --prod
  ```

- [ ] **Note deployment URL**
  ```
  https://your-site-name.netlify.app
  ```

- [ ] **Test deployment** - Open URL in browser

---

### Option C: Self-Hosted (VPS/Cloud)

#### C1. Prepare Server

- [ ] **Provision server** - Ubuntu 22.04 LTS recommended
  - Minimum specs: 2 vCPU, 2 GB RAM, 20 GB storage
  - [ ] Server IP address noted
  - [ ] SSH access configured
  - [ ] Root or sudo access available

- [ ] **SSH into server**
  ```bash
  ssh user@your-server-ip
  ```

#### C2. Install Dependencies on Server

- [ ] **Update system packages**
  ```bash
  sudo apt update
  sudo apt upgrade -y
  ```

- [ ] **Install Node.js 18.x**
  ```bash
  curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
  sudo apt install -y nodejs
  node --version  # Verify 18.x
  ```

- [ ] **Install PM2 (process manager)**
  ```bash
  sudo npm install -g pm2
  pm2 --version
  ```

- [ ] **Install Nginx (reverse proxy)**
  ```bash
  sudo apt install -y nginx
  sudo systemctl enable nginx
  sudo systemctl start nginx
  ```

#### C3. Deploy Application to Server

- [ ] **Create application directory**
  ```bash
  sudo mkdir -p /var/www/poa-chat-aggregator
  sudo chown -R $USER:$USER /var/www/poa-chat-aggregator
  ```

- [ ] **Copy application files to server**

  From your local machine:
  ```bash
  cd /home/user/poa-chat-aggregator

  # Create a tarball
  tar -czf poa-chat-aggregator.tar.gz \
    --exclude=node_modules \
    --exclude=.next \
    --exclude=.git \
    .

  # Copy to server
  scp poa-chat-aggregator.tar.gz user@your-server-ip:/var/www/poa-chat-aggregator/
  ```

- [ ] **Extract files on server**
  ```bash
  ssh user@your-server-ip
  cd /var/www/poa-chat-aggregator
  tar -xzf poa-chat-aggregator.tar.gz
  rm poa-chat-aggregator.tar.gz
  ```

- [ ] **Install dependencies on server**
  ```bash
  cd /var/www/poa-chat-aggregator
  npm clean-install --production
  ```

- [ ] **Build application on server**
  ```bash
  npm run build
  ```

#### C4. Configure PM2

- [ ] **Create PM2 ecosystem file**
  ```bash
  cd /var/www/poa-chat-aggregator
  nano ecosystem.config.js
  ```

- [ ] **Add PM2 configuration**
  ```javascript
  module.exports = {
    apps: [{
      name: 'poa-chat-aggregator',
      script: 'node_modules/next/dist/bin/next',
      args: 'start',
      cwd: '/var/www/poa-chat-aggregator',
      instances: 1,
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
        PORT: 3000
      }
    }]
  };
  ```

- [ ] **Start application with PM2**
  ```bash
  pm2 start ecosystem.config.js
  ```

- [ ] **Save PM2 configuration**
  ```bash
  pm2 save
  ```

- [ ] **Enable PM2 startup on boot**
  ```bash
  pm2 startup
  # Follow the command it provides
  ```

- [ ] **Verify app is running**
  ```bash
  pm2 status
  pm2 logs poa-chat-aggregator --lines 50
  ```

#### C5. Configure Nginx Reverse Proxy

- [ ] **Create Nginx configuration**
  ```bash
  sudo nano /etc/nginx/sites-available/poa-chat-aggregator
  ```

- [ ] **Add Nginx config**
  ```nginx
  server {
      listen 80;
      server_name your-domain.com;  # Replace with your domain or IP

      location / {
          proxy_pass http://localhost:3000;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection 'upgrade';
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
          proxy_cache_bypass $http_upgrade;
      }
  }
  ```

- [ ] **Enable site**
  ```bash
  sudo ln -s /etc/nginx/sites-available/poa-chat-aggregator /etc/nginx/sites-enabled/
  ```

- [ ] **Test Nginx configuration**
  ```bash
  sudo nginx -t
  ```

- [ ] **Reload Nginx**
  ```bash
  sudo systemctl reload nginx
  ```

#### C6. Setup SSL with Let's Encrypt (Recommended)

- [ ] **Install Certbot**
  ```bash
  sudo apt install -y certbot python3-certbot-nginx
  ```

- [ ] **Obtain SSL certificate**
  ```bash
  sudo certbot --nginx -d your-domain.com
  # Follow prompts, choose redirect option
  ```

- [ ] **Verify SSL renewal works**
  ```bash
  sudo certbot renew --dry-run
  ```

- [ ] **Test HTTPS access**
  - Navigate to `https://your-domain.com`
  - Verify SSL certificate is valid

---

## Connecting Your 8 BlueBubbles Servers

### Pre-Connection Checklist

- [ ] **Verify all 8 servers are accessible**
  ```bash
  # Test each server (replace with your actual URLs)
  curl https://server1.example.com/api/v1/ping?guid=YOUR_PASSWORD
  curl https://server2.example.com/api/v1/ping?guid=YOUR_PASSWORD
  curl https://server3.example.com/api/v1/ping?guid=YOUR_PASSWORD
  curl https://server4.example.com/api/v1/ping?guid=YOUR_PASSWORD
  curl https://server5.example.com/api/v1/ping?guid=YOUR_PASSWORD
  curl https://server6.example.com/api/v1/ping?guid=YOUR_PASSWORD
  curl https://server7.example.com/api/v1/ping?guid=YOUR_PASSWORD
  curl https://server8.example.com/api/v1/ping?guid=YOUR_PASSWORD
  ```

- [ ] **Expected response from each server**:
  ```json
  {"status":200,"message":"pong!","data":{"message":"pong!"}}
  ```

- [ ] **If any server fails** - Troubleshoot that server before proceeding

### Adding Server 1

- [ ] **Open deployed application in browser**
  - Navigate to your deployment URL
  - App should show "No chats yet - Add a server to get started"

- [ ] **Click "Add Server" button** (+ icon in server sidebar)

- [ ] **Fill in Server 1 details**:
  - [ ] **Name**: Enter friendly name (e.g., "Mac Studio")
  - [ ] **Server URL**: Enter full URL (e.g., `https://server1.example.com`)
    - ⚠️ Include `https://` or `http://`
    - ⚠️ Do NOT include trailing slash
    - ⚠️ Do NOT include `/api/v1` path
  - [ ] **Password**: Enter BlueBubbles server password

- [ ] **Click "Test Connection" button**
  - [ ] Should show "✓ Connection successful!"
  - [ ] If fails, verify URL and password are correct

- [ ] **Click "Add Server" button**
  - [ ] Server should appear in left sidebar with blue circle
  - [ ] Connection indicator should turn green
  - [ ] Chats should start loading automatically

- [ ] **Wait for initial sync** (may take 30-60 seconds)
  - [ ] Chats appear in middle panel
  - [ ] Click on a chat to verify messages load
  - [ ] Send a test message to verify sending works

- [ ] **Verify Server 1 is fully functional**
  - [ ] Can see all chats
  - [ ] Can click on chat and see messages
  - [ ] Can send a message
  - [ ] New messages appear in real-time (test from another device)

### Adding Server 2

- [ ] **Click "Add Server" button again**

- [ ] **Fill in Server 2 details**:
  - [ ] **Name**: "MacBook Pro" (or your chosen name)
  - [ ] **Server URL**: `https://server2.example.com`
  - [ ] **Password**: Server 2 password

- [ ] **Test connection** - Verify success

- [ ] **Add server**
  - [ ] Second server appears in sidebar
  - [ ] Has different color than Server 1

- [ ] **Wait for Server 2 chats to load**
  - [ ] New chats appear in middle panel
  - [ ] Chats that exist on both servers show indicator (e.g., "2 servers")
  - [ ] Small colored dots show which servers have each chat

- [ ] **Test aggregation** - Click on a chat that exists on both servers
  - [ ] Should see messages from BOTH servers
  - [ ] Messages should be in chronological order
  - [ ] Server indicator shows which server each message came from

### Adding Servers 3-8

Repeat the following for each remaining server:

#### Server 3

- [ ] Click "Add Server"
- [ ] Enter name: _____________ (your chosen name)
- [ ] Enter URL: _____________
- [ ] Enter password: _____________
- [ ] Test connection ✓
- [ ] Add server
- [ ] Verify chats load
- [ ] Test sending message
- [ ] Verify server color is unique

#### Server 4

- [ ] Click "Add Server"
- [ ] Enter name: _____________ (your chosen name)
- [ ] Enter URL: _____________
- [ ] Enter password: _____________
- [ ] Test connection ✓
- [ ] Add server
- [ ] Verify chats load
- [ ] Test sending message
- [ ] Verify server color is unique

#### Server 5

- [ ] Click "Add Server"
- [ ] Enter name: _____________ (your chosen name)
- [ ] Enter URL: _____________
- [ ] Enter password: _____________
- [ ] Test connection ✓
- [ ] Add server
- [ ] Verify chats load
- [ ] Test sending message
- [ ] Verify server color is unique

#### Server 6

- [ ] Click "Add Server"
- [ ] Enter name: _____________ (your chosen name)
- [ ] Enter URL: _____________
- [ ] Enter password: _____________
- [ ] Test connection ✓
- [ ] Add server
- [ ] Verify chats load
- [ ] Test sending message
- [ ] Verify server color is unique

#### Server 7

- [ ] Click "Add Server"
- [ ] Enter name: _____________ (your chosen name)
- [ ] Enter URL: _____________
- [ ] Enter password: _____________
- [ ] Test connection ✓
- [ ] Add server
- [ ] Verify chats load
- [ ] Test sending message
- [ ] Verify server color is unique

#### Server 8

- [ ] Click "Add Server"
- [ ] Enter name: _____________ (your chosen name)
- [ ] Enter URL: _____________
- [ ] Enter password: _____________
- [ ] Test connection ✓
- [ ] Add server
- [ ] Verify chats load
- [ ] Test sending message
- [ ] Verify server color is unique

### Verify All 8 Servers Connected

- [ ] **Check server sidebar** - Should show 8 servers
  - [ ] All 8 server icons visible
  - [ ] Each has unique color
  - [ ] All show green connection indicator
  - [ ] Counter shows "8/8" at bottom

- [ ] **Check chat list** - Should show aggregated chats
  - [ ] Chats that exist on multiple servers show "X servers" badge
  - [ ] Colored dots indicate which servers have each chat
  - [ ] All chats are visible (not duplicated)

- [ ] **Test cross-server messaging**
  - [ ] Find a chat that exists on multiple servers
  - [ ] Click on it
  - [ ] Messages from all servers appear in timeline
  - [ ] Each message shows server indicator
  - [ ] Send a message
  - [ ] Verify it appears on the server you sent from

---

## Post-Deployment Testing

### Functionality Testing

#### Basic Operations

- [ ] **Refresh browser** - Verify state persists
  - [ ] All 8 servers still connected
  - [ ] Chats still visible
  - [ ] Selected chat remains selected

- [ ] **Test chat selection**
  - [ ] Click different chats
  - [ ] Messages load for each chat
  - [ ] No errors in console

- [ ] **Test message sending**
  - [ ] Select a chat
  - [ ] Type a message
  - [ ] Press Enter to send
  - [ ] Message appears immediately
  - [ ] Message delivered to server (verify on another device)

- [ ] **Test real-time updates**
  - [ ] Send message from another device
  - [ ] Message appears in aggregator without refresh
  - [ ] Works for all 8 servers

#### Multi-Server Features

- [ ] **Test server indicators**
  - [ ] Each chat shows which servers it's on
  - [ ] Colors match server colors in sidebar
  - [ ] Can identify which server a message came from

- [ ] **Test aggregation**
  - [ ] Find chat that exists on multiple servers
  - [ ] Verify messages from all servers appear
  - [ ] Messages are in chronological order
  - [ ] No duplicate messages

- [ ] **Test server connection status**
  - [ ] All servers show green indicator
  - [ ] Hover over server to see tooltip (if implemented)
  - [ ] Counter shows correct count

#### Edge Cases

- [ ] **Test empty chat**
  - [ ] Find or create chat with no messages
  - [ ] Click on it
  - [ ] Should show "No messages yet" or similar
  - [ ] No errors

- [ ] **Test very long messages**
  - [ ] Send message with 500+ characters
  - [ ] Verify it displays correctly
  - [ ] No UI breaking

- [ ] **Test special characters**
  - [ ] Send message with emojis: 😀 🎉 ❤️
  - [ ] Send message with @ mentions
  - [ ] Send message with URLs
  - [ ] All render correctly

### Performance Testing

- [ ] **Test with many chats** (100+)
  - [ ] Scroll through chat list
  - [ ] Should be smooth
  - [ ] No lag

- [ ] **Test with many messages** (1000+)
  - [ ] Open chat with long history
  - [ ] Scroll through messages
  - [ ] Should be smooth

- [ ] **Test switching between chats rapidly**
  - [ ] Click through 10 different chats quickly
  - [ ] All should load without errors
  - [ ] No memory leaks (check browser DevTools)

### Browser Compatibility

- [ ] **Test in Chrome**
  - [ ] All features work
  - [ ] No console errors
  - [ ] UI looks correct

- [ ] **Test in Firefox**
  - [ ] All features work
  - [ ] No console errors
  - [ ] UI looks correct

- [ ] **Test in Safari** (if on Mac)
  - [ ] All features work
  - [ ] No console errors
  - [ ] UI looks correct

- [ ] **Test in Edge**
  - [ ] All features work
  - [ ] No console errors
  - [ ] UI looks correct

### Mobile Testing (If Applicable)

- [ ] **Test on mobile browser**
  - [ ] UI is responsive
  - [ ] Can navigate
  - [ ] Can send messages
  - [ ] Touch interactions work

---

## Monitoring & Maintenance

### Setup Monitoring

- [ ] **Browser Console Monitoring**
  - [ ] Open DevTools in browser
  - [ ] Check Console tab for errors
  - [ ] Check Network tab for failed requests
  - [ ] No errors should appear during normal use

- [ ] **Server Logs** (if self-hosted)
  - [ ] Monitor PM2 logs
    ```bash
    pm2 logs poa-chat-aggregator
    ```
  - [ ] Check for errors or warnings
  - [ ] Monitor CPU and memory usage
    ```bash
    pm2 monit
    ```

- [ ] **Uptime Monitoring** (Optional)
  - [ ] Set up UptimeRobot or similar
  - [ ] Monitor URL availability
  - [ ] Get alerts if site goes down

### Regular Maintenance Tasks

- [ ] **Weekly: Check server connections**
  - [ ] Verify all 8 servers still connected
  - [ ] Check green indicators in sidebar
  - [ ] Test sending messages on each

- [ ] **Weekly: Clear browser cache and test**
  - [ ] Hard refresh (Ctrl+Shift+R)
  - [ ] Verify everything still works
  - [ ] Servers reconnect automatically

- [ ] **Monthly: Check for updates**
  ```bash
  cd /home/user/poa-chat-aggregator
  npm outdated
  # Review outdated packages
  ```

- [ ] **Monthly: Review logs** (if self-hosted)
  ```bash
  pm2 logs poa-chat-aggregator --lines 1000 | grep -i error
  # Look for patterns or issues
  ```

### Backup Strategy

- [ ] **Document server credentials**
  - [ ] Store securely in password manager
  - [ ] Include all 8 server URLs and passwords
  - [ ] Include deployment credentials

- [ ] **Export database** (Optional - for debugging)
  - Open browser console and run:
    ```javascript
    // This exports IndexedDB data
    const { exportDatabase } = await import('/lib/db/index.js');
    const data = await exportDatabase();
    console.log(JSON.stringify(data, null, 2));
    ```
  - [ ] Save output for backup

---

## Troubleshooting

### Server Won't Connect

**Symptoms:** Server shows red indicator, won't connect

- [ ] **Check server URL format**
  - ✅ Correct: `https://server.example.com`
  - ❌ Wrong: `https://server.example.com/` (trailing slash)
  - ❌ Wrong: `https://server.example.com/api/v1` (includes path)
  - ❌ Wrong: `server.example.com` (missing https://)

- [ ] **Verify password is correct**
  - [ ] Try connecting to server API directly:
    ```bash
    curl https://your-server.com/api/v1/ping?guid=YOUR_PASSWORD
    ```
  - [ ] Should return `{"status":200,"message":"pong!"`

- [ ] **Check BlueBubbles server is running**
  - [ ] SSH into Mac running BlueBubbles
  - [ ] Verify BlueBubbles app is open
  - [ ] Check server logs for errors

- [ ] **Check network/firewall**
  - [ ] Verify deployment can reach server
  - [ ] Check firewall rules
  - [ ] Verify no VPN blocking connection

- [ ] **Check CORS settings** (if self-hosted BlueBubbles)
  - BlueBubbles should allow your deployment domain

### Messages Not Loading

**Symptoms:** Chat opens but messages don't appear

- [ ] **Check browser console for errors**
  - Open DevTools → Console
  - Look for red errors
  - Note any failed network requests

- [ ] **Check Network tab**
  - Open DevTools → Network
  - Filter by XHR
  - Look for failed requests to `/api/v1/chat/...`

- [ ] **Verify server connection**
  - [ ] Check server has green indicator
  - [ ] Try disconnecting and reconnecting server

- [ ] **Clear IndexedDB and retry**
  - Open DevTools → Application → IndexedDB
  - Right-click "ChatAggregatorDB" → Delete database
  - Refresh page
  - Reconnect servers

### Real-Time Messages Not Appearing

**Symptoms:** Have to refresh to see new messages

- [ ] **Check WebSocket connection**
  - Open DevTools → Network → WS (WebSocket)
  - Should see active WebSocket connections (one per server)
  - Status should be "101 Switching Protocols"

- [ ] **Check browser console**
  - Look for WebSocket connection errors
  - Should see messages like `[server-id] Connected to WebSocket`

- [ ] **Verify BlueBubbles WebSocket is enabled**
  - In BlueBubbles server settings
  - Socket server should be running

- [ ] **Firewall blocking WebSockets**
  - Some corporate firewalls block WebSocket
  - Try from different network

### App is Slow

**Symptoms:** Laggy UI, slow loading

- [ ] **Check number of messages in IndexedDB**
  - Open DevTools → Application → IndexedDB → ChatAggregatorDB
  - If messages table has 50,000+ entries, may be slow
  - Consider clearing old data

- [ ] **Check browser memory usage**
  - Open DevTools → Memory
  - Take heap snapshot
  - Look for memory leaks

- [ ] **Check CPU usage**
  - Open DevTools → Performance
  - Record session while using app
  - Look for slow functions

- [ ] **Try different browser**
  - Some browsers handle IndexedDB better than others
  - Chrome generally fastest

### Deployment Fails

**Symptoms:** Build fails or deploy errors

- [ ] **Check build locally first**
  ```bash
  cd /home/user/poa-chat-aggregator
  rm -rf .next node_modules
  npm install
  npm run build
  ```

- [ ] **Review error messages**
  - Note exact error message
  - Google error message
  - Check Stack Overflow

- [ ] **Check Node.js version**
  - Must be 18.x or higher
  - Some platforms default to older versions

- [ ] **Check disk space** (if self-hosted)
  ```bash
  df -h
  # Ensure enough space for build
  ```

### Can't Remove Server

**Symptoms:** Server won't delete

- [ ] **Try disconnecting first**
  - Click server
  - Look for disconnect option
  - Then try deleting

- [ ] **Clear IndexedDB manually**
  - Open DevTools → Application → IndexedDB
  - Expand ChatAggregatorDB → servers
  - Find server entry and delete

- [ ] **Hard refresh and retry**
  - Press Ctrl+Shift+R
  - Try deleting again

---

## Quick Reference Commands

### Local Development
```bash
# Start development server
npm run dev

# Build for production
npm run build

# Start production server
npm start

# Run linter
npm run lint
```

### Vercel
```bash
# Deploy to Vercel
vercel

# Deploy to production
vercel --prod

# Check deployment status
vercel ls

# View logs
vercel logs
```

### Self-Hosted (PM2)
```bash
# Start app
pm2 start ecosystem.config.js

# Stop app
pm2 stop poa-chat-aggregator

# Restart app
pm2 restart poa-chat-aggregator

# View logs
pm2 logs poa-chat-aggregator

# Monitor
pm2 monit

# View status
pm2 status
```

### Testing Server Connectivity
```bash
# Test server ping
curl https://your-server.com/api/v1/ping?guid=YOUR_PASSWORD

# Test server info
curl https://your-server.com/api/v1/server/info?guid=YOUR_PASSWORD

# Test chats endpoint
curl https://your-server.com/api/v1/chat/query?guid=YOUR_PASSWORD \
  -H "Content-Type: application/json" \
  -d '{"limit":10}'
```

---

## Success Criteria

Your deployment is successful when:

- [ ] ✅ All 8 BlueBubbles servers are connected
- [ ] ✅ All servers show green connection indicators
- [ ] ✅ Chats load from all servers
- [ ] ✅ Messages from multiple servers aggregate correctly
- [ ] ✅ Can send messages successfully
- [ ] ✅ Real-time messages appear without refresh
- [ ] ✅ App works across page refreshes
- [ ] ✅ No errors in browser console
- [ ] ✅ Deployment is accessible from intended devices
- [ ] ✅ HTTPS enabled (if using custom domain)

---

## Next Steps After Deployment

1. **Share with team** - Send deployment URL to teammates
2. **Gather feedback** - Ask team to test and report issues
3. **Monitor usage** - Watch for errors or performance issues
4. **Iterate** - Fix bugs and add features based on feedback
5. **Document issues** - Keep track of what needs improvement
6. **Plan enhancements** - Prioritize based on user needs

---

**Deployment Complete!** 🎉

You now have a fully functional multi-server BlueBubbles chat aggregator running with 8 connected servers.
