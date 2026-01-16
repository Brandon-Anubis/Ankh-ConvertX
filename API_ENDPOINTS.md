# ConvertX API Endpoint Inventory

## 1. Repository Reconnaissance

### Application Type
**Backend with Server-Side Rendering (SSR)**
- **Runtime**: Bun + Elysia (TypeScript web framework)
- **Primary Languages/Frameworks**: TypeScript, Bun, Elysia.js, KitaJS/HTML (JSX for HTML)
- **Architecture**: Monolithic backend with HTML rendering and file conversion capabilities
- **Port**: 3000 (default)

### Entry Points
- **Main Entry**: `/home/runner/work/Ankh-ConvertX/Ankh-ConvertX/src/index.tsx` - Application bootstrap and server setup
- **Routes**: `/home/runner/work/Ankh-ConvertX/Ankh-ConvertX/src/pages/*.tsx` - Individual route handlers
- **Converters**: `/home/runner/work/Ankh-ConvertX/Ankh-ConvertX/src/converters/*.ts` - File conversion logic

### Key Dependencies
- `elysia` - Web framework (like Express but for Bun)
- `@elysiajs/jwt` - JWT authentication
- `@elysiajs/html` - HTML rendering
- `@elysiajs/static` - Static file serving
- `@kitajs/html` - JSX to HTML compilation

---

## 2. Endpoint Inventory

### Authentication & User Management Endpoints

| Method | Path | Auth? | Request Schema | Response Schema | Handler Location | Notes |
|--------|------|-------|----------------|-----------------|------------------|-------|
| GET | `/setup` | No | None | HTML page | `src/pages/user.tsx:68` | First-run setup page, only accessible when `FIRST_RUN=true` |
| GET | `/register` | No | None | HTML page | `src/pages/user.tsx:129` | Registration page, requires `ACCOUNT_REGISTRATION=true` or `FIRST_RUN=true` |
| POST | `/register` | No | `{ email: string, password: string }` | Redirect to `/` or error JSON | `src/pages/user.tsx:183` | Creates new user account |
| GET | `/login` | No | None | HTML page | `src/pages/user.tsx:238` | Login page |
| POST | `/login` | No | `{ email: string, password: string }` | Redirect to `/` or error JSON | `src/pages/user.tsx:318` | Authenticates user, sets JWT cookie |
| GET | `/logoff` | Yes | None | Redirect to `/login` | `src/pages/user.tsx:363` | Clears auth cookie |
| POST | `/logoff` | Yes | None | Redirect to `/login` | `src/pages/user.tsx:370` | Alternative POST method for logout |
| GET | `/account` | Yes | None | HTML page | `src/pages/user.tsx:377` | User account management page |
| POST | `/account` | Yes | `{ email: string, password: string, newPassword?: string }` | Redirect or error JSON | `src/pages/user.tsx:457` | Updates user account details |

### Core Application Endpoints

| Method | Path | Auth? | Request Schema | Response Schema | Handler Location | Notes |
|--------|------|-------|----------------|-----------------|------------------|-------|
| GET | `/` | Conditional* | None | HTML page | `src/pages/root.tsx:19` | Home page with file upload form |
| GET | `/healthcheck` | No | None | `{ status: "ok" }` | `src/pages/healthcheck.tsx:4` | Health check endpoint for monitoring |
| GET | `/converters` | Yes | None | HTML page | `src/pages/listConverters.tsx:8` | Lists all available converters |
| POST | `/conversions` | Optional** | `{ fileType: string }` | HTML fragment | `src/pages/chooseConverter.tsx:5` | Returns converter options for a file type, typically used within authenticated workflow |

### File Upload & Conversion Endpoints

| Method | Path | Auth? | Request Schema | Response Schema | Handler Location | Notes |
|--------|------|-------|----------------|-----------------|------------------|-------|
| POST | `/upload` | Yes | `{ file: File \| File[] }` | `{ message: string }` | `src/pages/upload.tsx:8` | Uploads files for conversion, requires `jobId` cookie |
| POST | `/convert` | Yes | `{ convert_to: string, file_names: string }` | Redirect to `/results/:jobId` | `src/pages/convert.tsx:12` | Initiates file conversion, requires `jobId` cookie |
| POST | `/delete` | Yes | `{ filename: string }` | `{ message: string }` | `src/pages/deleteFile.tsx:10` | Deletes uploaded file before conversion |

### Results & Download Endpoints

| Method | Path | Auth? | Request Schema | Response Schema | Handler Location | Notes |
|--------|------|-------|----------------|-----------------|------------------|-------|
| GET | `/history` | Yes | None | HTML page | `src/pages/history.tsx:11` | Shows conversion history, disabled if `HIDE_HISTORY=true` |
| GET | `/results/:jobId` | Yes | None | HTML page | `src/pages/results.tsx:133` | Shows conversion results for a job |
| POST | `/results/:jobId/refresh` | Yes | None | HTML fragment | `src/pages/results.tsx:179` | Refreshes conversion status (HTMX endpoint) |
| GET | `/download/:userId/:jobId/:fileName` | Yes | None | File binary | `src/pages/download.tsx:12` | Downloads individual converted file |
| GET | `/archive/:jobId` | Yes | None | TAR archive | `src/pages/download.tsx:34` | Downloads all converted files as tar archive |

### Job Management Endpoints

| Method | Path | Auth? | Request Schema | Response Schema | Handler Location | Notes |
|--------|------|-------|----------------|-----------------|------------------|-------|
| GET | `/delete/:jobId` | Yes | None | Redirect to `/history` | `src/pages/deleteJob.tsx:11` | Deletes a conversion job |
| POST | `/delete-multiple` | Yes | `{ jobIds: string[] }` | `{ success: boolean, deleted: number, failed: number }` | `src/pages/deleteJob.tsx:41` | Batch deletes conversion jobs (max 100) |

### Static Assets

| Method | Path | Auth? | Request Schema | Response Schema | Handler Location | Notes |
|--------|------|-------|----------------|-----------------|------------------|-------|
| GET | `/generated.css` | No | None | CSS file | `src/index.tsx:61` | Development-only Tailwind CSS (only when `NODE_ENV !== "production"`) |
| GET | `/*` | No | None | Static files | `src/index.tsx:36` | Serves files from `/public` directory |

**Conditional Auth*: The `/` endpoint requires auth unless `ALLOW_UNAUTHENTICATED=true`

**Optional Auth**: The `/conversions` endpoint doesn't enforce authentication but is typically used within an authenticated session (e.g., from the home page)

---

## 3. Authentication & Middleware

### Authentication Mechanism
- **Type**: JWT (JSON Web Token) stored in HTTP-only cookie
- **Cookie Name**: `auth`
- **JWT Secret**: Set via `JWT_SECRET` environment variable (defaults to `randomUUID()` if unset)
- **Token Expiration**: 7 days
- **Cookie Settings**:
  - `httpOnly: true` (prevents JavaScript access)
  - `secure: !HTTP_ALLOWED` (requires HTTPS unless `HTTP_ALLOWED=true`)
  - `maxAge: 60 * 60 * 24 * 7` (7 days)
  - `sameSite: "strict"` (CSRF protection)

### Auth Macro
Routes can use `{ auth: true }` or `{ auth: false }` in their configuration. The auth macro:
1. Checks for the `auth` cookie
2. Verifies the JWT token
3. Returns user ID if valid
4. Returns 401 Unauthorized if invalid/missing

Location: `src/pages/user.tsx:43`

### Middleware Stack
1. **HTML Plugin** (`@elysiajs/html`) - Enables HTML rendering
2. **Static Plugin** (`@elysiajs/static`) - Serves static files from `/public`
3. **JWT Plugin** (`@elysiajs/jwt`) - JWT signing and verification
4. **User Service** - Custom auth middleware applied to all protected routes

### CORS
- Not explicitly configured (defaults to same-origin)

### Rate Limiting
- Not implemented

### Base Path / Versioning
- Configurable via `WEBROOT` environment variable (e.g., `/convert` for `example.com/convert/`)
- No API versioning (e.g., `/api/v1`) - single version

---

## 4. Environment Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `JWT_SECRET` | `randomUUID()` | Secret key for signing JWT tokens |
| `ACCOUNT_REGISTRATION` | `false` | Allow new user registrations |
| `HTTP_ALLOWED` | `false` | Allow HTTP connections (disables secure cookies) |
| `ALLOW_UNAUTHENTICATED` | `false` | Allow unauthenticated access to converter |
| `AUTO_DELETE_EVERY_N_HOURS` | `24` | Auto-delete converted files after N hours |
| `WEBROOT` | `""` | Base path for the application |
| `HIDE_HISTORY` | `false` | Hide the conversion history page |
| `LANGUAGE` | `en` | Language for date formatting (BCP 47 tag) |
| `UNAUTHENTICATED_USER_SHARING` | `false` | Share conversion history between unauthenticated users |
| `MAX_CONVERT_PROCESS` | `0` | Max concurrent conversion processes (0 = unlimited) |
| `FFMPEG_ARGS` | `""` | Additional FFmpeg input arguments |
| `FFMPEG_OUTPUT_ARGS` | `""` | Additional FFmpeg output arguments |

---

## 5. How to Hit Them - cURL Examples

### 5.1 Health Check

**Happy Path:**
```bash
curl -X GET http://localhost:3000/healthcheck
```

**Expected Response:**
```json
{
  "status": "ok"
}
```

### 5.2 User Registration (First Run)

**Happy Path:**
```bash
curl -X POST http://localhost:3000/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "securePassword123"
  }' \
  -c cookies.txt \
  -L
```

**Expected Response:** Redirect to `/` with auth cookie set

**Invalid Request:**
```bash
curl -X POST http://localhost:3000/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "short"
  }'
```

**Expected Error Response:**
```json
{
  "message": "Email already in use."
}
```
HTTP Status: 400

### 5.3 User Login

**Happy Path:**
```bash
curl -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "securePassword123"
  }' \
  -c cookies.txt \
  -L
```

**Expected Response:** Redirect to `/` with auth cookie set

**Invalid Request:**
```bash
curl -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "wrongPassword"
  }'
```

**Expected Error Response:**
```json
{
  "message": "Invalid credentials"
}
```
HTTP Status: 400

### 5.4 List Available Converters

**Happy Path:**
```bash
curl -X GET http://localhost:3000/converters \
  -b cookies.txt
```

**Expected Response:** HTML page listing all converters

**Unauthorized Request:**
```bash
curl -X GET http://localhost:3000/converters
```

**Expected Error Response:**
```json
{
  "success": false,
  "message": "Unauthorized"
}
```
HTTP Status: 401

### 5.5 Get Converter Options for File Type

**Happy Path (without authentication):**
```bash
curl -X POST http://localhost:3000/conversions \
  -H "Content-Type: application/json" \
  -d '{
    "fileType": "png"
  }'
```

**Expected Response:** HTML fragment with converter options

**Note:** This endpoint doesn't require authentication, but is typically called from within an authenticated session on the home page.

### 5.6 Upload File for Conversion

**Happy Path:**
```bash
# First, you need a jobId cookie set (this happens when accessing the home page)
curl -X POST http://localhost:3000/upload \
  -b cookies.txt \
  -F "file=@/path/to/image.png"
```

**Expected Response:**
```json
{
  "message": "Files uploaded successfully."
}
```

**Invalid Request (no jobId):**
```bash
curl -X POST http://localhost:3000/upload \
  -F "file=@/path/to/image.png"
```

**Expected Error Response:** Redirect to `/` (302)

### 5.7 Start Conversion

**Happy Path:**
```bash
curl -X POST http://localhost:3000/convert \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{
    "convert_to": "jpg,imagemagick",
    "file_names": "[\"image.png\"]"
  }' \
  -L
```

**Expected Response:** Redirect to `/results/:jobId`

### 5.8 View Conversion Results

**Happy Path:**
```bash
curl -X GET http://localhost:3000/results/abc123 \
  -b cookies.txt
```

**Expected Response:** HTML page with conversion results and download links

### 5.9 Download Converted File

**Happy Path:**
```bash
curl -X GET http://localhost:3000/download/1/abc123/image.jpg \
  -b cookies.txt \
  -o image.jpg
```

**Expected Response:** File binary data

### 5.10 Download All Files as Archive

**Happy Path:**
```bash
curl -X GET http://localhost:3000/archive/abc123 \
  -b cookies.txt \
  -o converted_files.tar
```

**Expected Response:** TAR archive with all converted files

### 5.11 Delete Conversion Job

**Happy Path:**
```bash
curl -X GET http://localhost:3000/delete/abc123 \
  -b cookies.txt \
  -L
```

**Expected Response:** Redirect to `/history`

### 5.12 Batch Delete Jobs

**Happy Path:**
```bash
curl -X POST http://localhost:3000/delete-multiple \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{
    "jobIds": ["abc123", "def456"]
  }'
```

**Expected Response:**
```json
{
  "success": true,
  "deleted": 2,
  "failed": 0,
  "details": {
    "success": ["abc123", "def456"],
    "failed": []
  }
}
```

**Invalid Request:**
```bash
curl -X POST http://localhost:3000/delete-multiple \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{
    "jobIds": []
  }'
```

**Expected Error Response:**
```json
{
  "success": false,
  "message": "Invalid job IDs provided"
}
```
HTTP Status: 400

### 5.13 Logout

**Happy Path:**
```bash
curl -X GET http://localhost:3000/logoff \
  -b cookies.txt \
  -c cookies.txt \
  -L
```

**Expected Response:** Redirect to `/login` with auth cookie cleared

---

## 6. Request/Response Schemas

### Common Response Types

**Success Redirect:**
- HTTP 302 Found
- `Location` header with redirect URL

**JSON Success:**
```typescript
{
  message: string
}
```

**JSON Error:**
```typescript
{
  success: false,
  message: string
}
```

**Unauthorized:**
```typescript
{
  success: false,
  message: "Unauthorized"
}
```
HTTP Status: 401

### Detailed Schemas

**User Registration/Login Request:**
```typescript
{
  email: string,      // Valid email format
  password: string    // Min 6 characters (inferred from typical web standards)
}
```

**File Upload Request:**
- Content-Type: `multipart/form-data`
- Body: `file` field with single or multiple files

**Conversion Request:**
```typescript
{
  convert_to: string,   // Format: "targetFormat,converterName" (e.g., "jpg,imagemagick")
  file_names: string    // JSON stringified array of filenames
}
```

**Delete File Request:**
```typescript
{
  filename: string    // Sanitized filename
}
```

**Batch Delete Request:**
```typescript
{
  jobIds: string[]    // Array of job IDs (max 100 items)
}
```

**Batch Delete Response:**
```typescript
{
  success: boolean,
  deleted: number,
  failed: number,
  details: {
    success: string[],
    failed: Array<{ jobId: string, error: string }>
  }
}
```

---

## 7. Database Schema

The application uses SQLite with the following key tables:

- **users**: Stores user accounts (email, hashed password)
- **jobs**: Stores conversion jobs (id, user_id, num_files, status, date_created)
- **file_names**: Stores file names for each job (job_id, file_name, output_file_name)

Database location: `./data/` (configurable via volume mount in Docker)

---

## 8. Security Considerations

### Implemented
- ✅ JWT-based authentication with HTTP-only cookies
- ✅ Password hashing using Bun's built-in password hashing
- ✅ Filename sanitization to prevent path traversal
- ✅ SameSite cookie attribute for CSRF protection
- ✅ Secure cookie flag (when HTTPS is enabled)
- ✅ User isolation (files stored per user ID)

### Considerations
- ⚠️ No rate limiting on authentication endpoints
- ⚠️ No explicit CORS configuration (same-origin only)
- ⚠️ File upload size is set to `Number.MAX_SAFE_INTEGER` (unlimited)
- ⚠️ Auto-deletion runs every N hours (configurable)

---

## 9. Local Development Setup

1. **Install Bun**:
   ```bash
   curl -fsSL https://bun.sh/install | bash
   ```

2. **Clone and Install**:
   ```bash
   git clone https://github.com/Brandon-Anubis/Ankh-ConvertX.git
   cd Ankh-ConvertX
   bun install
   ```

3. **Run Development Server**:
   ```bash
   bun run dev
   ```

4. **Access Application**:
   - URL: http://localhost:3000
   - First access will prompt for account creation

5. **Build for Production**:
   ```bash
   bun run build
   ```

---

## 10. Testing

The repository includes converter tests in `/tests/converters/*.test.ts`. These are unit tests for individual file converters.

**Run Tests:**
```bash
bun test
```

**Test Files:**
- `tests/converters/*.test.ts` - Converter-specific tests
- `tests/converters/helpers/` - Test helpers and common test utilities

---

## 11. API Integration Patterns

### Frontend Integration
The application uses HTMX for dynamic updates:
- Forms submit via HTMX `hx-post` attributes
- Dynamic content updates use `hx-target` and `hx-swap`
- No separate REST API client needed

### External Integration
For programmatic access:
1. Authenticate via POST `/login` to get JWT cookie
2. Store cookie for subsequent requests
3. Use file upload and conversion endpoints
4. Poll `/results/:jobId` for status
5. Download files when conversion is complete

### WebSocket/SSE
- Not implemented (use polling for real-time updates)

---

## 12. Deployment Notes

### Docker Deployment

**Original Upstream Repository (C4illin/ConvertX):**
```bash
docker run -p 3000:3000 \
  -v ./data:/app/data \
  -e JWT_SECRET=your-secret-key \
  ghcr.io/c4illin/convertx:latest
```

**Note:** This is a fork of the original ConvertX project. The upstream repository is `C4illin/ConvertX`.

### Environment Setup
- Set `JWT_SECRET` for production (strongly recommended)
- Use `HTTP_ALLOWED=false` in production (default)
- Configure `WEBROOT` if behind a reverse proxy with path prefix
- Set `AUTO_DELETE_EVERY_N_HOURS` based on storage capacity

### Reverse Proxy
Compatible with nginx, Traefik, Caddy. Example nginx config:
```nginx
location /convert/ {
    proxy_pass http://localhost:3000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

Remember to set `WEBROOT=/convert` in the environment.

---

## Summary

ConvertX is a **fully functional web application** with **existing HTTP API endpoints**. It uses Elysia (Bun's web framework) for routing, JWT for authentication, and provides both HTML pages and JSON responses depending on the endpoint. The application is production-ready with proper authentication, file handling, and conversion capabilities across 19+ converter backends.

**Total Endpoints**: 23 routes (not counting static file serving)
**Authentication**: JWT with HTTP-only cookies
**API Style**: Hybrid SSR + REST (HTML responses for pages, JSON for API operations)
**Framework**: Elysia.js (TypeScript, Bun runtime)
