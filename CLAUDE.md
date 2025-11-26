# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenIM Chat is a business application layer built on top of OpenIM Server. It implements user management, authentication, and admin functionality while delegating core IM features to the OpenIM Server. The system consists of four main services: chat-api, chat-rpc, admin-api, and admin-rpc.

## Build and Development Commands

### Initial Setup
```bash
# First time setup (Linux/Mac)
sh bootstrap.sh

# First time setup (Windows)
bootstrap.bat
```

### Building
```bash
# Build all services
mage

# Build specific services
mage build chat-api chat-rpc
mage build admin-api admin-rpc
```

### Running Services
```bash
# Start all services
mage start

# Start specific services
mage start chat-api admin-rpc

# Start in background with logs
nohup mage start >> _output/logs/chat.log 2>&1 &
```

### Management
```bash
# Check service status
mage check

# Stop all services
mage stop
```

### Testing and Code Generation
```bash
# Regenerate protobuf code after modifying .proto files
cd pkg/protocol && ./gen.sh
```

## Architecture

### Service Structure

The project follows a **dual-layer architecture** with REST APIs backed by gRPC services:

```
REST API (Gin) → gRPC Client → RPC Service (gRPC) → Database (MongoDB/Redis)
```

**Four main services:**
1. **chat-api** (port 10008) - User-facing REST API for registration, login, profiles, search
2. **chat-rpc** (port 30300) - gRPC service with core chat business logic and database access
3. **admin-api** - Admin dashboard REST API for user/group management, statistics, system config
4. **admin-rpc** - gRPC service for administrative operations

### Service Dependencies

- **OpenIM Server** must be running first (this is a business layer on top of it)
- **MongoDB** - Primary database
- **Redis** - Caching and sessions
- **etcd** - Service discovery and configuration
- **LiveKit** (optional) - Audio/video call functionality

### Code Organization

```
cmd/
  ├── api/chat-api/       # Chat REST API entry point
  ├── api/admin-api/      # Admin REST API entry point
  ├── rpc/chat-rpc/       # Chat gRPC service entry point
  └── rpc/admin-rpc/      # Admin gRPC service entry point

internal/
  ├── api/                # REST API handlers (Gin routes)
  │   ├── chat/           # User API implementations
  │   └── admin/          # Admin API implementations
  └── rpc/                # gRPC service implementations
      ├── chat/           # User RPC logic (user.go, account.go, etc.)
      └── admin/          # Admin RPC logic

pkg/
  ├── protocol/           # Protobuf definitions (.proto files)
  ├── common/             # Shared libraries
  │   ├── config/         # Configuration structs
  │   ├── db/             # Database models and interfaces
  │   └── mctx/           # Context utilities (auth, user info)
  ├── rpclient/           # gRPC client wrappers
  └── util/               # Utility functions

config/                   # YAML configuration files
  ├── chat-api-chat.yml   # Chat API config
  ├── chat-rpc-chat.yml   # Chat RPC config (includes LiveKit settings)
  ├── admin-api-admin.yml # Admin API config
  ├── admin-rpc-admin.yml # Admin RPC config
  ├── discovery.yml       # etcd service discovery
  ├── mongodb.yml         # Database connection
  ├── redis.yml           # Cache configuration
  ├── share.yml           # OpenIM server integration
  └── log.yml             # Logging configuration
```

## Adding New REST/RPC APIs

Full guide: See `HOW_TO_ADD_REST_RPC_API.md`

**Quick workflow:**

1. **Define protobuf messages and RPC method** in `pkg/protocol/{service}/{service}.proto`
   ```protobuf
   message YourNewReq { string field = 1; }
   message YourNewResp { string result = 1; }

   service chat {
     rpc YourNewMethod(YourNewReq) returns (YourNewResp);
   }
   ```

2. **Generate protobuf code**
   ```bash
   cd pkg/protocol && ./gen.sh
   ```

3. **Add validation** in `pkg/protocol/{service}/{service}.go`
   ```go
   func (x *YourNewReq) Check() error {
     if x.Field == "" {
       return errs.ErrArgs.WrapMsg("Field is required")
     }
     return nil
   }
   ```

4. **Register REST route** in `internal/api/chat/start.go` (for chat API) or `internal/api/admin/start.go` (for admin API)
   ```go
   func SetChatRoute(router gin.IRouter, chat *Api, mw *chatmw.MW) {
     user := router.Group("/user", mw.CheckToken)
     user.POST("/your/endpoint", chat.YourNewMethod)
   }
   ```

5. **Implement API handler** in `internal/api/{service}/{service}.go`
   ```go
   func (o *ChatApi) YourNewMethod(c *gin.Context) {
     a2r.Call(chat.ChatClient.YourNewMethod, o.chatClient, c)
   }
   ```

6. **Implement RPC logic** in `internal/rpc/{service}/{domain}.go`
   ```go
   func (o *chatSvr) YourNewMethod(ctx context.Context, req *chat.YourNewReq) (*chat.YourNewResp, error) {
     userID, _, err := mctx.Check(ctx)  // Extract auth info
     if err != nil {
       return nil, err
     }
     // Your business logic here
     return &chat.YourNewResp{Result: "success"}, nil
   }
   ```

## Key Patterns

### Authentication
- REST APIs use JWT tokens validated by `mw.CheckToken` middleware
- Extract user info from context: `userID, _, err := mctx.Check(ctx)`
- Admin endpoints use `mw.CheckAdmin` middleware

### Error Handling
- Use `pkg/eerrs` package for consistent error responses
- Common OpenIM base errors: `errs.ErrArgs`, `errs.ErrToken`, `errs.ErrRecordNotFound`
- Chat-specific errors (defined in `pkg/eerrs/predefine.go`):
  - `ErrPassword`, `ErrAccountNotFound` - Authentication
  - `ErrPhoneAlreadyRegister`, `ErrAccountAlreadyRegister`, `ErrEmailAlreadyRegister` - Registration
  - `ErrVerifyCodeNotMatch`, `ErrVerifyCodeExpired`, `ErrVerifyCodeUsed` - Verification codes
  - `ErrInvitationCodeUsed`, `ErrInvitationNotFound` - Invitation system
  - `ErrForbidden`, `ErrRefuseFriend` - Access control

### Database Access
- MongoDB models defined in `pkg/common/db/table/`
  - **Chat tables** (`pkg/common/db/table/chat/`): Account, Credential, Attribute, Register, VerifyCode, UserLoginRecord
  - **Admin tables** (`pkg/common/db/table/admin/`): Admin, Applet, Application, ClientConfig, ForbiddenAccount, InvitationRegister, IPForbidden, LimitUserLoginIP, RegisterAddFriend, RegisterAddGroup
- Database interfaces in `pkg/common/db/database/`
- Access patterns: RPC service → database interface → MongoDB

### Configuration
- Loaded from `config/*.yml` files at startup
- Uses Viper for YAML parsing
- Hierarchical: share.yml + service-specific configs

### Service Discovery
- Services register with etcd on startup
- RPC clients discover services via etcd
- Configured in `discovery.yml`

## Important Files

- `magefile.go` - Build system configuration (uses Mage, not Make)
- `start-config.yml` - Defines which services to start and tool binaries
- `pkg/protocol/gen.sh` - Protobuf code generation script
- `internal/api/chat/start.go` - Chat API route definitions (SetChatRoute function)
- `internal/api/admin/start.go` - Admin API route definitions (SetAdminRoute function)
- `internal/api/mw/` - Middleware (auth, validation, CORS)
- `pkg/common/mctx/get.go` - Context utilities for extracting user info
- `pkg/eerrs/predefine.go` - Chat-specific error codes

## Configuration Notes

- All services load configs from `./config/` directory
- Binary outputs go to `_output/` directory
- Logs typically written to `_output/logs/`
- LiveKit configuration in `chat-rpc-chat.yml` for video/audio calls
- SMS verification via Aliyun (configured in chat-rpc config)

## Dependencies

Key external services:
- OpenIM Server (https://github.com/openimsdk/open-im-server) - Must be running first
- MongoDB - User data, accounts, verification codes
- Redis - Session caching
- etcd - Service registry
- LiveKit (optional) - Video/audio calls

Setup LiveKit: See `HOW_TO_SETUP_LIVEKIT_SERVER.md`
