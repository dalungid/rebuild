# 🚀 Gaming Platform Rebuild - Master Plan

## Executive Summary

Rebuild dari Laravel monolith ke modern **microservices architecture** dengan Next.js frontend dan NestJS backend services, deployed pada Kubernetes cluster di existing VPS (32GB RAM, 8 CPU).

**Current System**: Laravel 10 monolith dengan multiple game provider integrations  
**Target System**: Microservices dengan single game API (DominicEngineAPI)  
**Infrastructure**: k3s Kubernetes + Rancher management pada dedicated VPS  
**Timeline**: 12-16 minggu (3-4 bulan)

---

## 📊 Current System Analysis

### Technology Stack (Existing)
- **Framework**: Laravel 10.x (PHP 8.1+)
- **Database**: MySQL/MariaDB
- **Cache**: Redis (predis)
- **Frontend**: Blade templates + jQuery
- **Deployment**: FastPanel (traditional LAMP stack)
- **Server**: Single VPS

### Core Features
1. **Multi-provider Gaming Platform**
   - Slots, Live Casino, Sports, Fishing, Lottery, P2P, Cockfight
   - Unified via DominicEngineAPI (single integration point)
   
2. **Payment System**
   - Multiple gateways: Hokipay, QRIS, Akses Bayar
   - Automated withdrawal: VIP Otomatis & GsPay
   - Deposit & Withdraw management
   
3. **User Management**
   - User registration (no OTP required)
   - Multi-level admin system
   - IP whitelist protection
   - Ban/unban functionality
   
4. **Referral System**
   - 2-level referral tracking
   - Promotor dengan custom rates
   - Commission calculation & claims
   - NDP (New Deposit Player) tracking
   
5. **Admin Features**
   - Transaction approval workflow
   - Member management
   - Banner & promotion management
   - Domain aliases (multi-domain support)
   - CRM dashboard
   - Analytics & turnover reports
   - Settings management (payment configs, website settings)

### Key Integration: DominicEngineAPI

**Simplified Game Integration** - All game providers unified behind single API:

```typescript
DominicEngineAPI endpoints:
- createMember(username)           // Create game account
- getBalanceMember(username)       // Check balance
- depositMember(username, amount)  // Transfer to game
- withdrawMember(username, amount) // Transfer from game
- launchGame(username, idGame)     // Get game URL
- getGame(provider)                // Get game list
- listProvider()                   // List providers
- getTurnoverSingleUser()          // Get bet history
```

**Critical Insight**: Tidak perlu integrate dengan multiple provider APIs - hanya 1 API endpoint!

---

## 🎯 Target Architecture

### Technology Stack (New)

```yaml
Frontend:
  - Next.js 14+ (App Router, TypeScript)
  - TailwindCSS (styling)
  - React Query (state management & caching)
  - Zustand (client state)

Backend:
  - NestJS microservices (TypeScript)
  - Prisma ORM (PostgreSQL)
  - ioredis (Redis client)
  - Bull Queue (background jobs)

API Gateway:
  - Traefik (ingress controller)
  - Rate limiting via Redis
  - JWT validation middleware
  - IP whitelist middleware

Database:
  - PostgreSQL 15+ (primary transactional data)
  - Redis 7+ (cache, sessions, queues)

Storage:
  - S3-compatible storage
  - CloudFlare CDN

Infrastructure:
  - k3s (lightweight Kubernetes)
  - Rancher (cluster management UI)
  - Helm Charts (package management)
  - ArgoCD (GitOps deployment)

Monitoring:
  - Prometheus (metrics collection)
  - Grafana (visualization)
  - Loki (log aggregation)
  - Sentry (error tracking)
```

### Microservices Architecture

```
┌─────────────────────────────────────────────────────────┐
│              TRAEFIK (API Gateway + LB)                 │
│      SSL, Rate Limiting, IP Whitelist, Routing          │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
┌───────▼──────┐ ┌──▼────────┐ ┌─▼──────────┐
│   Next.js    │ │  NestJS   │ │   Admin    │
│  (Player)    │ │  API GW   │ │   Panel    │
│   Frontend   │ │           │ │ (Next.js)  │
└──────────────┘ └──┬────────┘ └────────────┘
                    │
     ┌──────────────┼──────────────────────┐
     │              │                      │
┌────▼─────┐ ┌─────▼──────┐ ┌─────────────▼─────┐
│  Auth    │ │  Payment   │ │   Transaction     │
│ Service  │ │  Service   │ │     Service       │
└──────────┘ └────────────┘ └───────────────────┘
     │              │                      │
┌────▼─────┐ ┌─────▼──────┐ ┌─────────────▼─────┐
│  User    │ │ Referral   │ │    Analytics      │
│ Service  │ │  Service   │ │     Service       │
└──────────┘ └────────────┘ └───────────────────┘
     │              │                      │
     └──────────────┼──────────────────────┘
                    │
     ┌──────────────┼──────────────────┐
     │              │                  │
┌────▼─────┐ ┌─────▼──────┐ ┌─────────▼─────┐
│   Game   │ │PostgreSQL  │ │     Redis     │
│  Proxy   │ │  (Master)  │ │    (Cache)    │
│ Service  │ └────────────┘ └───────────────┘
└────┬─────┘
     │ Single API Integration
     ▼
┌──────────────────────────────┐
│   DominicEngineAPI           │
│   (External - All Games)     │
└──────────────────────────────┘
```

---

## 🗄️ Database Schema

### PostgreSQL Tables (Minimal but Complete)

```sql
-- ==================== USERS & AUTH ====================
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  username VARCHAR(50) UNIQUE NOT NULL,
  email VARCHAR(100) UNIQUE NOT NULL,
  phone VARCHAR(20) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  
  -- Bank Info
  bank_name VARCHAR(50),
  bank_account VARCHAR(50),
  bank_holder VARCHAR(100),
  bank_code VARCHAR(10),
  
  -- Status
  is_banned BOOLEAN DEFAULT FALSE,
  banned_reason TEXT,
  
  -- Referral
  referral_code VARCHAR(20) UNIQUE,
  upline_user_id BIGINT REFERENCES users(id),
  is_promotor BOOLEAN DEFAULT FALSE,
  promotor_rate DECIMAL(5,2), -- Custom rate untuk promotor
  
  -- Meta
  created_by BIGINT REFERENCES users(id), -- Admin who created
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_referral_code ON users(referral_code);
CREATE INDEX idx_users_upline ON users(upline_user_id);
CREATE INDEX idx_users_is_banned ON users(is_banned);

-- User Balances (separated for concurrency)
CREATE TABLE user_balances (
  user_id BIGINT PRIMARY KEY REFERENCES users(id),
  main_balance DECIMAL(15,2) DEFAULT 0.00,
  referral_balance DECIMAL(15,2) DEFAULT 0.00,
  last_synced_at TIMESTAMP,
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Sessions
CREATE TABLE user_sessions (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT REFERENCES users(id),
  token_hash VARCHAR(255) UNIQUE NOT NULL,
  ip_address INET,
  device_info TEXT,
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_sessions_user_id ON user_sessions(user_id);
CREATE INDEX idx_sessions_expires ON user_sessions(expires_at);

-- ==================== TRANSACTIONS ====================
CREATE TYPE transaction_type AS ENUM ('deposit', 'withdraw', 'bonus', 'commission', 'adjustment');
CREATE TYPE transaction_status AS ENUM ('pending', 'processing', 'success', 'failed', 'cancelled');

CREATE TABLE transactions (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT REFERENCES users(id),
  type transaction_type NOT NULL,
  amount DECIMAL(15,2) NOT NULL,
  status transaction_status DEFAULT 'pending',
  
  -- Payment Info
  payment_method VARCHAR(50), -- bank_transfer, qris, ewallet, dll
  payment_gateway VARCHAR(50), -- hokipay, gspay, aksesbayar, dll
  gateway_ref_id VARCHAR(100), -- Reference ID dari payment gateway
  gateway_response JSONB, -- Full response untuk debugging
  
  -- Approval
  admin_id BIGINT REFERENCES users(id), -- Admin who approved/rejected
  approved_at TIMESTAMP,
  
  -- Additional Info
  notes TEXT,
  metadata JSONB, -- Extra data (bank name, account, dll)
  
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_transactions_user_id ON transactions(user_id);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_transactions_type ON transactions(type);
CREATE INDEX idx_transactions_gateway_ref ON transactions(gateway_ref_id);
CREATE INDEX idx_transactions_created_at ON transactions(created_at DESC);

-- ==================== REFERRAL ====================
CREATE TABLE referral_commissions (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT REFERENCES users(id), -- Yang dapat komisi
  downline_id BIGINT REFERENCES users(id), -- Downline yang deposit
  transaction_id BIGINT REFERENCES transactions(id), -- Transaction yang generate komisi
  
  amount DECIMAL(15,2) NOT NULL,
  percentage DECIMAL(5,2) NOT NULL, -- Rate saat itu
  
  claimed BOOLEAN DEFAULT FALSE,
  claimed_at TIMESTAMP,
  
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_referral_commissions_user_id ON referral_commissions(user_id);
CREATE INDEX idx_referral_commissions_claimed ON referral_commissions(claimed);

-- Promotor NDP Tracking
CREATE TABLE promotor_ndp_tracking (
  id BIGSERIAL PRIMARY KEY,
  promotor_id BIGINT REFERENCES users(id),
  downline_id BIGINT REFERENCES users(id),
  first_deposit_amount DECIMAL(15,2),
  first_deposit_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_promotor_ndp_promotor ON promotor_ndp_tracking(promotor_id);

-- ==================== CMS & CONTENT ====================
CREATE TABLE banners (
  id SERIAL PRIMARY KEY,
  title VARCHAR(100),
  image_url TEXT NOT NULL,
  link TEXT, -- Redirect URL when clicked
  position VARCHAR(20), -- home, deposit, game, dll
  active BOOLEAN DEFAULT TRUE,
  sort_order INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE promotions (
  id SERIAL PRIMARY KEY,
  title VARCHAR(200) NOT NULL,
  description TEXT,
  image_url TEXT,
  terms TEXT, -- Syarat & ketentuan
  active BOOLEAN DEFAULT TRUE,
  start_date TIMESTAMP,
  end_date TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- ==================== DOMAIN MANAGEMENT ====================
CREATE TYPE domain_status AS ENUM ('pending', 'active', 'failed', 'deleted');

CREATE TABLE domain_aliases (
  id SERIAL PRIMARY KEY,
  domain VARCHAR(255) UNIQUE NOT NULL,
  status domain_status DEFAULT 'pending',
  
  -- Cloudflare Integration
  cloudflare_zone_id VARCHAR(100),
  cloudflare_status VARCHAR(50),
  
  -- Nginx Config
  nginx_config_path TEXT,
  
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- ==================== SETTINGS ====================
CREATE TABLE settings (
  key VARCHAR(100) PRIMARY KEY,
  value JSONB NOT NULL,
  category VARCHAR(50), -- website, payment, game, contact, dll
  description TEXT,
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Example settings data:
-- key: 'payment.hokipay', value: { "api_key": "...", "secret": "...", "enabled": true }
-- key: 'game.dominic_engine', value: { "agent_code": "...", "agent_token": "...", "url": "..." }
-- key: 'website.name', value: "Gaming Platform"

-- ==================== ADMIN ====================
CREATE TABLE admins (
  id BIGSERIAL PRIMARY KEY,
  username VARCHAR(50) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  level INT DEFAULT 3, -- 1=super, 2=admin, 3=cs
  permissions JSONB, -- Custom permissions
  last_login_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE admin_activities (
  id BIGSERIAL PRIMARY KEY,
  admin_id BIGINT REFERENCES admins(id),
  action VARCHAR(100) NOT NULL, -- 'approve_deposit', 'reject_withdraw', 'ban_user', dll
  resource_type VARCHAR(50), -- 'transaction', 'user', 'setting', dll
  resource_id BIGINT,
  
  before_data JSONB, -- Data sebelum perubahan
  after_data JSONB, -- Data sesudah perubahan
  
  ip_address INET,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_admin_activities_admin_id ON admin_activities(admin_id);
CREATE INDEX idx_admin_activities_created_at ON admin_activities(created_at DESC);

CREATE TABLE whitelist_ips (
  id SERIAL PRIMARY KEY,
  ip_address INET UNIQUE NOT NULL,
  label VARCHAR(100), -- 'Office', 'Admin Home', dll
  active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT NOW()
);

-- ==================== GAME DATA (Optional - bisa di cache) ====================
CREATE TABLE game_sessions (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT REFERENCES users(id),
  provider_code VARCHAR(50),
  game_id VARCHAR(100),
  game_url TEXT,
  
  bet_amount DECIMAL(15,2),
  win_amount DECIMAL(15,2),
  
  started_at TIMESTAMP DEFAULT NOW(),
  ended_at TIMESTAMP
);

CREATE INDEX idx_game_sessions_user_id ON game_sessions(user_id);
CREATE INDEX idx_game_sessions_started_at ON game_sessions(started_at DESC);
```

### Redis Cache Structure

```redis
# ==================== SESSIONS ====================
session:user:{user_id}:{token_hash}     → { userId, username, ... } [TTL: 24h]
session:admin:{admin_id}:{token_hash}   → { adminId, level, ... } [TTL: 8h]

# ==================== BALANCE CACHE ====================
balance:{user_id}                       → { main: 1000.50, referral: 250.00 } [TTL: 5m]
balance:game:{username}                 → 850.25 [TTL: 2m] // From DominicEngine

# ==================== GAME DATA CACHE ====================
game:providers                          → [{ code, name, ... }] [TTL: 24h]
game:list:{provider_code}               → [{ id, name, image, ... }] [TTL: 1h]

# ==================== SETTINGS CACHE ====================
settings:all                            → { website: {...}, payment: {...} } [TTL: 1h]
settings:{category}                     → { ... } [TTL: 1h]

# ==================== RATE LIMITING ====================
ratelimit:login:{ip}                    → 5 [TTL: 15m] // Max 5 attempts
ratelimit:otp:{phone}                   → 3 [TTL: 5m] // Max 3 OTP requests
ratelimit:deposit:{user_id}             → 5 [TTL: 1h] // Max 5 deposits/hour
ratelimit:withdraw:{user_id}            → 3 [TTL: 1h] // Max 3 withdrawals/hour
ratelimit:api:{ip}                      → 100 [TTL: 1m] // Max 100 req/min

# ==================== IDEMPOTENCY ====================
payment:idempotency:{gateway}:{ref_id}  → transaction_id [TTL: 24h]

# ==================== DISTRIBUTED LOCKS ====================
lock:balance:{user_id}                  → "transaction_id" [TTL: 30s]
lock:withdraw:{transaction_id}          → "1" [TTL: 60s]

# ==================== QUEUES (Bull) ====================
queue:payment:callbacks                 → [job1, job2, ...]
queue:referral:commissions              → [job1, job2, ...]
queue:notifications                     → [job1, job2, ...]
```

---

## 🔧 Microservices Detail

### 1. Auth Service

**Responsibilities:**
- User authentication (login/logout)
- JWT token generation & validation
- Session management
- Admin authentication
- IP whitelist validation

**Tech Stack:**
- NestJS + TypeScript
- Passport.js (JWT strategy)
- bcrypt (password hashing)
- node-otp (OTP generation)

**API Endpoints:**
```typescript
POST   /auth/register          // Direct registration (no OTP)
POST   /auth/login             // Login (username/password)
POST   /auth/logout            // Logout & invalidate token
POST   /auth/refresh           // Refresh JWT token
GET    /auth/me                // Get current user info
POST   /auth/admin/login       // Admin login with IP check
```

**Database Tables:**
- `users` (read)
- `user_sessions` (write)
- `admins` (read)
- `whitelist_ips` (read)

**Redis Keys:**
- `session:user:{userId}:{token}`
- `session:admin:{adminId}:{token}`
- `ratelimit:login:{ip}`
- `ratelimit:otp:{phone}`

---

### 2. User Service

**Responsibilities:**
- User profile management
- User bank account management
- KYC verification
- Ban/unban users
- User search & listing

**API Endpoints:**
```typescript
GET    /users/:id              // Get user profile
PUT    /users/:id              // Update user profile
PUT    /users/:id/bank         // Update bank info
POST   /users/:id/ban          // Ban user
POST   /users/:id/unban        // Unban user
GET    /users/search           // Search users (admin)
GET    /users/:id/downlines    // Get referral downlines
```

**Database Tables:**
- `users` (read/write)
- `user_balances` (read)

---

### 3. Game Proxy Service

**Responsibilities:**
- Proxy to DominicEngineAPI
- Create game member on first launch
- Launch game URLs
- Get game list (cached)
- Get provider list (cached)
- Sync balance between platform & game
- Get turnover/bet history

**Critical Features:**
- Circuit breaker (prevent cascade failures)
- Retry mechanism with exponential backoff
- Request/response logging
- Cache management

**API Endpoints:**
```typescript
POST   /game/launch/:gameId           // Launch game
GET    /game/providers                // List all providers
GET    /game/list/:provider           // List games by provider
GET    /game/balance/:username        // Get game balance
POST   /game/sync/:username           // Sync balance
GET    /game/turnover/:username       // Get bet history
POST   /game/member/create            // Create game member
```

**External API:**
- DominicEngineAPI (https://api.indortp77.site/)

**Cache Strategy:**
```typescript
Providers list:     24 hours (rarely changes)
Game list:          1 hour (updated occasionally)
Game balance:       2 minutes (sync frequently)
```

---

### 4. Payment Service

**Responsibilities:**
- Deposit processing
- Withdraw processing
- Payment gateway integrations
- Webhook/callback handling
- Balance operations (atomic)
- Transaction creation & updates

**Payment Gateways:**
- Hokipay (deposit)
- QRIS - multiple providers (deposit)
- Akses Bayar (deposit)
- GsPay (deposit + automated withdrawal)
- VIP Otomatis (automated withdrawal)

**API Endpoints:**
```typescript
POST   /payment/deposit/initiate         // Initiate deposit
POST   /payment/withdraw/initiate        // Initiate withdrawal
POST   /payment/callback/hokipay         // Hokipay webhook
POST   /payment/callback/gspay           // GsPay webhook
POST   /payment/callback/aksesbayar      // Akses Bayar webhook
GET    /payment/status/:transactionId    // Check transaction status
PUT    /payment/deposit/:id/confirm      // Admin approve deposit
PUT    /payment/deposit/:id/reject       // Admin reject deposit
PUT    /payment/withdraw/:id/confirm     // Admin approve withdrawal
PUT    /payment/withdraw/:id/reject      // Admin reject withdrawal
```

**Critical Flows:**

**Deposit Flow:**
```
1. User request deposit (amount, method)
2. Create transaction (status: pending)
3. Call payment gateway API
4. Receive callback from gateway
5. Validate signature & idempotency
6. Update transaction (status: success)
7. Call Game Proxy → depositMember()
8. Update user balance in DB
9. Invalidate balance cache
10. Send notification to user
```

**Withdraw Flow:**
```
1. User request withdraw (amount)
2. Check user balance (cache + DB)
3. Call Game Proxy → withdrawMember()
4. If success: Create transaction (pending)
5. Admin approval OR VIP Otomatis auto-process
6. Disburse to user bank account
7. Update transaction (status: success)
8. Update user balance
9. Invalidate cache
10. Send notification
```

**Database Tables:**
- `transactions` (write/read)
- `user_balances` (write)
- `settings` (read - payment configs)

**Redis:**
- `lock:balance:{userId}`
- `payment:idempotency:{gateway}:{ref}`
- Cache invalidation on balance change

---

### 5. Transaction Service

**Responsibilities:**
- Transaction history & listing
- Transaction reports
- Admin approval queue
- Audit logs

**API Endpoints:**
```typescript
GET    /transactions                    // List all (with filters)
GET    /transactions/:id                // Get transaction detail
GET    /transactions/pending            // Pending approvals (admin)
GET    /transactions/user/:userId       // User transaction history
GET    /transactions/export             // Export to CSV/Excel
```

---

### 6. Referral Service

**Responsibilities:**
- Referral code generation
- Track downlines
- Commission calculation
- Claim referral balance
- Promotor management

**API Endpoints:**
```typescript
GET    /referral/code                   // Get user referral code
GET    /referral/downlines              // List downlines
GET    /referral/commissions            // List commissions
POST   /referral/claim                  // Claim referral balance
GET    /referral/stats                  // Referral statistics
GET    /referral/promotors              // List promotors (admin)
PUT    /referral/promotor/:id/rate      // Update promotor rate (admin)
```

**Database Tables:**
- `users` (read - for downline tracking)
- `referral_commissions` (read/write)
- `promotor_ndp_tracking` (write)

---

### 7. Analytics Service

**Responsibilities:**
- Player analytics
- Turnover calculation & reports
- CRM dashboard data
- Revenue reports
- Admin dashboard statistics

**API Endpoints:**
```typescript
GET    /analytics/dashboard             // Admin dashboard data
GET    /analytics/player/:userId        // Player analytics
GET    /analytics/turnover              // Turnover reports
GET    /analytics/revenue               // Revenue reports
GET    /analytics/trends                // Trends & insights
```

---

## 🏗️ Infrastructure Setup

### VPS Specifications
- **RAM**: 32GB
- **CPU**: 8 cores (16 threads)
- **Storage**: (to be determined based on needs)
- **OS**: Ubuntu 22.04 LTS (recommended)

### Infrastructure Stack

```yaml
Kubernetes Distribution: k3s
  - Lightweight K8s (perfect for single-node)
  - Built-in Traefik (will be replaced)
  - Low resource overhead

Cluster Management: Rancher
  - Web UI for K8s management
  - Helm catalog & app marketplace
  - Monitoring (Prometheus + Grafana)
  - Logging (Loki)
  - GitOps support

Ingress Controller: Traefik v2.10+
  - SSL/TLS termination (Let's Encrypt)
  - Rate limiting
  - Load balancing
  - Middleware support

Certificate Management: cert-manager
  - Automatic SSL certificate generation
  - Let's Encrypt integration
  - Auto-renewal

GitOps: ArgoCD
  - Declarative deployments
  - Git as source of truth
  - Auto-sync from repository
  - Easy rollbacks

Monitoring Stack:
  - Prometheus (metrics)
  - Grafana (visualization)
  - Loki (logs)
  - Alertmanager (alerts)

Error Tracking: Sentry
  - Application error tracking
  - Performance monitoring
  - Release tracking
```

### Installation Steps

#### Step 1: Install k3s
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install k3s (disable traefik, we'll install via Rancher)
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -

# Setup kubeconfig
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
export KUBECONFIG=~/.kube/config

# Verify
kubectl get nodes
```

#### Step 2: Install Helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

#### Step 3: Install Rancher
```bash
# Install cert-manager (required by Rancher)
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --set installCRDs=true \
  --wait

# Install Rancher
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
kubectl create namespace cattle-system
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.yourdomain.com \
  --set replicas=1 \
  --set bootstrapPassword=admin \
  --wait

# Access Rancher
# Setup port forwarding if no domain yet:
kubectl -n cattle-system port-forward svc/rancher 8443:443 --address=0.0.0.0

# Visit: https://your-vps-ip:8443
# Username: admin
# Password: admin
```

#### Step 4: Install Traefik via Rancher
```bash
# Via Rancher UI:
# 1. Apps & Marketplace
# 2. Search "Traefik"
# 3. Install with custom values:

# Or via Helm:
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --set ports.web.port=80 \
  --set ports.websecure.port=443 \
  --set certificatesResolvers.letsencrypt.acme.email=your@email.com \
  --set certificatesResolvers.letsencrypt.acme.storage=/data/acme.json
```

#### Step 5: Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access ArgoCD UI
kubectl -n argocd port-forward svc/argocd-server 8080:443 --address=0.0.0.0

# Get initial password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

#### Step 6: Install Monitoring Stack
```bash
# Install kube-prometheus-stack (Prometheus + Grafana + Alertmanager)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin

# Access Grafana
kubectl -n monitoring port-forward svc/monitoring-grafana 3000:80 --address=0.0.0.0
# Visit: http://your-vps-ip:3000
# Username: admin, Password: admin
```

#### Step 7: Deploy Databases

**PostgreSQL:**
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install postgresql bitnami/postgresql \
  --namespace database \
  --create-namespace \
  --set auth.username=appuser \
  --set auth.password=securepassword \
  --set auth.database=gaming_platform \
  --set primary.persistence.size=20Gi
```

**Redis:**
```bash
helm install redis bitnami/redis \
  --namespace database \
  --set auth.password=redispassword \
  --set replica.replicaCount=1 \
  --set master.persistence.size=5Gi
```

---

## 📁 Project Structure

### Monorepo Layout (Recommended)

```
gaming-platform/
├── apps/
│   ├── frontend/                # Next.js player frontend
│   │   ├── src/
│   │   │   ├── app/            # App router pages
│   │   │   ├── components/     # React components
│   │   │   ├── lib/            # Utilities
│   │   │   └── hooks/          # Custom hooks
│   │   ├── public/
│   │   ├── package.json
│   │   └── next.config.js
│   │
│   ├── admin/                   # Next.js admin panel
│   │   └── ... (similar structure)
│   │
│   └── api-gateway/             # NestJS API Gateway
│       ├── src/
│       ├── package.json
│       └── nest-cli.json
│
├── services/
│   ├── auth-service/            # NestJS Auth Service
│   │   ├── src/
│   │   │   ├── auth/           # Auth module
│   │   │   ├── common/         # Shared code
│   │   │   └── main.ts
│   │   ├── Dockerfile
│   │   └── package.json
│   │
│   ├── user-service/            # NestJS User Service
│   ├── game-proxy-service/      # NestJS Game Proxy
│   ├── payment-service/         # NestJS Payment
│   ├── transaction-service/     # NestJS Transaction
│   ├── referral-service/        # NestJS Referral
│   └── analytics-service/       # NestJS Analytics
│
├── packages/
│   ├── shared-types/            # Shared TypeScript types
│   ├── shared-config/           # Shared configs
│   └── shared-utils/            # Shared utilities
│
├── infrastructure/
│   ├── k8s/                     # Kubernetes manifests
│   │   ├── base/               # Base configs
│   │   ├── overlays/
│   │   │   ├── dev/
│   │   │   ├── staging/
│   │   │   └── production/
│   │   └── apps/
│   │       ├── auth-service.yaml
│   │       ├── user-service.yaml
│   │       └── ...
│   │
│   ├── helm/                    # Helm charts
│   │   ├── gaming-platform/
│   │   │   ├── Chart.yaml
│   │   │   ├── values.yaml
│   │   │   └── templates/
│   │   └── dependencies/
│   │
│   └── terraform/               # Infrastructure as Code (optional)
│       ├── main.tf
│       └── variables.tf
│
├── scripts/
│   ├── deploy.sh               # Deployment scripts
│   ├── migrate.sh              # Database migration
│   └── seed.sh                 # Seed data
│
├── docs/
│   ├── api/                    # API documentation
│   ├── architecture/           # Architecture diagrams
│   └── runbooks/               # Operational guides
│
├── .github/
│   └── workflows/
│       ├── ci.yml              # Continuous Integration
│       └── cd.yml              # Continuous Deployment
│
├── package.json                # Root package.json (monorepo)
├── pnpm-workspace.yaml         # pnpm workspace config
├── turbo.json                  # Turborepo config
├── docker-compose.yml          # Local development
└── README.md
```

---

## 🚀 Development Workflow

### Local Development

**Prerequisites:**
```bash
# Install Node.js 20 LTS
nvm install 20
nvm use 20

# Install pnpm (faster than npm)
npm install -g pnpm

# Install Docker & Docker Compose
# For Ubuntu:
sudo apt install docker.io docker-compose
```

**Setup Local Environment:**
```bash
# Clone repository
git clone <your-repo>
cd gaming-platform

# Install dependencies
pnpm install

# Start local databases (Docker Compose)
docker-compose up -d

# Run migrations
pnpm db:migrate

# Seed data
pnpm db:seed

# Start all services (Turborepo parallel execution)
pnpm dev
```

**Development URLs:**
```
Frontend:         http://localhost:3000
Admin Panel:      http://localhost:3001
API Gateway:      http://localhost:4000
Auth Service:     http://localhost:4001
User Service:     http://localhost:4002
Game Proxy:       http://localhost:4003
Payment Service:  http://localhost:4004
PostgreSQL:       localhost:5432
Redis:            localhost:6379
```

### Git Workflow

```
main                    # Production branch
  ├── develop          # Development branch
      ├── feature/xxx  # Feature branches
      └── hotfix/xxx   # Hotfix branches
```

**Branch Protection:**
- `main`: No direct push, require PR review
- `develop`: No direct push, require PR
- Feature branches: Created from `develop`

### CI/CD Pipeline

**GitHub Actions Workflow:**

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [develop, main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v3
        with:
          node-version: 20
          cache: 'pnpm'
      
      - run: pnpm install
      - run: pnpm lint
      - run: pnpm test
      - run: pnpm build

  docker-build:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v3
      - uses: docker/build-push-action@v4
        with:
          context: .
          push: false
          tags: gaming-platform:${{ github.sha }}
```

```yaml
# .github/workflows/cd.yml
name: CD

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      # Build & Push Docker images
      - uses: docker/build-push-action@v4
        with:
          push: true
          tags: |
            your-registry/auth-service:${{ github.sha }}
            your-registry/auth-service:latest
      
      # Update ArgoCD deployment
      - name: Update K8s manifests
        run: |
          sed -i "s/IMAGE_TAG/${{ github.sha }}/g" infrastructure/k8s/overlays/production/kustomization.yaml
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add infrastructure/k8s/overlays/production/
          git commit -m "Update image to ${{ github.sha }}"
          git push
```

**ArgoCD automatically syncs changes from Git to Kubernetes!**

---

## 📅 Implementation Timeline

### Phase 1: Infrastructure Setup (Week 1-2)

**Tasks:**
- [x] Setup VPS (already done - 32GB RAM, 8 CPU)
- [ ] Install k3s Kubernetes
- [ ] Install Rancher management UI
- [ ] Configure domain & DNS
- [ ] Install Traefik ingress controller
- [ ] Install cert-manager (SSL)
- [ ] Setup PostgreSQL & Redis
- [ ] Install ArgoCD
- [ ] Install monitoring stack (Prometheus + Grafana)
- [ ] Install Sentry for error tracking

**Deliverables:**
- Kubernetes cluster running
- Rancher accessible via web UI
- SSL certificates working
- Databases ready
- Monitoring dashboards configured

---

### Phase 2: Core Services Development (Week 3-8)

**Week 3-4: Auth & User Services**
- [ ] Setup NestJS boilerplate
- [ ] Auth Service: JWT authentication
- [ ] Auth Service: OTP system
- [ ] Auth Service: Admin login with IP whitelist
- [ ] User Service: User CRUD
- [ ] User Service: Bank info management
- [ ] User Service: Ban/unban functionality
- [ ] Database migrations
- [ ] Unit tests
- [ ] Deploy to K8s (dev environment)

**Week 4-5: Game Proxy Service**
- [ ] DominicEngineAPI client implementation
- [ ] Circuit breaker pattern
- [ ] Retry mechanism
- [ ] Create member endpoint
- [ ] Launch game endpoint
- [ ] Get game list (with caching)
- [ ] Get providers (with caching)
- [ ] Balance sync logic
- [ ] Turnover/history endpoint
- [ ] Integration tests with real API
- [ ] Deploy to K8s

**Week 5-7: Payment Service**
- [ ] Transaction model & database
- [ ] Deposit flow implementation
- [ ] Withdraw flow implementation
- [ ] Hokipay integration
- [ ] QRIS integration
- [ ] GsPay integration
- [ ] Akses Bayar integration
- [ ] VIP Otomatis integration
- [ ] Webhook/callback handlers
- [ ] Idempotency implementation
- [ ] Admin approval endpoints
- [ ] Balance operation (atomic with locks)
- [ ] Integration tests
- [ ] Deploy to K8s

**Week 7-8: Transaction & Referral Services**
- [ ] Transaction Service: History & listing
- [ ] Transaction Service: Reports
- [ ] Transaction Service: Export functionality
- [ ] Referral Service: Code generation
- [ ] Referral Service: Downline tracking
- [ ] Referral Service: Commission calculation
- [ ] Referral Service: Claim balance
- [ ] Referral Service: Promotor management
- [ ] Deploy to K8s

**Week 8: Analytics Service**
- [ ] Dashboard data aggregation
- [ ] Turnover calculations
- [ ] Revenue reports
- [ ] Player analytics
- [ ] CRM data preparation
- [ ] Deploy to K8s

---

### Phase 3: Frontend Development (Week 4-10, Parallel)

**Week 4-6: Player Frontend (Next.js)**
- [ ] Next.js 14 project setup (App Router)
- [ ] TailwindCSS configuration
- [ ] Component library (buttons, cards, forms)
- [ ] Layout & navigation
- [ ] Authentication pages (login, register, OTP)
- [ ] Homepage with banners & promotions
- [ ] Game listing by category
- [ ] Game search & filters
- [ ] Game launch modal/page
- [ ] Deposit page
- [ ] Withdraw page
- [ ] Transaction history
- [ ] Profile management
- [ ] Referral dashboard
- [ ] Responsive design (mobile-first)
- [ ] SEO optimization
- [ ] Deploy to K8s

**Week 7-10: Admin Panel (Next.js)**
- [ ] Admin layout & navigation
- [ ] Dashboard with statistics
- [ ] Transaction management (approve/reject)
- [ ] Member management
- [ ] User search & filters
- [ ] Ban/unban users
- [ ] Balance injection tool
- [ ] Banner management (upload, CRUD)
- [ ] Promotion management
- [ ] Settings pages
- [ ] Referral settings
- [ ] Promotor management
- [ ] Admin activity logs
- [ ] Reports & analytics
- [ ] Domain alias management
- [ ] IP whitelist management
- [ ] Deploy to K8s

---

### Phase 4: Integration & Testing (Week 11-12)

**Week 11: Integration Testing**
- [ ] End-to-end testing (Playwright/Cypress)
- [ ] Service-to-service integration tests
- [ ] Payment gateway integration tests
- [ ] DominicEngine API integration tests
- [ ] Load testing (k6 or Artillery)
- [ ] Security testing (OWASP top 10)
- [ ] Performance optimization
- [ ] Bug fixes

**Week 12: User Acceptance Testing**
- [ ] Internal testing with team
- [ ] Fix critical bugs
- [ ] Documentation updates
- [ ] Deployment runbooks
- [ ] Backup & recovery procedures

---

### Phase 5: Data Migration (Week 13-14)

**Week 13: Data Migration**
- [ ] Export data from existing Laravel database
- [ ] Data transformation scripts
- [ ] Import users to new PostgreSQL
- [ ] Import transactions
- [ ] Import referral data
- [ ] Import banners & promotions
- [ ] Verify data integrity
- [ ] Test with migrated data

**Week 14: Parallel Run & Cutover**
- [ ] Deploy new system to production K8s
- [ ] Run old & new systems in parallel
- [ ] Route 10% traffic to new system
- [ ] Monitor errors & performance
- [ ] Increase to 50% traffic
- [ ] Monitor & fix issues
- [ ] Route 100% traffic to new system
- [ ] Keep old system as backup for 1 week
- [ ] Decommission old system

---

### Phase 6: Post-Launch (Week 15-16)

- [ ] Monitor system 24/7
- [ ] Fix production bugs
- [ ] Performance optimization
- [ ] User feedback collection
- [ ] Documentation finalization
- [ ] Team training
- [ ] Celebrate! 🎉

---

## 💰 Cost Estimation

### Infrastructure Costs

```yaml
VPS (Existing):
  Cost: $0 (already owned)
  Specs: 32GB RAM, 8 CPU

Domain Name:
  Cost: ~$10/year (~$1/month)
  Provider: Namecheap, GoDaddy, dll

CloudFlare (CDN + DNS):
  Plan: Free tier
  Cost: $0/month
  (Upgrade to Pro $20/month jika perlu)

S3-Compatible Storage:
  Option 1: Wasabi ($5.99/month for 1TB)
  Option 2: Backblaze B2 ($5/month for 1TB)
  Option 3: DigitalOcean Spaces ($5/month for 250GB)

SSL Certificate:
  Cost: $0 (Let's Encrypt via cert-manager)

Monitoring (Self-hosted):
  Cost: $0 (Prometheus + Grafana on K8s)

Error Tracking:
  Sentry: Free tier (5K errors/month)
  Or Self-hosted Sentry: $0

Email Service (for notifications):
  Option 1: SendGrid (Free: 100 emails/day)
  Option 2: AWS SES (~$0.10 per 1000 emails)
  Cost: $0-5/month

Total Monthly Cost: ~$10-15/month

```

### Development Costs

```
Assuming freelance/agency development:

Senior Full-Stack Developer: $30-50/hour
  Timeline: 3-4 months
  Hours: ~500-600 hours
  Cost: $15,000 - $30,000

Or DIY with team:
  3 developers × 3 months
  Cost: Salaries only
```

### Software Licenses

```
ALL FREE & OPEN SOURCE:
✅ k3s: FREE
✅ Rancher: FREE
✅ Traefik: FREE
✅ cert-manager: FREE
✅ ArgoCD: FREE
✅ PostgreSQL: FREE
✅ Redis: FREE
✅ Prometheus + Grafana: FREE
✅ Loki: FREE
✅ Next.js: FREE
✅ NestJS: FREE
✅ All npm packages: FREE (MIT/Apache licenses)

Total: $0/month 🎉
```

---

## 🔒 Security Considerations

### Application Security

1. **Authentication & Authorization**
   - JWT tokens with short expiration (15m access, 7d refresh)
   - Secure password hashing (bcrypt with cost 12)
   - OTP with rate limiting
   - IP whitelist for admin panel
   - Multi-factor authentication (optional)

2. **API Security**
   - Rate limiting (per IP, per user)
   - Request validation (class-validator)
   - SQL injection prevention (ORM)
   - XSS prevention (sanitization)
   - CSRF protection
   - CORS configuration

3. **Data Security**
   - Encryption at rest (database encryption)
   - Encryption in transit (SSL/TLS)
   - Sensitive data encrypted (payment info, passwords)
   - PII data handling compliance

4. **Network Security**
   - Firewall rules (only necessary ports)
   - DDoS protection (CloudFlare)
   - Intrusion detection (fail2ban)
   - Security groups on VPS

### Infrastructure Security

1. **Kubernetes Security**
   - RBAC (Role-Based Access Control)
   - Network policies (restrict pod-to-pod)
   - Pod security policies
   - Secret management (encrypted)
   - Image scanning (Trivy)

2. **Monitoring & Alerting**
   - Failed login attempts
   - Large withdrawals
   - Balance discrepancies
   - API errors spike
   - Resource usage alerts

3. **Backup & Disaster Recovery**
   - Daily PostgreSQL backups
   - Backup retention: 30 days
   - Point-in-time recovery capability
   - Offsite backup storage
   - Disaster recovery plan documented

### Compliance

- **Data Privacy**: GDPR-ready (if applicable)
- **Audit Logs**: All admin actions logged
- **Transaction Logs**: Immutable transaction records
- **Right to be Forgotten**: User data deletion capability

---

## 📊 Success Metrics

### Technical Metrics

```yaml
Performance:
  - API response time: < 200ms (p95)
  - Game launch time: < 2 seconds
  - Page load time: < 1.5 seconds
  - Uptime: 99.9% (max 43 minutes downtime/month)

Scalability:
  - Support 1,000 concurrent users (initial)
  - Support 10,000 concurrent users (6 months)
  - Horizontal scaling ready

Reliability:
  - Zero data loss
  - Transaction success rate: > 99.5%
  - Payment gateway uptime dependency
```

### Business Metrics

```yaml
User Experience:
  - Registration completion rate: > 80%
  - Deposit success rate: > 95%
  - Withdraw processing time: < 24 hours
  - Customer support tickets: < 5% of users

Operational:
  - Deployment frequency: Daily (via GitOps)
  - Mean time to recovery: < 1 hour
  - Change failure rate: < 5%
```

---

## 🎓 Team Training & Handover

### Documentation Deliverables

1. **Technical Documentation**
   - Architecture diagrams
   - API documentation (OpenAPI/Swagger)
   - Database schema & ERD
   - Service interaction diagrams
   - Deployment procedures

2. **Operational Runbooks**
   - How to deploy services
   - How to rollback deployments
   - How to scale services
   - Troubleshooting guides
   - Monitoring & alerting guides

3. **Developer Guides**
   - Local development setup
   - Code contribution guidelines
   - Git workflow
   - Testing guidelines
   - Code review process

### Training Sessions

```
Week 1: Infrastructure Overview
  - Kubernetes basics
  - Rancher UI walkthrough
  - GitOps with ArgoCD
  - Monitoring with Grafana

Week 2: Application Architecture
  - Microservices overview
  - Service communication
  - Database access patterns
  - DominicEngine API integration

Week 3: Deployment & Operations
  - CI/CD pipeline
  - How to deploy changes
  - How to monitor services
  - Incident response procedures

Week 4: Hands-on Practice
  - Deploy a new service
  - Make a code change & deploy
  - Investigate a simulated incident
  - Q&A session
```

---

## 🚨 Risk Mitigation

### Technical Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| DominicEngine API downtime | High | Circuit breaker, cached game list, fallback UI |
| Database corruption | Critical | Daily backups, point-in-time recovery, replication |
| Payment gateway failures | High | Multiple gateway options, manual fallback |
| K8s cluster failure | Critical | Regular backups, documented recovery procedure |
| Security breach | Critical | Security best practices, regular audits, monitoring |

### Business Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| User resistance to new UI | Medium | Familiar design, user training, feedback loop |
| Data migration issues | High | Extensive testing, parallel run, rollback plan |
| Extended downtime during cutover | High | Gradual traffic shift, monitoring, quick rollback |
| Budget overrun | Medium | Phased development, MVP first, iterate later |

---

## 📞 Support & Maintenance

### Post-Launch Support Plan

**Month 1-3: Critical Support**
- 24/7 monitoring
- Immediate bug fixes
- Performance optimization
- User feedback incorporation

**Month 4-6: Active Maintenance**
- Regular updates
- Feature enhancements
- Security patches
- Performance tuning

**Month 7+: Ongoing Maintenance**
- Monthly updates
- Quarterly security reviews
- Feature roadmap execution
- Continuous improvement

---

## 🎯 Next Steps

### Immediate Actions (This Week)

1. **[ ] Review & approve this plan**
   - Discuss with stakeholders
   - Adjust timeline if needed
   - Allocate budget

2. **[ ] Setup development environment**
   - Create GitHub organization/repo
   - Setup project structure
   - Configure Turborepo monorepo

3. **[ ] Infrastructure preparation**
   - Backup existing VPS data
   - Purchase domain name (if not yet)
   - Setup CloudFlare account

4. **[ ] Team assembly**
   - Hire developers (if needed)
   - Assign roles & responsibilities
   - Setup communication channels (Slack, etc)

### Week 1 Kickoff

1. **[ ] Install k3s + Rancher** (follow guide above)
2. **[ ] Create project repository**
3. **[ ] Setup CI/CD pipeline**
4. **[ ] Begin Auth Service development**
5. **[ ] Daily standup meetings**

---

## 📚 Additional Resources

### Learning Resources

**Kubernetes:**
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Rancher Documentation](https://ranchermanager.docs.rancher.com/)
- [k3s Documentation](https://docs.k3s.io/)

**NestJS:**
- [NestJS Documentation](https://docs.nestjs.com/)
- [NestJS Microservices](https://docs.nestjs.com/microservices/basics)

**Next.js:**
- [Next.js 14 Documentation](https://nextjs.org/docs)
- [Next.js App Router](https://nextjs.org/docs/app)

**GitOps:**
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [GitOps Guide](https://www.gitops.tech/)

**Monitoring:**
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)

### Community & Support

- Stack Overflow
- NestJS Discord
- Next.js Discord
- Kubernetes Slack
- Rancher Forums

---

## ✅ Checklist Summary

### Infrastructure Setup
- [ ] Install k3s
- [ ] Install Rancher
- [ ] Install Traefik
- [ ] Install cert-manager
- [ ] Deploy PostgreSQL
- [ ] Deploy Redis
- [ ] Install ArgoCD
- [ ] Install monitoring stack
- [ ] Configure backups

### Development
- [ ] Setup monorepo
- [ ] Auth Service
- [ ] User Service
- [ ] Game Proxy Service
- [ ] Payment Service
- [ ] Transaction Service
- [ ] Referral Service
- [ ] Analytics Service
- [ ] Player Frontend
- [ ] Admin Panel

### Testing & Deployment
- [ ] Unit tests
- [ ] Integration tests
- [ ] E2E tests
- [ ] Load tests
- [ ] Security tests
- [ ] CI/CD pipeline
- [ ] Production deployment

### Migration
- [ ] Data export
- [ ] Data transformation
- [ ] Data import
- [ ] Parallel run
- [ ] Traffic cutover
- [ ] Old system decommission

---

## 🎉 Conclusion

This comprehensive plan provides a roadmap to rebuild your gaming platform from a Laravel monolith to a modern, scalable microservices architecture. The phased approach allows for incremental development, testing, and deployment while minimizing risk.

**Key Success Factors:**
1. ✅ **Simplified game integration** via DominicEngineAPI
2. ✅ **Existing VPS** (32GB RAM) perfect for Kubernetes
3. ✅ **Free & open-source** tools (zero licensing costs)
4. ✅ **Phased migration** (reduced risk)
5. ✅ **GitOps workflow** (modern deployment)
6. ✅ **Comprehensive monitoring** (observability)

**Timeline**: 12-16 weeks to production  
**Budget**: ~$15K-30K development + ~$10-30/month operational  
**ROI**: Improved scalability, reliability, and developer productivity

Ready to start? Let's build something great! 🚀

---

**Document Version**: 1.0  
**Last Updated**: 2025-11-27  
**Author**: AI Assistant  
**Status**: Ready for Review
