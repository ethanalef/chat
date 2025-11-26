# OpenIM Chat - Project Overview

## Table of Contents

- [What is OpenIM Chat?](#what-is-openim-chat)
- [Project Purpose](#project-purpose)
- [Architecture Overview](#architecture-overview)
- [Four Core Services](#four-core-services)
- [Features](#features)
- [Dependencies](#dependencies)
- [Usage](#usage)
- [Development Lifecycle](#development-lifecycle)
- [Request Flow](#request-flow)
- [Configuration](#configuration)

---

## What is OpenIM Chat?

**OpenIM Chat** is a business application layer built on top of [OpenIM Server](https://github.com/openimsdk/open-im-server). It implements user management, authentication, and administrative functionality while delegating core instant messaging features to the OpenIM Server.

### Key Characteristics

- **Language**: Go 1.22.7+ (toolchain 1.23.2)
- **Version**: v1.8.4-patch.3
- **License**: Dual-licensed (GPLv3 or Commercial)
- **Platform**: Cross-platform (Linux/Windows/Mac, ARM/AMD64)
- **Architecture**: Microservices (4 independent services)

---

## Project Purpose

OpenIM Chat serves as the **business logic layer** that sits between client applications and the core OpenIM instant messaging system. It provides:

1. **User System** - Registration, login, profile management, search
2. **Admin System** - User/group management, statistics, system configuration
3. **Integration Layer** - Bridges client applications with OpenIM Server

**Why separate from OpenIM Server?**
- Separation of concerns (business logic vs. core IM functionality)
- Independent scaling and deployment
- Customizable business rules without modifying core IM system
- Multi-tenancy and enterprise features

---

## Architecture Overview

### High-Level Architecture

```
┌─────────────┐
│   Clients   │ (Web, Mobile, Desktop)
└──────┬──────┘
       │ HTTP/REST
       ▼
┌─────────────────────────────────────┐
│     OpenIM Chat (This Project)      │
│  ┌──────────────┬──────────────┐   │
│  │  chat-api    │  admin-api   │   │ REST Layer (Gin)
│  │  :10008      │              │   │
│  └──────┬───────┴──────┬───────┘   │
│         │ gRPC         │ gRPC       │
│  ┌──────▼───────┬──────▼───────┐   │
│  │  chat-rpc    │  admin-rpc   │   │ Business Logic Layer
│  │  :30300      │              │   │
│  └──────┬───────┴──────┬───────┘   │
└─────────┼──────────────┼───────────┘
          │              │
    ┌─────▼──────┐  ┌────▼────────┐
    │  MongoDB   │  │   Redis     │
    │  :37017    │  │   :16379    │
    └────────────┘  └─────────────┘
          │
          │ HTTP API
          ▼
    ┌─────────────────┐
    │  OpenIM Server  │ Core IM functionality
    │    :10002       │ (Must be running first)
    └─────────────────┘
```

### Dual-Layer Pattern

All functionality follows this pattern:
```
Client Request → API Service (HTTP) → RPC Service (gRPC) → Database/OpenIM Server
```

This provides:
- **API Layer**: Authentication, validation, HTTP handling
- **RPC Layer**: Business logic, database operations, OpenIM integration

---

## Four Core Services

### 1. **chat-api** (User REST API)
- **Port**: 10008
- **Purpose**: User-facing REST endpoints
- **Entry Point**: `cmd/api/chat-api/main.go`
- **Config**: `config/chat-api-chat.yml`

**Key Endpoints:**
- `/account/*` - Registration, login, verification codes, password management
- `/user/*` - Profile updates, user search, video meeting tokens
- `/friend/search` - Friend search
- `/applet/find` - Applet listing
- `/client_config/get` - Client initialization config
- `/callback/open_im` - OpenIM callbacks

### 2. **chat-rpc** (User Business Logic)
- **Port**: 30300
- **Purpose**: gRPC service for chat business logic
- **Entry Point**: `cmd/rpc/chat-rpc/main.go`
- **Config**: `config/chat-rpc-chat.yml`

**Responsibilities:**
- User registration and authentication
- Account management
- Verification code handling (SMS via Aliyun, Email)
- Real-time communication (RTC) integration with LiveKit
- User statistics and login records
- OpenIM callback processing

### 3. **admin-api** (Admin REST API)
- **Purpose**: Admin dashboard REST endpoints
- **Entry Point**: `cmd/api/admin-api/main.go`
- **Config**: `config/chat-api-admin.yml`

**Key Endpoint Groups:**
- `/admin/*` - Admin account management
- `/user/import/*` - Bulk user import (JSON/Excel)
- `/default/*` - Default friends/groups for new users
- `/invitation_code/*` - Invitation code system
- `/forbidden/*` - IP/user access control
- `/applet/*` - Applet management
- `/block/*` - User blocking
- `/client_config/*` - System configuration
- `/statistic/*` - User statistics
- `/application/*` - Application version management

### 4. **admin-rpc** (Admin Business Logic)
- **Purpose**: gRPC service for administrative operations
- **Entry Point**: `cmd/rpc/admin-rpc/main.go`
- **Config**: `config/chat-rpc-admin.yml`

**Responsibilities:**
- Admin authentication and authorization
- User management (CRUD, import, block)
- Invitation code generation and validation
- IP-based access control
- Default friend/group configuration
- Application version management
- System statistics and reporting

---

## Features

### User System Features

#### Authentication & Registration
- ✅ SMS verification codes (Aliyun integration)
- ✅ Email verification codes
- ✅ User registration with verification
- ✅ Login with phone/email/account
- ✅ Password reset and change
- ✅ JWT token-based authentication
- ✅ Invitation code support

#### User Management
- ✅ Profile information update
- ✅ Public and full user info queries
- ✅ User search functionality
- ✅ Friend search
- ✅ Login history tracking

#### Real-Time Communication
- ✅ Video/audio meeting support via LiveKit
- ✅ RTC token generation
- ✅ WebRTC integration

#### Integration
- ✅ OpenIM Server callback handling
- ✅ Client configuration management
- ✅ Applet discovery

### Admin System Features

#### Admin Management
- ✅ Admin account CRUD operations
- ✅ Admin authentication
- ✅ Password management
- ✅ Role-based access control

#### User Management
- ✅ User account creation and management
- ✅ Bulk user import (JSON/Excel formats)
- ✅ User search and filtering
- ✅ Password reset
- ✅ User blocking/unblocking

#### Access Control
- ✅ IP-based registration/login restrictions
- ✅ User-specific IP login limits
- ✅ Forbidden account management
- ✅ Registration control (enable/disable)

#### Invitation System
- ✅ Invitation code generation
- ✅ Invitation code management (add/delete/search)
- ✅ Usage tracking and validation

#### Default Configuration
- ✅ Default friends added at registration
- ✅ Default groups added at registration
- ✅ Customizable per-deployment

#### Applet Management
- ✅ Applet registration
- ✅ Applet CRUD operations
- ✅ Applet discovery for users

#### Application Version Management
- ✅ Version registration
- ✅ Version updates
- ✅ Latest version query
- ✅ Version history pagination

#### Statistics & Analytics
- ✅ New user count statistics
- ✅ Login user count tracking
- ✅ Time-based reporting

#### System Configuration
- ✅ Client initialization config management
- ✅ Dynamic configuration via etcd
- ✅ Per-client customization

---

## Dependencies

### External Services (Must Be Running)

| Service | Default Address | Purpose | Required |
|---------|----------------|---------|----------|
| **OpenIM Server** | http://127.0.0.1:10002 | Core IM functionality | ✅ Yes |
| **MongoDB** | mongodb://127.0.0.1:37017 | Primary database | ✅ Yes |
| **Redis** | redis://127.0.0.1:16379 | Caching, sessions | ✅ Yes |
| **etcd** | http://127.0.0.1:12379 | Service discovery, config | ✅ Yes |
| **LiveKit** | ws://127.0.0.1:7880 | Audio/video calls | ⭕ Optional |

### Key Go Dependencies

#### Web Framework & Network
```go
github.com/gin-gonic/gin v1.9.1              // HTTP web framework
google.golang.org/grpc v1.68.0               // gRPC framework
google.golang.org/protobuf v1.35.1           // Protocol Buffers
```

#### Database & Cache
```go
go.mongodb.org/mongo-driver v1.14.0          // MongoDB client
github.com/redis/go-redis/v9 v9.5.1          // Redis client
gorm.io/gorm v1.25.8                         // ORM (if used)
```

#### Service Discovery & Configuration
```go
go.etcd.io/etcd/client/v3 v3.5.13           // etcd client
github.com/spf13/viper v1.18.2              // Configuration management
github.com/spf13/cobra v1.8.0               // CLI commands
```

#### OpenIM Integration
```go
github.com/openimsdk/protocol v0.0.73-alpha.5  // OpenIM protocols
github.com/openimsdk/tools v0.0.50-alpha.65    // OpenIM utilities
```

#### Third-Party Integrations
```go
github.com/livekit/protocol v1.10.1                       // LiveKit RTC
github.com/alibabacloud-go/dysmsapi-20170525/v2 v2.0.18  // Aliyun SMS
github.com/xuri/excelize/v2 v2.8.0                       // Excel processing
gopkg.in/gomail.v2 v2.0.0                                 // Email sending
```

#### Utilities
```go
github.com/golang-jwt/jwt/v4 v4.5.0         // JWT authentication
github.com/google/uuid v1.6.0               // UUID generation
github.com/jinzhu/copier v0.4.0             // Struct copying
github.com/mitchellh/mapstructure v1.5.0    // Map to struct conversion
```

#### Build Tools
```go
github.com/openimsdk/gomake v0.0.15-alpha.11  // Mage build tool
```

---

## Usage

### Prerequisites

1. **OpenIM Server must be running first**
   - See: https://github.com/openimsdk/open-im-server
   - Default API: http://127.0.0.1:10002

2. **External services must be available**
   - MongoDB (port 37017)
   - Redis (port 16379)
   - etcd (port 12379)

### Initial Setup

**Linux/Mac:**
```bash
git clone https://github.com/openimsdk/chat openim-chat
cd openim-chat
sh bootstrap.sh
```

**Windows:**
```bash
git clone https://github.com/openimsdk/chat openim-chat
cd openim-chat
bootstrap.bat
```

The bootstrap script:
- Installs Mage build tool
- Downloads Go dependencies
- Verifies system requirements

### Build

```bash
# Build all services
mage

# Build specific services
mage build chat-api chat-rpc
mage build admin-api admin-rpc

# Build with tools
mage build chat-api chat-rpc check-component
```

Binaries are output to `_output/bin/platforms/linux/amd64/` (or your platform).

### Run

```bash
# Start all services (foreground)
mage start

# Start specific services
mage start chat-api admin-api

# Start in background with logging
nohup mage start >> _output/logs/chat.log 2>&1 &
```

### Management

```bash
# Check service status
mage check

# Stop all services
mage stop
```

### Testing

```bash
# Run all tests
go test ./...

# Run tests for specific package
go test ./internal/rpc/chat/...

# Run with coverage
go test -cover ./...

# Run with verbose output
go test -v ./...
```

### Code Generation

After modifying `.proto` files:

```bash
cd pkg/protocol
./gen.sh
```

This regenerates Go code from Protocol Buffer definitions.

---

## Development Lifecycle

### 1. Initial Setup Phase

```
Clone Repository → Run bootstrap.sh → Configure config/*.yml → Verify Dependencies
```

### 2. Development Phase

```
Modify Code → Write Tests → Run Locally → Debug
     ↓            ↓            ↓           ↓
  (Edit)      (go test)   (mage start) (logs)
```

### 3. Build Phase

```
Update .proto (if needed) → Generate Code → Build Binaries → Verify Build
          ↓                      ↓              ↓              ↓
  (pkg/protocol/*.proto)    (./gen.sh)      (mage)       (mage check)
```

### 4. Deployment Phase

```
Build → Package → Deploy → Configure → Start Services
  ↓        ↓        ↓         ↓            ↓
(mage)  (docker)  (k8s)   (etcd)    (mage start)
```

### 5. Maintenance Phase

```
Monitor → Analyze Issues → Hotfix → Deploy → Verify
   ↓           ↓             ↓        ↓        ↓
(logs)    (debugging)    (patch)  (rolling) (check)
```

---

## Request Flow

### User Registration Flow

```
1. Client Application
   ↓ POST /account/code/send
2. chat-api (Port 10008)
   ├─ CheckToken middleware (skipped for public endpoint)
   ├─ Parse request body
   └─ Call chat-rpc via gRPC
      ↓
3. chat-rpc (Port 30300)
   ├─ Generate verification code
   ├─ Store in MongoDB (verify_code collection)
   ├─ Send SMS via Aliyun or Email
   └─ Return success
      ↓
4. User receives SMS/Email with code
   ↓ POST /account/code/verify
5. chat-api → chat-rpc
   ├─ Validate code from MongoDB
   ├─ Check expiry and usage count
   └─ Mark code as used
      ↓
6. Client submits registration
   ↓ POST /account/register
7. chat-api → chat-rpc
   ├─ Validate verification code (already verified)
   ├─ Create account in MongoDB (account collection)
   ├─ Create user in OpenIM Server via HTTP API
   ├─ Add default friends (if configured)
   ├─ Add to default groups (if configured)
   ├─ Generate JWT token
   └─ Return user info + token
      ↓
8. Client stores token for future requests
```

### Authenticated User Request Flow

```
1. Client Application
   ↓ POST /user/update (with Authorization header)
2. chat-api (Port 10008)
   ├─ CheckToken middleware
   │  ├─ Extract JWT from Authorization header
   │  ├─ Validate JWT signature
   │  ├─ Extract userID and platform
   │  └─ Inject into context
   ├─ Parse request body
   └─ Call chat-rpc via gRPC (with userID in context)
      ↓
3. chat-rpc (Port 30300)
   ├─ Extract userID from context (mctx.Check)
   ├─ Query MongoDB for user info
   ├─ Update user attributes
   ├─ Update OpenIM Server via HTTP API
   └─ Return updated user info
      ↓
4. chat-api returns JSON response to client
```

### Admin Operation Flow

```
1. Admin Dashboard/Client
   ↓ POST /admin/user/add (with admin token)
2. admin-api
   ├─ CheckAdmin middleware
   │  ├─ Validate admin JWT token
   │  ├─ Verify admin privileges
   │  └─ Inject admin context
   ├─ Parse request body
   └─ Call admin-rpc via gRPC
      ↓
3. admin-rpc
   ├─ Extract admin userID from context
   ├─ Validate admin permissions
   ├─ Create user account in MongoDB
   ├─ Create user in OpenIM Server
   ├─ Apply default configurations
   ├─ Record admin action (audit log)
   └─ Return created user info
      ↓
4. admin-api returns JSON response
```

### Service Discovery Flow (etcd)

```
1. Service Startup (any of 4 services)
   ↓
2. Connect to etcd (127.0.0.1:12379)
   ↓
3. Register service instance
   ├─ Service name: "chat-rpc-service" or "admin-rpc-service"
   ├─ Instance address: IP:Port
   ├─ Health check interval
   └─ Lease TTL
   ↓
4. Keep-alive lease with etcd
   ├─ Periodic heartbeat
   └─ Auto-deregister on failure
   ↓
5. API services discover RPC services
   ├─ Watch etcd for service endpoints
   ├─ Build gRPC connection pool
   └─ Load balance across instances
   ↓
6. Service shutdown
   └─ Deregister from etcd
```

---

## Configuration

### Configuration Files Structure

```
config/
├── chat-api-chat.yml      # Chat API service config
├── chat-rpc-chat.yml      # Chat RPC service config
├── chat-api-admin.yml     # Admin API service config (or chat-api-chat.yml)
├── chat-rpc-admin.yml     # Admin RPC service config
├── discovery.yml          # etcd service discovery config
├── mongodb.yml            # MongoDB connection config
├── redis.yml              # Redis connection config
├── share.yml              # Shared config (OpenIM integration)
└── log.yml                # Logging configuration
```

### Configuration Hierarchy

```
Base Configs (loaded by all services)
├── discovery.yml          # Service discovery settings
├── mongodb.yml            # Database settings
├── redis.yml              # Cache settings
├── share.yml              # OpenIM integration
└── log.yml                # Logging settings
    ↓
Service-Specific Configs (override base)
├── chat-rpc-chat.yml      # Chat RPC: verification codes, LiveKit
├── chat-api-chat.yml      # Chat API: ports, endpoints
├── chat-rpc-admin.yml     # Admin RPC: admin settings
└── chat-api-admin.yml     # Admin API: admin ports
```

### Key Configuration Sections

#### Service Registration (discovery.yml)
```yaml
enable: "etcd"
etcd:
  address: [ 127.0.0.1:12379 ]
  rootDirectory: openim
rpcService:
  chat: chat-rpc-service
  admin: admin-rpc-service
```

#### OpenIM Integration (share.yml)
```yaml
openIM:
  apiURL: http://127.0.0.1:10002
  secret: openIM123
  adminUserID: imAdmin
chatAdmin:
  - chatAdmin
```

#### Database (mongodb.yml)
```yaml
uri: mongodb://127.0.0.1:37017/openim_v3?maxPoolSize=100
database: openim_v3
```

#### Verification Codes (chat-rpc-chat.yml)
```yaml
verifyCode:
  validTime: 300        # 5 minutes
  validCount: 5         # Max verification attempts
  maxCount: 10          # Max sends per day
  superCode: "666666"   # Dev/testing bypass code
  phone:
    use: "ali"
    ali:
      endpoint: ""
      accessKeyId: ""
      accessKeySecret: ""
```

#### LiveKit RTC (chat-rpc-chat.yml)
```yaml
liveKit:
  url: "ws://127.0.0.1:7880"
  key: "APIGPW3gnFTzqHH"
  secret: "23ztfSqsfQ8hKkHzHTl3Z4bvaxro0snjk5jwbp5p6Q3"
```

### Dynamic Configuration

Configurations can be updated at runtime via etcd:
- Services watch etcd for config changes
- Automatic reload without service restart
- Managed through admin APIs or etcd CLI

---

## Database Schema

### MongoDB Collections

**Chat Database Collections:**
- `account` - User accounts (userID, phone, email, account name)
- `credential` - Login credentials (password hashes)
- `attribute` - User attributes and profile data
- `register` - Registration records
- `verify_code` - Verification codes for phone/email
- `user_login_record` - Login history and session tracking

**Admin Database Collections:**
- `admin` - Admin accounts and roles
- `applet` - Registered applets
- `application_version` - Application version history
- `client_config` - Client initialization configurations
- `forbidden_account` - Blocked user accounts
- `invitation_register` - Invitation codes and usage
- `ip_forbidden` - Banned IP addresses
- `limit_user_login_ip` - User-specific IP restrictions
- `register_add_friend` - Default friends for new users
- `register_add_group` - Default groups for new users

---

## Port Reference

| Service | Default Port | Purpose |
|---------|-------------|---------|
| chat-api | 10008 | User REST API |
| chat-rpc | 30300 | Chat gRPC service |
| admin-api | (configurable) | Admin REST API |
| admin-rpc | (configurable) | Admin gRPC service |
| OpenIM Server | 10002 | Core IM API |
| MongoDB | 37017 | Database |
| Redis | 16379 | Cache |
| etcd | 12379 | Service discovery |
| LiveKit | 7880 | WebRTC server |

---

## Additional Resources

- **Main Documentation**: See `README.md`
- **API Development Guide**: See `HOW_TO_ADD_REST_RPC_API.md`
- **LiveKit Setup**: See `HOW_TO_SETUP_LIVEKIT_SERVER.md`
- **AI Assistant Guide**: See `CLAUDE.md`
- **Q&A**: See `09_Q&A.md`
- **OpenIM Server**: https://github.com/openimsdk/open-im-server
- **Contributing**: See `CONTRIBUTING.md`

---

## License

OpenIM Chat is dual-licensed:
- **Open Source**: GNU General Public License v3.0 (GPLv3)
- **Commercial**: Contact contact@openim.io for commercial licensing

See `LICENSE` file for details.
