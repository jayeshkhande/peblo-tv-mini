# Peblo TV Mini – Code Review

## Critical Issues (Security)

### 1. Path Traversal Vulnerability in Media Endpoint ✅ FIXED
**File:** `backend/app/main.py` (line ~560)  
**Severity:** CRITICAL

**Issue:**  
The `/media/{path:path}` endpoint uses `MEDIA_ROOT / path` without validating the resolved path. An attacker can use `../` sequences to read arbitrary files outside the media directory.

**Fix Applied:**
```python
@app.get("/media/{path:path}")
def media(path: str):
    file_path = (MEDIA_ROOT / path).resolve()
    try:
        file_path.relative_to(MEDIA_ROOT.resolve())
    except ValueError:
        raise HTTPException(status_code=403, detail="Access denied.")
    if not file_path.exists() or not file_path.is_file():
        raise HTTPException(status_code=404, detail="Artwork not found.")
    return FileResponse(file_path)
```

---

### 2. CORS Configuration Too Permissive ✅ FIXED
**File:** `backend/app/main.py` (line ~538)  
**Severity:** HIGH

**Issue:**  
`allow_origins=["*"]` combined with `allow_credentials=True` is a security anti-pattern. This allows any origin to make credentialed requests, including sending cookies and auth headers cross-site.

**Fix Applied:**
```python
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_credentials=False,
                   allow_methods=["*"], allow_headers=["*"])
```

**Alternative:** Restrict to known origins:
```python
app.add_middleware(CORSMiddleware, 
    allow_origins=["http://localhost:5173", "http://localhost:5174"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"])
```

---

### 3. Weak JWT Secret in Production ⚠️ PARTIALLY FIXED
**File:** `backend/app/main.py` (line 24)  
**Severity:** HIGH

**Issue:**  
Default JWT secret is `"local-only-secret"`. If `.env` is misconfigured, the secret is publicly knowable and attackers can forge auth tokens.

**Fix Applied:**  
Added startup warning when using default secret:
```python
@app.on_event("startup")
def startup() -> None:
    if JWT_SECRET == "local-only-secret" and not os.getenv("DOCKER_CONTAINER", ""):
        import sys
        print("WARNING: JWT_SECRET is using the default local-only secret.", file=sys.stderr)
    # ...
```

**Recommendation:**  
- Add a production deployment check in Docker environment
- Use `JWT_SECRET_REQUIRED=true` to fail startup if not set
- Never commit `.env` — ensure it's in `.gitignore` (already done ✓)

---

## Medium Issues

### 4. Episode Status Not Validated in Duplicate Check
**File:** `backend/app/main.py` (line ~369)  
**Severity:** MEDIUM

**Issue:**  
The duplicate language check for content groups only applies to published episodes during validation, but the `create_episode` endpoint doesn't enforce this. You can create drafts that would later block publishing.

**Current Code:**
```python
duplicate = db.scalar(select(Episode).where(
    Episode.content_group == payload.content_group,
    Episode.language == payload.language,
    Episode.id != episode_id,
))
```

**Recommended Fix:**
```python
# Check across ALL statuses, not just published
duplicate = db.scalar(select(Episode).where(
    Episode.season_id == db.get(Season, season_id).id,
    Episode.content_group == payload.content_group,
    Episode.language == payload.language,
))
# Or add a comment explaining why draft duplicates are allowed:
# "Draft duplicates are allowed; validation_report() will catch issues at publish time"
```

---

### 5. Missing HTTP Response Models in API Docs
**File:** `backend/app/main.py`  
**Severity:** MEDIUM

**Issue:**  
Many endpoints return 404 or validation errors but don't document response shapes. FastAPI docs are unclear about error responses.

**Recommendation:**  
```python
class NotFoundResponse(BaseModel):
    detail: str

@app.get("/admin/shows/{show_id}", responses={404: {"model": NotFoundResponse}})
def get_show(show_id: int, ...):
    ...
```

---

### 6. Hardcoded Pagination in CMS Frontend
**File:** `cms/src/App.tsx` (line ~106)  
**Severity:** MEDIUM

**Issue:**  
`totalPages=2` is hardcoded with a comment about intentional demo data. If data changes or scales, pagination breaks silently.

**Fix Applied:**
```tsx
const [total, setTotal] = useState(0);
const totalPages = Math.ceil(total / 6) || 1;

async function load() {
  // ...
  const d = await request(`/admin/shows?${p}`);
  setItems(d.items);
  setTotal(d.total);  // ← capture total from API response
}
```

---

### 7. No Retry Logic on Transient Failures
**Files:** `cms/src/App.tsx`, `viewer/src/App.tsx`  
**Severity:** MEDIUM

**Issue:**  
Network requests have no retry logic. A single network hiccup causes cascading UI failures and poor UX.

**Recommendation:**  
Implement exponential backoff for safe operations (GET, HEAD):
```tsx
async function request(path: string, options: RequestInit = {}, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      const token = localStorage.getItem("peblo_token");
      const res = await fetch(`${API}${path}`, { ...options, headers: { ... } });
      const data = await res.json().catch(() => ({}));
      if (!res.ok) throw new Error(typeof data.detail === "string" ? data.detail : "Error");
      return data;
    } catch (err) {
      if (i === retries - 1) throw err;
      await new Promise(r => setTimeout(r, Math.pow(2, i) * 100));
    }
  }
}
```

---

### 8. No Error Handling in Search Filters (Viewer)
**File:** `viewer/src/App.tsx`  
**Severity:** MEDIUM

**Issue:**  
The `SearchPage` doesn't handle fetch errors. Network failures silently fail with no user feedback.

**Fix Applied:**
```tsx
const [error, setError] = useState("");

useEffect(() => {
  get(`/catalog/search?q=${encodeURIComponent(query)}&category=${category}&language=${language}`)
    .then(setData)
    .catch(e => setError(e.message));  // ← capture and display error
}, [query, category, language]);

return (
  <>
    {error ? (
      <div className="search-empty">{error}</div>
    ) : !data ? (
      <div className="search-empty">Searching…</div>
    ) : /* ... */}
  </>
);
```

---

## Low Issues (Quality & Maintainability)

### 9. Dockerfile Build Removed (Frontend) ✅ FIXED
**Files:** `cms/Dockerfile`, `viewer/Dockerfile`  
**Severity:** LOW (but improves dev experience)

**Issue:**  
The Dockerfiles ran `npm run build` during the image build, then ran `npm run dev` (development server). This wasted time and added unnecessary layers.

**Fix Applied:**  
Removed the `RUN npm run build` step. The dev server now builds on startup as intended.

---

### 10. No Healthcheck on Frontend Services
**File:** `docker-compose.yml`  
**Severity:** LOW

**Issue:**  
CMS and Viewer containers have no healthcheck. Docker Compose cannot detect failures or implement proper startup ordering.

**Fix Applied:**
```yaml
cms:
  # ...
  healthcheck:
    test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:5173"]
    interval: 10s
    timeout: 5s
    retries: 3

viewer:
  # ...
  healthcheck:
    test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:5173"]
    interval: 10s
    timeout: 5s
    retries: 3
```

---

### 11. Catalogue Version Not Incremented
**File:** `backend/app/main.py` (line ~456)  
**Severity:** LOW

**Issue:**  
`build_catalogue()` always sets `"version": 1`. After publishing, the version never changes. Viewers cannot detect catalogue updates via version checks.

**Recommendation:**
```python
def build_catalogue(db: Session) -> tuple[dict[str, Any], int, int]:
    # Get current version from file and increment
    old_catalogue = read_catalogue()
    version = old_catalogue.get("version", 0) + 1
    
    catalogue = {
        "version": version,  # ← increment, not hardcode
        "published_at": datetime.now(timezone.utc).isoformat(),
        "sections": [...]
    }
    return catalogue, len(rows), episodes_count
```

---

### 12. Overly Dense Component Functions
**Files:** `cms/src/App.tsx`, `viewer/src/App.tsx`  
**Severity:** LOW

**Issue:**  
Multi-line inline callbacks and complex logic make testing and reuse difficult.

**Example Refactor:**
```tsx
// Instead of:
async function submit(e: React.FormEvent) {
  e.preventDefault(); setError("");
  try {
    const d = await request("/auth/login", {method:"POST", body:JSON.stringify({email,password})});
    localStorage.setItem("peblo_token", d.access_token);
    localStorage.setItem("peblo_role", d.role);
    onLogin(d.role);
  } catch (err) {
    setError((err as Error).message);
  }
}

// Extract to:
async function handleLogin(email: string, password: string) {
  const data = await request("/auth/login", {
    method: "POST",
    body: JSON.stringify({ email, password })
  });
  localStorage.setItem("peblo_token", data.access_token);
  localStorage.setItem("peblo_role", data.role);
  return data.role;
}

async function submit(e: React.FormEvent) {
  e.preventDefault();
  setError("");
  try {
    const role = await handleLogin(email, password);
    onLogin(role);
  } catch (err) {
    setError((err as Error).message);
  }
}
```

---

### 13. Missing Test Coverage for Auth Edge Cases
**File:** `backend/tests/test_core.py`  
**Severity:** LOW

**Issue:**  
No tests for expired/malformed tokens, missing Authorization header, or role enforcement.

**Recommended Tests:**
```python
def test_current_user_requires_token():
    with pytest.raises(HTTPException) as exc:
        current_user(None)
    assert exc.value.status_code == 401

def test_current_user_rejects_invalid_signature():
    fake_token = "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyQGV4YW1wbGUuY29tIiwicm9sZSI6ImVkaXRvciJ9.invalid_signature"
    with pytest.raises(HTTPException) as exc:
        current_user(fake_token)
    assert exc.value.status_code == 401

def test_require_admin_blocks_editors():
    editor_user = {"email": "editor@example.com", "role": "editor"}
    with pytest.raises(HTTPException) as exc:
        require_admin(editor_user)
    assert exc.value.status_code == 403
```

---

## Summary of Fixes Applied

| Issue | Severity | Status |
|-------|----------|--------|
| Path traversal in media endpoint | CRITICAL | ✅ Fixed |
| CORS + credentials | HIGH | ✅ Fixed |
| Weak JWT secret (no startup check) | HIGH | ✅ Partial (added warning) |
| Duplicate check for drafts | MEDIUM | ⏳ Needs review |
| Missing API response models | MEDIUM | ⏳ Optional |
| Hardcoded pagination | MEDIUM | ✅ Fixed |
| No retry logic | MEDIUM | ⏳ Optional improvement |
| Search error handling | MEDIUM | ✅ Fixed |
| Frontend build in Dockerfile | LOW | ✅ Fixed |
| Missing healthchecks | LOW | ✅ Fixed |
| Catalogue version immutable | LOW | ⏳ Optional enhancement |
| Dense component functions | LOW | ⏳ Code quality (optional) |
| Missing auth tests | LOW | ⏳ Test coverage (optional) |

---

## Verification

All services are running:
- ✅ Database: `postgres:16-alpine` (healthy)
- ✅ API: `http://localhost:8000` (responds with health check)
- ✅ CMS: `http://localhost:5173` (dev server)
- ✅ Viewer: `http://localhost:5174` (dev server)

Path traversal protection is active (relative_to check prevents escaping MEDIA_ROOT).  
CORS credentials disabled to prevent cross-origin auth leaks.  
Frontend error handling in SearchPage added.
