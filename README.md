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
   - **Automated Deposit (PGA - Payment Gateway Automated):**
     - QRIS with callback: GsPay & VIP Otomatis
     - Auto-confirm via webhook callback
   - **Manual Deposit (Admin Approval Required):**
     - QRIS Manual
     - Bank Transfer
     - E-Wallet
     - Pulsa
   - **Automated Withdrawal:**
     - VIP Otomatis
     - GsPay
   
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
  
  -- User Level & Permissions
  user_level INT DEFAULT 0, -- 0=member, 1=superadmin, 2=finance, 3=CS, 4=CRM/promotor, 5=affiliate_member
  affiliate_approved BOOLEAN DEFAULT FALSE, -- For level 5 (member with affiliate access)
  
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
-- key: 'payment.pulsa_rate', value: 0.8  -- Pulsa conversion rate (0.1 - 1.0)
-- key: 'payment.deposit_methods', value: {
--   "qris_auto": { "enabled": true, "gateway": "gspay" },
--   "qris_manual": { "enabled": true },
--   "bank_transfer": { "enabled": true },
--   "ewallet": { "enabled": false },
--   "pulsa": { "enabled": true }
-- }
-- key: 'payment.auto_withdrawal_provider', value: "vipotomatis"  -- 'vipotomatis' or 'gspay'
-- key: 'game.dominic_engine', value: { "agent_code": "...", "agent_token": "...", "url": "..." }
-- key: 'website.name', value: "Gaming Platform"
-- key: 'localization.default_language', value: "id"  -- Indonesian (id)

-- ==================== PAYMENT ACCOUNTS (Manual Deposit) ====================
CREATE TYPE payment_account_type AS ENUM ('bank', 'qris', 'ewallet', 'pulsa');

CREATE TABLE payment_accounts (
  id SERIAL PRIMARY KEY,
  type payment_account_type NOT NULL,
  
  -- Common fields
  name VARCHAR(100) NOT NULL, -- Display name: "BCA - ABCD", "OVO - Payment", dll
  active BOOLEAN DEFAULT TRUE, -- On/Off toggle
  sort_order INT DEFAULT 0, -- Display order
  
  -- For Bank
  bank_name VARCHAR(50), -- BCA, Mandiri, BRI, BNI, dll
  bank_code VARCHAR(10), -- Bank code
  account_number VARCHAR(50),
  account_holder VARCHAR(100),
  
  -- For QRIS Manual
  qris_image_url TEXT, -- Static QRIS image
  qris_code TEXT, -- QRIS string
  
  -- For E-Wallet
  ewallet_provider VARCHAR(50), -- OVO, DANA, GoPay, LinkAja, dll
  ewallet_number VARCHAR(50),
  ewallet_name VARCHAR(100), -- Account holder name
  
  -- For Pulsa
  pulsa_provider VARCHAR(50), -- Telkomsel, XL, Indosat, Tri, dll
  pulsa_number VARCHAR(20),
  pulsa_name VARCHAR(100), -- Account holder name
  
  -- Metadata
  notes TEXT, -- Admin notes
  created_by BIGINT REFERENCES admins(id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_payment_accounts_type ON payment_accounts(type);
CREATE INDEX idx_payment_accounts_active ON payment_accounts(active);

-- Example data:
-- INSERT INTO payment_accounts (type, name, bank_name, account_number, account_holder, active) 
-- VALUES ('bank', 'BCA - ABCD', 'BCA', '0123456', 'ABCD', true);
-- 
-- INSERT INTO payment_accounts (type, name, bank_name, account_number, account_holder, active) 
-- VALUES ('bank', 'Mandiri - ABCD', 'Mandiri', '9876543', 'ABCD', false);
--
-- INSERT INTO payment_accounts (type, name, ewallet_provider, ewallet_number, ewallet_name, active)
-- VALUES ('ewallet', 'OVO - Payment', 'OVO', '081234567890', 'Payment Account', true);


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

## � User Levels & Permissions (RBAC)

### User Level System

The platform supports **6 user levels** with distinct permissions:

| Level | Role | Panel Access | Description |
|-------|------|--------------|-------------|
| **0** | Member | Member Panel | Regular member, NO affiliate/referral access |
| **1** | Superadmin | Admin Panel | Full access to all features |
| **2** | Finance | Admin Panel | Limited to transactions, deposits, withdrawals, player data, CRM (can edit) |
| **3** | CS (Customer Service) | Admin Panel | Similar to Finance but CRM is read-only |
| **4** | CRM / Promotor | Admin Panel | Own CRM data only (NDP, REGIST, reports, downline, commission) - read-only |
| **5** | Affiliate Member | Member Panel | Member + affiliate/referral features (NDP, REGIST, reports, downline, commission) |

---

### Level 0: Member (Regular)

**Panel**: Member Panel  
**Permissions**:
- ✅ Deposit
- ✅ Withdraw  
- ✅ Play games
- ✅ View transaction history
- ✅ Update profile
- ✅ Lucky draw
- ❌ **NO** Affiliate/Referral features
- ❌ **NO** Downline data
- ❌ **NO** Commission tracking

---

### Level 1: Superadmin

**Panel**: Admin Panel  
**Permissions**: **FULL ACCESS** to all features

- ✅ All transactions (deposit, withdrawal, approvals)
- ✅ All user management (create, edit, delete, ban/unban)
- ✅ All CRM features (view, edit, bonus settings)
- ✅ Payment settings (payment accounts, methods, rates)
- ✅ Game settings
- ✅ Analytics & reports (all data)
- ✅ Domain management
- ✅ Admin management (create admins, set levels)
- ✅ Settings (all categories)

---

### Level 2: Finance

**Panel**: Admin Panel  
**Access**: Limited to financial operations

**Can Access:**
- ✅ Transactions (all)
- ✅ Deposits (view, approve, reject)
- ✅ Withdrawals (view, approve, reject, choose disbursement method)
- ✅ Transaction history
- ✅ Player data (view user list, balances, bank info)
- ✅ CRM data (view & **EDIT** - can set/edit CRM bonus)

**Cannot Access:**
- ❌ Game settings
- ❌ Domain management
- ❌ Admin management
- ❌ Payment account management
- ❌ Global settings

---

### Level 3: CS (Customer Service)

**Panel**: Admin Panel  
**Access**: Similar to Finance but more restricted

**Can Access:**
- ✅ Transactions (view only, cannot approve/reject)
- ✅ Transaction history
- ✅ Player data (view user list, balances, bank info)
- ✅ CRM data (**READ-ONLY** - cannot edit)

**Cannot Access:**
- ❌ Approve/reject deposits
- ❌ Approve/reject withdrawals
- ❌ Edit CRM bonus
- ❌ Game settings
- ❌ Domain management
- ❌ Admin management
- ❌ Payment settings

---

### Level 4: CRM / Promotor

**Panel**: Admin Panel (Limited)  
**Access**: Own CRM data only

**Can Access (Own Data Only):**
- ✅ Own CRM dashboard
  - NDP (New Deposit Players) count
  - REGIST (New Registrations) count
  - Daily/Weekly/Monthly reports
- ✅ Own downline list (users referred by them)
- ✅ Own commission data (earned commissions)
- ✅ **READ-ONLY** - cannot edit anything

**Cannot Access:**
- ❌ Other promotors' data
- ❌ All player data (except own downline)
- ❌ Transactions
- ❌ Deposits/Withdrawals
- ❌ Any edit/create/delete operations

**Report Access:**
- Daily Report: Registrations, deposits, NDPs for today
- Weekly Report: Same for last 7 days
- Monthly Report: Same for last 30 days

---

### Level 5: Affiliate Member

**Panel**: Member Panel (Enhanced)  
**Access**: Member features + Affiliate/Referral system

**Can Access:**
- ✅ All Level 0 features (deposit, withdraw, play, etc.)
- ✅ **Affiliate Dashboard:**
  - NDP count
  - REGIST count
  - Daily/Weekly/Monthly reports
- ✅ Downline list (users who used their referral code)
- ✅ Commission tracking (earned commissions)
- ✅ Referral code (unique code to share)
- ✅ Claim commission (transfer referral_balance to main_balance)

**Approval Required:**
- User must have `affiliate_approved = true` to access affiliate features
- Superadmin can approve/reject affiliate requests

---

### Permission Matrix

| Feature | Level 0 | Level 1 | Level 2 | Level 3 | Level 4 | Level 5 |
|---------|---------|---------|---------|---------|---------|---------|
| **Member Panel** |
| Deposit | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Withdraw | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Play Games | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Lucky Draw | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Affiliate/Referral | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Admin Panel** |
| Transactions View | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ |
| Approve Deposit | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Approve Withdrawal | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Player Data | ❌ | ✅ | ✅ | ✅ | Own only | ❌ |
| CRM Data View | ❌ | ✅ | ✅ | ✅ | Own only | ❌ |
| CRM Data Edit | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Payment Settings | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Admin Management | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Domain Management | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |

---

### Implementation (RBAC Middleware)

**Backend (NestJS Guards):**

```typescript
// auth/guards/level.guard.ts
@Injectable()
export class LevelGuard implements CanActivate {
  constructor(private reflector: Reflector) {}
  
  canActivate(context: ExecutionContext): boolean {
    const requiredLevels = this.reflector.get<number[]>('levels', context.getHandler());
    if (!requiredLevels) return true;
    
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    
    return requiredLevels.includes(user.user_level);
  }
}

// Usage in controllers
@Controller('admin/transactions')
@UseGuards(LevelGuard)
export class TransactionsController {
  
  @Get()
  @Levels(1, 2, 3) // Superadmin, Finance, CS can view
  getTransactions() { }
  
  @Put(':id/approve')
  @Levels(1, 2) // Only Superadmin and Finance can approve
  approveTransaction() { }
}

// Custom decorator
export const Levels = (...levels: number[]) => SetMetadata('levels', levels);
```

**Frontend (Next.js Route Protection):**

```typescript
// middleware.ts
export function middleware(request: NextRequest) {
  const user = getAuthUser(request);
  
  // Member panel routes
  if (request.nextUrl.pathname.startsWith('/member')) {
    if (![0, 5].includes(user.user_level)) {
      return NextResponse.redirect('/admin');
    }
    
    // Affiliate routes only for level 5
    if (request.nextUrl.pathname.startsWith('/member/affiliate')) {
      if (user.user_level !== 5 || !user.affiliate_approved) {
        return NextResponse.redirect('/member');
      }
    }
  }
  
  // Admin panel routes
  if (request.nextUrl.pathname.startsWith('/admin')) {
    if (![1, 2, 3, 4].includes(user.user_level)) {
      return NextResponse.redirect('/member');
    }
    
    // Check specific admin routes
    if (request.nextUrl.pathname.startsWith('/admin/settings')) {
      if (user.user_level !== 1) { // Only Superadmin
        return NextResponse.redirect('/admin');
      }
    }
  }
}
```

---

## �🔧 Microservices Detail


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

**Payment Gateway Types:**

**Automated Deposit (PGA with Callback):**
- GsPay (QRIS automated with webhook callback)
- VIP Otomatis (QRIS automated with webhook callback)
- Auto-confirmation via callback
- Instant balance top-up

**Manual Deposit (Admin Approval):**
- QRIS Manual (admin verify payment proof)
- Bank Transfer (admin verify payment proof)
- E-Wallet (admin verify payment proof)
- Pulsa (admin verify payment proof) **with rate conversion**
  - Admin can set pulsa rate (0.1 - 1.0)
  - Example: Rate 0.8 → 100k pulsa = 80k balance
  - Example: Rate 1.0 → 100k pulsa = 100k balance (full)

**Automated Withdrawal:**
- VIP Otomatis
- GsPay

**API Endpoints:**
```typescript
// Automated Deposits (PGA)
POST   /payment/deposit/automated        // Initiate automated deposit (QRIS)
POST   /payment/callback/vipotomatis     // VIP Otomatis webhook
POST   /payment/callback/gspay           // GsPay webhook

// Manual Deposits
POST   /payment/deposit/manual           // Initiate manual deposit
POST   /payment/deposit/proof/upload     // Upload payment proof
GET    /payment/deposit/info/:method     // Get payment details (bank account, QRIS, etc)

// Withdrawals
POST   /payment/withdraw/auto            // Initiate auto withdrawal (PGA)
POST   /payment/withdraw/manual          // Initiate manual withdrawal (admin review)
POST   /payment/withdraw/callback        // PGA withdrawal callback

// Status & Management
GET    /payment/status/:transactionId    // Check transaction status

// Admin Actions
PUT    /payment/deposit/:id/approve      // Admin approve manual deposit
PUT    /payment/deposit/:id/reject       // Admin reject manual deposit
PUT    /payment/withdraw/:id/approve     // Admin approve withdrawal (+ choose method)
                                          // body: { method: 'pga_auto' | 'manual_transfer', gateway?: 'vipotomatis' | 'gspay' }
PUT    /payment/withdraw/:id/reject      // Admin reject withdrawal
PUT    /payment/withdraw/:id/complete    // Admin mark manual transfer as completed
GET    /payment/pending                  // Get pending transactions (admin)

// Admin Settings
GET    /payment/settings/pulsa-rate      // Get current pulsa rate
PUT    /payment/settings/pulsa-rate      // Update pulsa rate (body: { rate: 0.8 })
GET    /payment/settings/deposit-methods // Get all deposit methods status
PUT    /payment/settings/deposit-methods // Toggle deposit methods
                                          // body: { qris_auto: true, qris_manual: false, bank_transfer: true, ewallet: false, pulsa: true }
GET    /payment/settings/auto-withdrawal-provider  // Get current auto WD provider
PUT    /payment/settings/auto-withdrawal-provider  // Set auto WD provider
                                                     // body: { provider: 'vipotomatis' | 'gspay' }

// Payment Accounts Management (Bank, QRIS, E-Wallet, Pulsa)
GET    /payment/accounts                 // Get all payment accounts
GET    /payment/accounts/active          // Get active payment accounts only
GET    /payment/accounts/:id             // Get specific payment account
POST   /payment/accounts                 // Create new payment account
PUT    /payment/accounts/:id             // Update payment account
PUT    /payment/accounts/:id/toggle      // Toggle active/inactive
DELETE /payment/accounts/:id             // Delete payment account
```

**Example Payment Account API Usage:**

```typescript
// Create Bank Account
POST /payment/accounts
{
  "type": "bank",
  "name": "BCA - ABCD",
  "active": true,
  "bank_name": "BCA",
  "bank_code": "014",
  "account_number": "0123456",
  "account_holder": "ABCD"
}

// Create E-Wallet Account
POST /payment/accounts
{
  "type": "ewallet",
  "name": "OVO - Payment",
  "active": true,
  "ewallet_provider": "OVO",
  "ewallet_number": "081234567890",
  "ewallet_name": "Payment Account"
}

// Toggle Account On/Off
PUT /payment/accounts/123/toggle
// Response: { id: 123, active: false }
```

**Critical Flows:**

**Deposit Flow Type 1: Automated (PGA with Callback)**
```
Use Case: QRIS Automated via GsPay or VIP Otomatis

1. User select deposit method: "QRIS Automated"
2. User input amount
3. Create transaction (status: pending, type: deposit_automated)
4. Call payment gateway API (GsPay or VIP Otomatis)
5. Gateway returns QRIS code/image
6. Display QRIS to user (scan & pay)
7. User pays via mobile banking
   ↓
8. Payment gateway sends callback/webhook to our API
9. Validate callback:
   - Signature verification
   - Idempotency check (prevent double processing)
   - Amount verification
10. Update transaction (status: success)
11. Call Game Proxy → depositMember() (sync to game)
12. Update user balance in DB
13. Invalidate balance cache
14. Send notification to user
15. Auto-redirect user to success page

⏱️ Processing Time: INSTANT (1-30 seconds after payment)
✅ No admin action required
```

**Deposit Flow Type 2: Manual (Admin Approval Required)**
```
Use Case: QRIS Manual, Bank Transfer, E-Wallet, Pulsa

1. User select deposit method: 
- "QRIS Manual" 
   - "Bank Transfer (BCA, Mandiri, BRI, dll)"
   - "E-Wallet (OVO, Dana, GoPay, dll)"
   - "Pulsa (Telkomsel, XL, Indosat, dll)"
2. User input amount
3. System shows payment details:
   - For Bank: Bank name, Account number, Account holder
   - For QRIS Manual: Static QRIS image/code
   - For E-Wallet: E-wallet number
   - For Pulsa: Pulsa number + **Rate info**
     
     **Pulsa Rate Calculation Example:**
     ```
     User inputs: 100,000 (pulsa nominal)
     Current rate: 0.8 (from settings)
     
     Display to user:
     ┌─────────────────────────────────┐
     │ Pulsa Deposit                   │
     │                                 │
     │ Pulsa Amount:    Rp 100,000     │
     │ Conversion Rate: 0.8 (80%)      │
     │ You will receive: Rp 80,000     │
     │                                 │
     │ Send to: 081234567890           │
     └─────────────────────────────────┘
     
     Calculation: 100,000 × 0.8 = 80,000
     ```
4. User makes payment externally
5. User upload payment proof (screenshot/photo)
6. Create transaction (status: pending, type: deposit_manual)
7. Save payment proof to storage (S3)
   ↓
8. Admin receives notification (new pending deposit)
9. Admin reviews:
   - Check payment proof
   - Verify amount
   - Verify sender info
10. Admin action:
    APPROVE:
      → Update transaction (status: success)
      → Call Game Proxy → depositMember()
      → Update user balance
      → Invalidate cache
      → Send notification to user
    
    REJECT:
      → Update transaction (status: failed)
      → Add rejection reason
      → Send notification to user

⏱️ Processing Time: 5 minutes - 24 hours (depends on admin availability)
⚠️ Requires admin review & approval
```

**Database Schema for Deposits:**
```sql
-- Enhanced transactions table
CREATE TABLE transactions (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT REFERENCES users(id),
  type transaction_type, -- deposit_automated, deposit_manual, withdraw
  
  -- Amounts
  amount DECIMAL(15,2) NOT NULL,
  
  -- Status
  status transaction_status DEFAULT 'pending',
  
  -- Payment Details
  payment_method VARCHAR(50), -- qris_auto, qris_manual, bank_transfer, ewallet, pulsa
  payment_gateway VARCHAR(50), -- gspay, vipotomatis, manual
  
  -- For Automated Deposits
  gateway_ref_id VARCHAR(100), -- Reference from payment gateway
  gateway_response JSONB, -- Full callback response
  qris_code TEXT, -- QRIS string/URL
  
  -- For Manual Deposits
  payment_proof_url TEXT, -- S3 URL of uploaded proof
  bank_account_name VARCHAR(100), -- User's bank account name (for verification)
  bank_account_number VARCHAR(50), -- User's bank account number
  payment_timestamp TIMESTAMP, -- When user claims they paid
  
  -- For Pulsa Deposits (with rate conversion)
  pulsa_amount DECIMAL(15,2), -- Original pulsa amount
  pulsa_rate DECIMAL(3,2), -- Rate applied (0.1 - 1.0)
  final_amount DECIMAL(15,2), -- Calculated amount after rate (pulsa_amount * pulsa_rate)
  
  -- Approval
  admin_id BIGINT REFERENCES admins(id),
  approved_at TIMESTAMP,
  rejection_reason TEXT,
  
  -- Metadata
  notes TEXT,
  metadata JSONB,
  
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_transactions_user_id ON transactions(user_id);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_transactions_type ON transactions(type);
CREATE INDEX idx_transactions_payment_method ON transactions(payment_method);
```


**Withdrawal Flow Type 1: Auto Withdrawal (PGA)**
```
Use Case: Automated withdrawal via VIP Otomatis or GsPay

1. User request withdraw (amount)
2. Check user balance (cache + DB)
3. Validate minimum withdrawal amount
4. Call Game Proxy → withdrawMember() (pull from game)
5. If success: Create transaction (status: pending, type: withdraw_auto)
6. Call PGA API (VIP Otomatis or GsPay) for disbursement
7. PGA processes payment automatically
8. Receive callback/webhook from PGA
9. Validate callback
10. Update transaction (status: success)
11. Update user balance in DB
12. Invalidate cache
13. Send notification to user

⏱️ Processing Time: INSTANT (1-5 minutes)
✅ No admin action required
💰 May have PGA fees
```

**Withdrawal Flow Type 2: Manual Withdrawal (Admin Controlled)**
```
Use Case: Admin review and choose disbursement method

1. User request withdraw (amount)
2. Check user balance (cache + DB)
3. Validate minimum withdrawal amount
4. Call Game Proxy → withdrawMember() (pull from game)
5. If success: Create transaction (status: pending, type: withdraw_manual)
6. Save user bank details for withdrawal
   ↓
7. Admin receives notification (new withdrawal request)
8. Admin reviews:
   - Check user balance history
   - Verify bank details
   - Check for suspicious activity
   
9. Admin decision:
   
   ┌─── APPROVE ───┐
   │               │
   │  Admin chooses disbursement method:
   │  
   │  Option A: Send via PGA (Automated)
   │    → Call VIP Otomatis or GsPay API
   │    → Automatic disbursement
   │    → Receive callback
   │    → Update transaction (status: success)
   │    → Send notification
   │    ⏱️ Processing: 1-5 minutes
   │    💰 PGA fees apply
   │  
   │  Option B: Send Manual (Bank Transfer)
   │    → Admin marks as "processing"
   │    → Admin manually transfers via bank
   │    → Admin uploads transfer proof (optional)
   │    → Admin marks as "completed"
   │    → Update transaction (status: success)
   │    → Send notification
   │    ⏱️ Processing: varies (same day - next day)
   │    💰 No PGA fees
   │
   └───────────────┘
   
   OR
   
   ┌─── REJECT ────┐
   │               │
   │  → Return balance to user (if already withdrawn from game)
   │  → Call Game Proxy → depositMember() (return to game)
   │  → Update transaction (status: rejected)
   │  → Add rejection reason
   │  → Send notification to user
   │
   └───────────────┘

⏱️ Processing Time: Varies (depends on admin + method)
⚠️ Requires admin review
💡 Admin can choose cheaper manual method or faster PGA
```

**Enhanced Database Schema for Withdrawals:**
```sql
-- Add withdrawal-specific fields to transactions table
ALTER TABLE transactions ADD COLUMN IF NOT EXISTS
  withdrawal_method VARCHAR(20), -- 'auto_pga', 'manual_pga', 'manual_transfer'
  disbursement_status VARCHAR(20), -- 'pending', 'processing', 'completed', 'failed'
  disbursement_proof_url TEXT, -- For manual transfers
  pga_reference_id VARCHAR(100), -- Reference from PGA
  pga_fees DECIMAL(15,2), -- Fees charged by PGA
  disbursed_at TIMESTAMP, -- When money was sent
  admin_notes TEXT; -- Admin notes for manual processing
```

**Admin Panel Withdrawal Approval UI:**
```typescript
// Example withdrawal approval interface
interface WithdrawalApprovalAction {
  transactionId: number;
  action: 'approve' | 'reject';
  
  // If approved
  disbursementMethod?: 'pga_auto' | 'manual_transfer';
  pga_gateway?: 'vipotomatis' | 'gspay'; // If using PGA
  
  // If rejected
  rejectionReason?: string;
  
  // Optional
  adminNotes?: string;
}
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

### 8. Togel Service (NEW)

**Responsibilities:**
- Togel game integration via DominicEngineAPI
- In-game panel page for user to play
- Betting management
- Results tracking
- Winnings calculation

**Integration:**
- DominicEngineAPI togel endpoints
- Real-time betting interface
- Live results display

**API Endpoints:**
```typescript
GET    /togel/games                     // List available togel games
GET    /togel/markets                   // Get current markets
POST   /togel/bet                       // Place bet
GET    /togel/results                   // Get latest results
GET    /togel/history/:userId           // User betting history
GET    /togel/winnings/:userId          // User winnings
```

**Frontend Pages:**
- `/togel` - Main togel game page (in-game panel)
- Betting interface with number selection
- Live draw results
- Betting history
- Winnings tracker

**Database Tables:**
```sql
CREATE TABLE togel_bets (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT REFERENCES users(id),
  game_id VARCHAR(50),
  market_code VARCHAR(20), -- 4D, 3D, 2D, dll
  numbers VARCHAR(50), -- Betting numbers
  bet_amount DECIMAL(15,2),
  win_amount DECIMAL(15,2) DEFAULT 0,
  status VARCHAR(20), -- pending, win, lose
  draw_date DATE,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_togel_bets_user ON togel_bets(user_id);
CREATE INDEX idx_togel_bets_status ON togel_bets(status);
```

---

### 9. Lotre Service (NEW)

**Responsibilities:**
- Lottery game integration via DominicEngineAPI
- In-game lottery panel page
- Ticket purchase management
- Draw results tracking
- Prize distribution

**Integration:**
- DominicEngineAPI lottery endpoints
- Automated draw system
- Prize payout integration

**API Endpoints:**
```typescript
GET    /lotre/draws                     // List upcoming/active draws
GET    /lotre/draw/:id                  // Get draw details
POST   /lotre/ticket/buy                // Purchase lottery ticket
GET    /lotre/tickets/:userId           // User's tickets
GET    /lotre/results/:drawId           // Draw results
GET    /lotre/winners/:drawId           // Draw winners
```

**Frontend Pages:**
- `/lotre` - Main lottery page (in-game panel)
- Ticket purchase interface
- Current draws display
- Results announcement
- Winning tickets tracking

**Database Tables:**
```sql
CREATE TABLE lotre_draws (
  id SERIAL PRIMARY KEY,
  draw_code VARCHAR(50) UNIQUE,
  draw_name VARCHAR(100),
  ticket_price DECIMAL(15,2),
  prize_pool DECIMAL(15,2),
  draw_date TIMESTAMP,
  status VARCHAR(20), -- open, closed, drawn
  winning_numbers JSONB, -- Array of winning numbers
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE lotre_tickets (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT REFERENCES users(id),
  draw_id INT REFERENCES lotre_draws(id),
  ticket_number VARCHAR(50) UNIQUE,
  numbers JSONB, -- User selected numbers
  purchase_amount DECIMAL(15,2),
  win_amount DECIMAL(15,2) DEFAULT 0,
  status VARCHAR(20), -- active, win, lose
  purchased_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_lotre_tickets_user ON lotre_tickets(user_id);
CREATE INDEX idx_lotre_tickets_draw ON lotre_tickets(draw_id);
CREATE INDEX idx_lotre_tickets_status ON lotre_tickets(status);
```

---

### 10. Luckydraw Service (NEW)

**Responsibilities:**
- Lucky draw/spin reward system for members
- In-panel reward page
- Daily/weekly spin mechanics
- Prize management
- Claim rewards

**Integration:**
- DominicEngineAPI luckydraw endpoints
- Spin eligibility checking (based on deposits, turnover, etc)
- Automated prize distribution

**API Endpoints:**
```typescript
GET    /luckydraw/eligibility           // Check if user can spin
GET    /luckydraw/prizes                // List available prizes
POST   /luckydraw/spin                  // Execute spin
POST   /luckydraw/claim/:prizeId        // Claim won prize
GET    /luckydraw/history/:userId       // User spin history
GET    /luckydraw/settings              // Admin: prize settings
PUT    /luckydraw/settings              // Admin: update prizes
```

**Frontend Pages:**
- `/luckydraw` - Lucky draw page (in-panel)
- Spin wheel interface
- Prize showcase
- Spin history
- Claim rewards interface

**Features:**
- Daily spin (free for eligible users)
- Spin based on deposit amount
- Spin based on turnover
- Multiple prize tiers
- Instant claim or accumulate

**Database Tables:**
```sql
CREATE TABLE luckydraw_prizes (
  id SERIAL PRIMARY KEY,
  prize_name VARCHAR(100),
  prize_type VARCHAR(20), -- bonus, free_spin, cashback, dll
  prize_value DECIMAL(15,2),
  probability DECIMAL(5,2), -- Winning probability (0-100)
  active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE luckydraw_spins (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT REFERENCES users(id),
  prize_id INT REFERENCES luckydraw_prizes(id),
  prize_claimed BOOLEAN DEFAULT FALSE,
  claimed_at TIMESTAMP,
  spin_type VARCHAR(20), -- daily_free, deposit_bonus, turnover_reward
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE luckydraw_eligibility (
  user_id BIGINT PRIMARY KEY REFERENCES users(id),
  daily_spin_available BOOLEAN DEFAULT TRUE,
  last_daily_spin TIMESTAMP,
  bonus_spins_available INT DEFAULT 0,
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_luckydraw_spins_user ON luckydraw_spins(user_id);
CREATE INDEX idx_luckydraw_spins_claimed ON luckydraw_spins(prize_claimed);
```

**Eligibility Rules:**
```typescript
// Example eligibility logic
class LuckydrawEligibilityService {
  async checkEligibility(userId: number) {
    const user = await this.getUserEligibility(userId);
    
    return {
      canSpin: user.daily_spin_available || user.bonus_spins_available > 0,
      dailyAvailable: user.daily_spin_available,
      bonusSpins: user.bonus_spins_available,
      nextDailySpin: this.getNextDailySpinTime(user.last_daily_spin)
    };
  }
  
  async grantBonusSpin(userId: number, reason: string) {
    // Grant bonus spin based on:
    // - Deposit amount (e.g., 1 spin per 100k deposit)
    // - Turnover milestone (e.g., 1 spin per 1M turnover)
    // - Referral rewards
    await this.incrementBonusSpins(userId, 1, reason);
  }
}
```

---

## 🎯 Updated Microservices Count

Total services: **10 microservices**

1. Auth Service
2. User Service
3. Game Proxy Service
4. Payment Service
5. Transaction Service
6. Referral Service
7. Analytics Service
8. **Togel Service** ← NEW
9. **Lotre Service** ← NEW
10. **Luckydraw Service** ← NEW

---

## 🎮 DominicEngineAPI Extended Integration

### Togel API Integration
```typescript
// game-proxy-service/src/dominic/togel.client.ts
export class TogelClient extends DominicEngineAPI {
  async getTogelMarkets() {
    return this.sendRequest('API/togel/markets/', {});
  }
  
  async placeBet(username: string, market: string, numbers: string, amount: number) {
    return this.sendRequest('API/togel/bet/', {
      user_code: username,
      market_code: market,
      numbers: numbers,
      amount: amount
    });
  }
  
  async getResults(date: string) {
    return this.sendRequest('API/togel/results/', {
      draw_date: date
    });
  }
}
```

### Lotre API Integration
```typescript
// game-proxy-service/src/dominic/lotre.client.ts
export class LotreClient extends DominicEngineAPI {
  async getActiveDraws() {
    return this.sendRequest('API/lotre/draws/', {
      status: 'open'
    });
  }
  
  async purchaseTicket(username: string, drawId: string, numbers: number[]) {
    return this.sendRequest('API/lotre/ticket/buy/', {
      user_code: username,
      draw_id: drawId,
      selected_numbers: numbers
    });
  }
  
  async getDrawResults(drawId: string) {
    return this.sendRequest('API/lotre/results/', {
      draw_id: drawId
    });
  }
}
```

### Luckydraw API Integration
```typescript
// game-proxy-service/src/dominic/luckydraw.client.ts
export class LuckydrawClient extends DominicEngineAPI {
  async getPrizePool() {
    return this.sendRequest('API/luckydraw/prizes/', {});
  }
  
  async executeSpin(username: string) {
    return this.sendRequest('API/luckydraw/spin/', {
      user_code: username
    });
  }
  
  async claimPrize(username: string, prizeId: string) {
    return this.sendRequest('API/luckydraw/claim/', {
      user_code: username,
      prize_id: prizeId
    });
  }
}
```

---

## 🌐 Domain Alias Management (Cloudflare Integration)

### Overview

Domain alias management menggunakan **Cloudflare API** untuk automated DNS configuration. Admin dapat menambahkan domain/subdomain melalui panel, dan system akan otomatis setup DNS records via Cloudflare API.

### Architecture Flow

```
Admin Panel → Domain Service → Cloudflare API → DNS Setup
                    ↓
              Check if domain/subdomain
                    ↓
        ├─ Main Domain ─→ Create Zone → Get Nameservers
        └─ Subdomain ───→ Check parent zone exists
                           ├─ Yes → Add A/CNAME record
                           └─ No  → Create parent zone first → Add subdomain record
```

### Database Schema Enhancement

```sql
-- Enhanced domain_aliases table
CREATE TABLE domain_aliases (
  id SERIAL PRIMARY KEY,
  domain VARCHAR(255) UNIQUE NOT NULL,
  is_subdomain BOOLEAN DEFAULT FALSE,
  parent_domain VARCHAR(255), -- Null for main domains
  
  status domain_status DEFAULT 'pending',
  
  -- Cloudflare Integration
  cloudflare_zone_id VARCHAR(100),
  cloudflare_nameservers JSONB, -- ["ns1.cloudflare.com", "ns2.cloudflare.com"]
  cloudflare_status VARCHAR(50), -- 'pending', 'active', 'failed'
  cloudflare_record_id VARCHAR(100), -- For subdomain A/CNAME record
  
  -- Target server
  target_ip INET, -- IP address to point to (your VPS)
  
  -- Nginx Config  
  nginx_config_path TEXT,
  nginx_status VARCHAR(20), -- 'pending', 'configured', 'failed'
  
  -- Metadata
  created_by BIGINT REFERENCES admins(id),
  notes TEXT,
  
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_domain_aliases_status ON domain_aliases(status);
CREATE INDEX idx_domain_aliases_parent ON domain_aliases(parent_domain);
CREATE INDEX idx_domain_aliases_cf_zone ON domain_aliases(cloudflare_zone_id);
```

### Workflow Detail

#### **Flow 1: Add Main Domain (example.com)**

```
1. Admin inputs: example.com
   ↓
2. System validates domain format
   ↓
3. Check if subdomain
   → Extract: is_subdomain = FALSE
   ↓
4. Call Cloudflare API: Create Zone
   POST https://api.cloudflare.com/client/v4/zones
   Body: {
     "name": "example.com",
     "type": "full"
   }
   ↓
5. Cloudflare Response:
   {
     "result": {
       "id": "zone_id_abc123",
       "name": "example.com",
       "status": "pending",
       "name_servers": [
         "ns1.cloudflare.com",
         "ns2.cloudflare.com"
       ]
     }
   }
   ↓
6. Save to database:
   - cloudflare_zone_id: "zone_id_abc123"
   - cloudflare_nameservers: ["ns1.cloudflare.com", "ns2.cloudflare.com"]
   - status: 'pending'
   ↓
7. Add DNS A Record (point domain to VPS)
   POST https://api.cloudflare.com/client/v4/zones/{zone_id}/dns_records
   Body: {
     "type": "A",
     "name": "@", // Root domain
     "content": "217.216.72.31", // Your VPS IP
     "ttl": 1, // Auto
     "proxied": true // Cloudflare proxy enabled
   }
   ↓
8. Display to Admin:
   ╔══════════════════════════════════════╗
   ║ Domain Added Successfully!           ║
   ║                                      ║
   ║ Domain: example.com                  ║
   ║ Status: Pending Nameserver Setup     ║
   ║                                      ║
   ║ Nameservers (paste to your domain   ║
   ║ registrar):                          ║
   ║ ├─ ns1.cloudflare.com                ║
   ║ └─ ns2.cloudflare.com                ║
   ║                                      ║
   ║ DNS propagation may take 24-48 hours ║
   ╚══════════════════════════════════════╝
   ↓
9. Admin copies nameservers → paste to domain registrar (GoDaddy, Namecheap, dll)
   ↓
10. Background job checks zone status every hour:
    GET https://api.cloudflare.com/client/v4/zones/{zone_id}
    
    If status = "active":
      → Update domain_aliases.status = 'active'
      → Generate Nginx config
      → Reload Nginx
```

#### **Flow 2: Add Subdomain (sub.example.com)**

```
1. Admin inputs: sub.example.com
   ↓
2. System validates domain format
   ↓
3. Check if subdomain
   → Extract parent domain: example.com
   → is_subdomain = TRUE
   ↓
4. Check if parent zone exists in Cloudflare
   Query DB: SELECT cloudflare_zone_id FROM domain_aliases 
             WHERE domain = 'example.com' AND status = 'active'
   
   ├─ Case A: Parent zone EXISTS
   │   ↓
   │   5a. Use existing zone_id
   │   ↓
   │   6a. Add DNS A Record for subdomain
   │       POST https://api.cloudflare.com/client/v4/zones/{zone_id}/dns_records
   │       Body: {
   │         "type": "A",
   │         "name": "sub", // Subdomain prefix only
   │         "content": "217.216.72.31",
   │         "ttl": 1,
   │         "proxied": true
   │       }
   │   ↓
   │   7a. Save to database:
   │       - domain: 'sub.example.com'
   │       - parent_domain: 'example.com'
   │       - is_subdomain: TRUE
   │       - cloudflare_zone_id: (parent's zone_id)
   │       - cloudflare_record_id: "record_id_xyz789"
   │       - status: 'active' (immediately active!)
   │   ↓
   │   8a. Generate Nginx config
   │   ↓
   │   9a. Reload Nginx
   │   ↓
   │   10a. Display success:
   │       ╔══════════════════════════════════════╗
   │       ║ Subdomain Added Successfully!        ║
   │       ║                                      ║
   │       ║ Subdomain:  sub.example.com          ║
   │       ║ Status:     Active ✅                 ║
   │       ║ IP:         217.216.72.31            ║
   │       ║                                      ║
   │       ║ DNS propagation: 1-5 minutes         ║
   │       ╚══════════════════════════════════════╝
   │
   └─ Case B: Parent zone DOES NOT EXIST
       ↓
       5b. Create parent zone FIRST
           (Follow Flow 1 steps 4-7)
       ↓
       6b. Then add subdomain record
           (Same as steps 6a-10a above)
       ↓
       7b. Display to admin:
           ╔══════════════════════════════════════╗
           ║ Parent Domain Setup Required         ║
           ║                                      ║
           ║ Created: example.com                 ║
           ║ Nameservers:                         ║
           ║ ├─ ns1.cloudflare.com                ║
           ║ └─ ns2.cloudflare.com                ║
           ║                                      ║
           ║ Subdomain: sub.example.com           ║
           ║ Status: Pending (waiting for NS      ║
           ║         propagation)                 ║
           ║                                      ║
           ║ Please update nameservers at your    ║
           ║ domain registrar first!              ║
           ╚══════════════════════════════════════╝
```

### API Service Implementation

**Domain Service (NestJS):**

```typescript
// domain-service/src/cloudflare/cloudflare.client.ts
import axios from 'axios';

export class CloudflareClient {
  private readonly apiToken: string;
  private readonly baseUrl = 'https://api.cloudflare.com/client/v4';
  
  constructor() {
    this.apiToken = process.env.CLOUDFL ARE_API_TOKEN;
  }
  
  // Create Zone (for main domain)
  async createZone(domain: string) {
    const response = await axios.post(
      `${this.baseUrl}/zones`,
      {
        name: domain,
        type: 'full',
        jump_start: true // Auto-scan DNS records
      },
      {
        headers: {
          'Authorization': `Bearer ${this.apiToken}`,
          'Content-Type': 'application/json'
        }
      }
    );
    
    return {
      zoneId: response.data.result.id,
      nameServers: response.data.result.name_servers,
      status: response.data.result.status
    };
  }
  
  // Add DNS A Record
  async addARecord(zoneId: string, name: string, ip: string, proxied = true) {
    const response = await axios.post(
      `${this.baseUrl}/zones/${zoneId}/dns_records`,
      {
        type: 'A',
        name: name, // '@' for root, 'sub' for subdomain
        content: ip,
        ttl: 1, // Auto
        proxied: proxied
      },
      {
        headers: {
          'Authorization': `Bearer ${this.apiToken}`,
          'Content-Type': 'application/json'
        }
      }
    );
    
    return {
      recordId: response.data.result.id,
      name: response.data.result.name,
      content: response.data.result.content
    };
  }
  
  // Check Zone Status
  async getZoneStatus(zoneId: string) {
    const response = await axios.get(
      `${this.baseUrl}/zones/${zoneId}`,
      {
        headers: {
          'Authorization': `Bearer ${this.apiToken}`
        }
      }
    );
    
    return response.data.result.status; // 'pending' or 'active'
  }
  
  // List DNS Records (untuk verify)
  async listDNSRecords(zoneId: string) {
    const response = await axios.get(
      `${this.baseUrl}/zones/${zoneId}/dns_records`,
      {
        headers: {
          'Authorization': `Bearer ${this.apiToken}`
        }
      }
    );
    
    return response.data.result;
  }
  
  // Delete DNS Record (untuk remove subdomain)
  async deleteDNSRecord(zoneId: string, recordId: string) {
    await axios.delete(
      `${this.baseUrl}/zones/${zoneId}/dns_records/${recordId}`,
      {
        headers: {
          'Authorization': `Bearer ${this.apiToken}`
        }
      }
    );
  }
}
```

**Domain Service Controller:**

```typescript
// domain-service/src/domain/domain.controller.ts
import { Controller, Post, Get, Delete, Body, Param } from '@nestjs/common';
import { DomainService } from './domain.service';

@Controller('domains')
export class DomainController {
  constructor(private readonly domainService: DomainService) {}
  
  @Post('add')
  async addDomain(@Body() dto: AddDomainDto) {
    // Extract domain info
    const domainInfo = this.parseDomain(dto.domain);
    
    if (domainInfo.isSubdomain) {
      // Subdomain logic
      return await this.domainService.addSubdomain(
        dto.domain,
        domainInfo.parentDomain,
        dto.targetIp
      );
    } else {
      // Main domain logic
      return await this.domainService.addMainDomain(
        dto.domain,
        dto.targetIp
      );
    }
  }
  
  @Get('status/:id')
  async checkStatus(@Param('id') id: number) {
    return await this.domainService.checkDomainStatus(id);
  }
  
  @Delete(':id')
  async removeDomain(@Param('id') id: number) {
    return await this.domainService.removeDomain(id);
  }
  
  private parseDomain(domain: string) {
    const parts = domain.split('.');
    
    if (parts.length > 2) {
      // Subdomain (sub.example.com)
      return {
        isSubdomain: true,
        subdomain: parts.slice(0, -2).join('.'),
        parentDomain: parts.slice(-2).join('.')
      };
    } else {
      // Main domain (example.com)
      return {
        isSubdomain: false,
        parentDomain: null
      };
    }
  }
}
```

### Nginx Configuration Generation

Setelah DNS setup, system auto-generate Nginx config:

```nginx
# /etc/nginx/sites-available/sub.example.com.conf
server {
    listen 80;
    listen [::]:80;
    server_name sub.example.com;
    
    # Cloudflare SSL/TLS (Flexible/Full)
    # Or use cert-manager for Let's Encrypt
    
    location / {
        proxy_pass http://localhost:3000; # Next.js frontend
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    location /api {
        proxy_pass http://localhost:4000; # API Gateway
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Background Jobs

**Cron job untuk check zone status:**

```typescript
// domain-service/src/jobs/domain-status-checker.job.ts
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class DomainStatusCheckerJob {
  constructor(
    private readonly cloudflareClient: CloudflareClient,
    private readonly domainRepository: DomainRepository
  ) {}
  
  @Cron(CronExpression.EVERY_HOUR) // Check every hour
  async checkPendingDomains() {
    const pendingDomains = await this.domainRepository.findPending();
    
    for (const domain of pendingDomains) {
      try {
        const status = await this.cloudflareClient.getZoneStatus(
          domain.cloudflareZoneId
        );
        
        if (status === 'active') {
          // Update domain status
          await this.domainRepository.updateStatus(domain.id, 'active');
          
          // Generate Nginx config
          await this.generateNginxConfig(domain);
          
          // Reload Nginx
          await this.reloadNginx();
          
          // Send notification to admin
          await this.notifyAdmin(domain, 'activated');
        }
      } catch (error) {
        console.error(`Error checking domain ${domain.domain}:`, error);
      }
    }
  }
  
  private async generateNginxConfig(domain: DomainAlias) {
    // Generate nginx config file
    const config = this.getNginxTemplate(domain);
    const configPath = `/etc/nginx/sites-available/${domain.domain}.conf`;
    
    await fs.writeFile(configPath, config);
    await fs.symlink(
      configPath,
      `/etc/nginx/sites-enabled/${domain.domain}.conf`
    );
    
    // Update database
    await this.domainRepository.update(domain.id, {
      nginxConfigPath: configPath,
      nginxStatus: 'configured'
    });
  }
  
  private async reloadNginx() {
    // Test config first
    await exec('nginx -t');
    
    // Reload
    await exec('systemctl reload nginx');
  }
}
```

### Admin Panel UI

**Add Domain Form:**

```typescript
// admin-panel/src/components/domains/AddDomainForm.tsx
export function AddDomainForm() {
  const [domain, setDomain] = useState('');
  const [result, setResult] = useState(null);
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    
    const response = await fetch('/api/domains/add', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        domain: domain,
        targetIp: '217.216.72.31' // Your VPS IP
      })
    });
    
    const data = await response.json();
    setResult(data);
  };
  
  return (
    <div>
      <form onSubmit={handleSubmit}>
        <input 
          type="text"
          placeholder="example.com or sub.example.com"
          value={domain}
          onChange={(e) => setDomain(e.target.value)}
        />
        <button type="submit">Add Domain</button>
      </form>
      
      {result && (
        <div className="result-card">
          <h3>Domain Added!</h3>
          <p>Domain: {result.domain}</p>
          <p>Status: {result.status}</p>
          
          {result.nameservers && (
            <div className="nameservers">
              <h4>Nameservers (paste to your registrar):</h4>
              <ul>
                {result.nameservers.map(ns => (
                  <li key={ns}>
                    {ns}
                    <button onClick={() => navigator.clipboard.writeText(ns)}>
                      Copy
                    </button>
                  </li>
                ))}
              </ul>
            </div>
          )}
          
          {result.isSubdomain && result.status === 'active' && (
            <div className="success">
              ✅ Subdomain is active! DNS propagation: 1-5 minutes
            </div>
          )}
        </div>
      )}
    </div>
  );
}
```

### Key Features

1. ✅ **Automated DNS Setup** - No manual DNS configuration
2. ✅ **Nameserver Display** - Auto-show CF nameservers to admin
3. ✅ **Subdomain Intelligence** - Auto-detect & handle subdomains
4. ✅ **Parent Zone Check** - Verify parent domain exists before adding subdomain
5. ✅ **Background Monitoring** - Auto-check zone activation status
6. ✅ **Nginx Auto-Config** - Generate & reload Nginx configs
7. ✅ **Cloudflare Proxy** - Enable DDoS protection & CDN
8. ✅ **Status Tracking** - Real-time domain status updates

### Settings Required

```sql
-- Cloudflare API credentials in settings table
INSERT INTO settings (key, value, category) VALUES
('cloudflare.api_token', '"your_cf_api_token_here"', 'dns'),
('cloudflare.account_id', '"your_cf_account_id"', 'dns'),
('cloudflare.default_proxied', 'true', 'dns'),
('server.primary_ip', '"217.216.72.31"', 'infrastructure');
```

---

## � Localization (Bahasa Indonesia)

### Language Requirements

**Default Language**: **Bahasa Indonesia (id)**

All user-facing interfaces (Admin Panel & Member Panel) must use Indonesian language:

### Admin Panel (Bahasa Indonesia)
```
Dashboard → Dasbor
Users → Pengguna
Transactions → Transaksi
Deposits → Deposit
Withdrawals → Penarikan
Pending Approval → Menunggu Persetujuan
Approve → Setujui
Reject → Tolak
Settings → Pengaturan
Payment Methods → Metode Pembayaran
Reports → Laporan
Analytics → Analitik
Games → Permainan
Referrals → Referral
Lucky Draw → Undian Berhadiah
```

### Member Panel (Bahasa Indonesia)
```
Home → Beranda
Deposit → Deposit
Withdraw → Tarik Dana
History → Riwayat
Profile → Profil
Referral → Referral
Lucky Draw → Undian
Balance → Saldo
Main Balance → Saldo Utama
Referral Balance → Saldo Referral
Play Now → Main Sekarang
Claim → Klaim
Terms & Conditions → Syarat & Ketentuan
```

### Implementation (i18n)

**Frontend (Next.js):**
```typescript
// Use next-intl or react-i18next

// locales/id/common.json
{
  "deposit": "Deposit",
  "withdraw": "Tarik Dana",
  "balance": "Saldo",
  "history": "Riwayat",
  "approve": "Setujui",
  "reject": "Tolak",
  "pending": "Menunggu",
  "success": "Berhasil",
  "failed": "Gagal",
  "amount": "Jumlah",
  "payment_method": "Metode Pembayaran",
  "bank_transfer": "Transfer Bank",
  "qris": "QRIS",
  "pulsa": "Pulsa",
  "ewallet": "E-Wallet"
}

// locales/id/payment.json
{
  "deposit_methods": {
    "qris_auto": "QRIS Otomatis",
    "qris_manual": "QRIS Manual",
    "bank_transfer": "Transfer Bank",
    "ewallet": "E-Wallet",
    "pulsa": "Pulsa"
  },
  "pulsa_conversion": "Konversi Pulsa",
  "you_will_receive": "Anda akan menerima",
  "upload_proof": "Upload Bukti Bayar",
  "waiting_confirmation": "Menunggu Konfirmasi Admin"
}
```

**Backend (NestJS):**
```typescript
// For API error messages and notifications
{
  "errors": {
    "insufficient_balance": "Saldo tidak mencukupi",
    "invalid_amount": "Jumlah tidak valid",
    "payment_failed": "Pembayaran gagal",
    "withdrawal_limit": "Melebihi batas penarikan"
  },
  "notifications": {
    "deposit_success": "Deposit berhasil! Saldo Anda telah ditambahkan.",
    "withdrawal_approved": "Penarikan Anda telah disetujui dan diproses.",
    "withdrawal_rejected": "Penarikan Anda ditolak. Alasan: {{reason}}"
  }
}
```

### Date & Number Formatting

**Indonesian Locale:**
```typescript
// Date format: DD/MM/YYYY HH:mm
// Example: 27/11/2025 13:45

// Number format: Use period (.) for thousands, comma (,) for decimals
// Example: Rp 1.000.000,50

import { format } from 'date-fns';
import { id } from 'date-fns/locale';

// Date formatting
format(new Date(), 'dd/MM/yyyy HH:mm', { locale: id });

// Currency formatting
new Intl.NumberFormat('id-ID', {
  style: 'currency',
  currency: 'IDR',
  minimumFractionDigits: 0
}).format(100000); // Rp100.000
```

### Validation Messages (Indonesian)
```
- "Field wajib diisi"
- "Format email tidak valid"
- "Password minimal 8 karakter"
- "Nomor telepon tidak valid"
- "Minimal penarikan Rp 50.000"
- "Maksimal deposit Rp 10.000.000"
```

---

## �🏗️ Infrastructure Setup



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
