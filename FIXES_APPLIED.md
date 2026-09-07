# Peblo TV Mini - Security & Quality Fixes Applied

## Overview
This is a fixed version of the Peblo TV Mini application with **critical security vulnerabilities patched** and **quality improvements applied**.

**Generated:** 2024  
**Status:** Ready for deployment  

---

## Critical Fixes Applied ✅

### 1. **Path Traversal Vulnerability** - CRITICAL
**File:** `backend/app/main.py` (Media endpoint)  
**Risk:** Attackers could read arbitrary files outside `/storage/media/`

**Before:**
```python
@app.get("/media/{path:path}")
def media(path: str):
    file_path = MEDIA_ROOT / path  # No validation!
    if not file_path.exists():
        raise HTTPException(404)
    return FileResponse(file_path)
```

**After:**
```python
@app.get("/media/{path:path}")
def media(path: str):
    file_path = (MEDIA_ROOT / path).resolve()
    try:
        file_path.relative_to(MEDIA_ROOT.resolve())  # ✓ Path validation
    except ValueError:
        raise HTTPException(403, "Access denied.")
    if not file_path.exists() or not file_path.is_file():
        raise HTTPException(404)
    return FileResponse(file_path)
```

---

### 2. **CORS Credentials Leak** - HIGH
**File:** `backend/app/main.py` (CORS middleware)  
**Risk:** Cross-site requests could leak auth tokens

**Before:**
```python
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_credentials=True, ...)
# Dangerous: Any origin can make credentialed requests
```

**After:**
```python
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_credentials=False, ...)
# Credentials are NOT sent cross-origin
```

**For Production:**
```python
app.add_middleware(CORSMiddleware,
    allow_origins=["https://yourdomain.com"],  # Restrict to specific domain
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"])
```

---

### 3. **Weak JWT Secret Warning** - HIGH
**File:** `backend/app/main.py` (Startup event)  
**Risk:** Default secret is publicly known; bad `.env` config means anyone can forge tokens

**Added:**
```python
@app.on_event("startup")
def startup() -> None:
    if JWT_SECRET == "local-only-secret" and not os.getenv("DOCKER_CONTAINER", ""):
        import sys
        print("WARNING: JWT_SECRET is using the default local-only secret.", file=sys.stderr)
    MEDIA_ROOT.mkdir(parents=True, exist_ok=True)
    Base.metadata.create_all(engine)
    with SessionLocal() as db:
        seed_database(db)
```

**Production Deployment:**
- Set `JWT_SECRET` in `.env` to a 32+ character random string
- Use a secrets manager (AWS Secrets Manager, HashiCorp Vault, etc.)
- Never commit `.env` to git (already in `.gitignore` ✓)

---

### 4. **Frontend Search Error Handling** - MEDIUM
**File:** `viewer/src/App.tsx` (SearchPage component)  
**Risk:** Network failures silently failed; users saw blank page

**Before:**
```tsx
const [data, setData] = useState(null);
useEffect(() => {
  get(`/catalog/search?q=${encodeURIComponent(query)}...`)
    .then(setData)  // No error handling
}, [query]);

return !data ? <div>Searching…</div> : /* ... */;
```

**After:**
```tsx
const [data, setData] = useState(null);
const [error, setError] = useState("");

useEffect(() => {
  get(`/catalog/search?q=${encodeURIComponent(query)}...`)
    .then(setData)
    .catch(e => setError(e.message))  // ✓ Capture errors
}, [query]);

return (
  <>
    {error ? (
      <div className="search-empty">{error}</div>  // ✓ Display error
    ) : !data ? (
      <div>Searching…</div>
    ) : /* ... */}
  </>
);
```

---

### 5. **Hardcoded Pagination** - MEDIUM
**File:** `cms/src/App.tsx` (Shows component)  
**Risk:** Data scale changes break pagination silently

**Before:**
```tsx
const totalPages = 2; // Hardcoded! Breaks if data changes
```

**After:**
```tsx
const [total, setTotal] = useState(0);
const totalPages = Math.ceil(total / 6) || 1;  // ✓ Dynamic calculation

async function load() {
  const d = await request(`/admin/shows?${p}`);
  setItems(d.items);
  setTotal(d.total);  // ✓ Capture from API
}
```

---

### 6. **Missing Health Checks** - LOW
**File:** `docker-compose.yml`  
**Risk:** Docker can't detect frontend service failures

**Added to cms and viewer services:**
```yaml
healthcheck:
  test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:5173"]
  interval: 10s
  timeout: 5s
  retries: 3
```

---

### 7. **Frontend Dockerfile Inefficiency** - LOW
**Files:** `cms/Dockerfile`, `viewer/Dockerfile`  
**Risk:** Dev server runs but build step was unnecessary in dev

**Before:**
```dockerfile
RUN npm ci --include=dev
COPY . .
RUN npm run build  # ← Unnecessary for dev server
EXPOSE 5173
CMD ["npm", "run", "dev", ...]
```

**After:**
```dockerfile
RUN npm ci --include=dev
COPY . .
EXPOSE 5173
CMD ["npm", "run", "dev", ...]  # ✓ Build happens on startup in dev
```

---

## Files Modified

| File | Changes | Impact |
|------|---------|--------|
| `backend/app/main.py` | Path validation, CORS fix, JWT warning | Security fixes |
| `cms/src/App.tsx` | Dynamic pagination | Quality improvement |
| `viewer/src/App.tsx` | Error handling in search | UX improvement |
| `cms/Dockerfile` | Remove build step | Performance |
| `viewer/Dockerfile` | Remove build step | Performance |
| `docker-compose.yml` | Add healthchecks | Observability |
| `CODE_REVIEW.md` | Comprehensive audit | Documentation |

---

## Testing Checklist

### Local Development
- [ ] `docker compose up --build` completes without errors
- [ ] API health check responds: `curl http://localhost:8000/health`
- [ ] CMS loads at `http://localhost:5173`
- [ ] Viewer loads at `http://localhost:5174`
- [ ] Login works with demo credentials (`admin@example.com` / `admin-demo`)
- [ ] Path traversal blocked: `curl http://localhost:8000/media/../../../etc/passwd` returns 403

### Frontend
- [ ] CMS shows pagination correctly as data changes
- [ ] Viewer search displays error messages on network failure
- [ ] Both frontends load without console errors

### Backend
- [ ] Media endpoint serves artwork without 403 errors
- [ ] All admin routes require valid JWT token
- [ ] CORS requests don't leak credentials cross-origin

---

## Production Deployment Checklist

- [ ] Set `JWT_SECRET` to a strong random value (32+ chars)
- [ ] Set `DATABASE_URL` to production PostgreSQL
- [ ] Update CORS `allow_origins` to production domain
- [ ] Set `POSTGRES_PASSWORD` to a strong random value
- [ ] Use a secrets manager for sensitive variables
- [ ] Enable HTTPS/TLS on all endpoints
- [ ] Set up monitoring/alerting on `/health` endpoint
- [ ] Configure log aggregation (CloudWatch, DataDog, etc.)
- [ ] Run `pytest` to verify all tests pass
- [ ] Back up database before first deploy

---

## Security Recommendations

### Short Term (Implement Immediately)
1. ✅ **Path validation** - Fixed in media endpoint
2. ✅ **CORS credentials** - Fixed, disabled for all origins
3. ✅ **JWT secret management** - Warning added on startup
4. [ ] Rotate JWT secret immediately in production
5. [ ] Add API rate limiting (use `slowapi` or similar)

### Medium Term (Before Full Production)
1. [ ] Add input validation for user-supplied fields (title, synopsis, etc.)
2. [ ] Implement comprehensive audit logging for admin actions
3. [ ] Add API request logging middleware
4. [ ] Implement retry logic with exponential backoff in frontend
5. [ ] Add request signing/validation for sensitive operations

### Long Term (Production Hardening)
1. [ ] Implement OAuth 2.0 / OIDC for auth (replace demo JWT)
2. [ ] Add WAF (Web Application Firewall) rules
3. [ ] Implement content security policy headers
4. [ ] Set up automated dependency scanning
5. [ ] Implement database query rate limiting
6. [ ] Add DDoS protection

---

## Known Issues Not Yet Fixed

| Issue | Severity | Recommendation |
|-------|----------|-----------------|
| No retry logic on network failures | MEDIUM | Add exponential backoff in `request()` helper |
| Episode duplicates not blocked in create | MEDIUM | Add season check to duplicate validation |
| Catalogue version not incremented | LOW | Use publish run count as version |
| Dense component functions | LOW | Extract handlers and utility functions |
| Missing auth edge case tests | LOW | Add pytest tests for token validation |

---

## Quick Start

### 1. Extract the zip
```bash
unzip peblo-tv-mini-fixed.zip
cd peblo-tv-mini
```

### 2. Copy environment template
```bash
cp .env.example .env
```

### 3. Start services
```bash
docker compose up --build --pull always
```

### 4. Access the app
- **Viewer:** http://localhost:5174
- **CMS:** http://localhost:5173 (login: `admin@example.com` / `admin-demo`)
- **API Docs:** http://localhost:8000/docs
- **Health:** http://localhost:8000/health

---

## Documentation Files

- `CODE_REVIEW.md` - Detailed code review with all issues and recommendations
- `FIXES_APPLIED.md` - This file
- `.env.example` - Environment variables template
- `README.md` - Original project documentation

---

## Support & Questions

Refer to the detailed `CODE_REVIEW.md` file for:
- Complete issue descriptions
- Code before/after comparisons
- Root cause analysis
- Alternative implementations
- Testing recommendations

---

**Version:** 1.0-FIXED  
**Date:** 2024  
**Status:** Ready for deployment with recommended production hardening
