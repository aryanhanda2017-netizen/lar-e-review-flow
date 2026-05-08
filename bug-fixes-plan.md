# LAR-E Bug Fixes — All 10 Codex Review Issues

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix all 10 bugs surfaced by the Codex adversarial review — 4 backend (data integrity / reliability) + 6 frontend (correctness / UX safety).

**Architecture:** Backend fixes are surgical changes to 3 files + 1 new Alembic migration. Frontend fixes are isolated to 5 files. No new abstractions, no feature additions.

**Tech Stack:** Python 3.12 / FastAPI / SQLAlchemy 2.0 async / PostgreSQL / arq · React 19 / TypeScript strict / TanStack Query v5

**Working directories:**
- Backend: `Code — Backend/lare-backend-documents/`
- Frontend: `/tmp/lare-review/frontend/` (clone of `github.com/aryanhanda2017-netizen/LARE-FRONTEND`, branch `main`)

---

## Context

A Codex adversarial review found 4 backend bugs (1 critical tenant-isolation breach, 3 high-severity reliability gaps) and 6 frontend bugs (data loss, silent key mismatches, idempotency failures). These must be fixed before onboarding any pilot client. All changes are backward-compatible except one new migration that adds a `sending` enum value to PostgreSQL.

---

## Files Modified

### Backend (`Code — Backend/lare-backend-documents/`)
| File | Change |
|------|--------|
| `app/api/webhooks.py` | Scope phone lookup to active conversations first; re-raise processing exceptions |
| `app/api/campaigns.py` | Enqueue Redis job before committing ACTIVE state |
| `app/services/outreach.py` | `FOR UPDATE SKIP LOCKED` claim + SENDING status + retry failed leads |
| `app/models/lead.py` | Add `SENDING` to `OutreachStatus` enum |
| `alembic/versions/20260509_0007_outreach_sending_status.py` | New migration: add `sending` to DB enum |

### Frontend (`/tmp/lare-review/frontend/src/`)
| File | Change |
|------|--------|
| `pages/NewCampaignPage.tsx` | Fix tier key map; lock campaign name post-upload |
| `pages/OnboardingPage.tsx` | Track per-file upload success; skip on retry |
| `pages/SettingsPage.tsx` | Guard form hydration; block save on failed fetch |
| `hooks/useLeads.ts` | Replace page_size:1000 with parameterized pagination |
| `pages/LoginPage.tsx` | Only persist token after /api/auth/me succeeds |
| `pages/SignupPage.tsx` | Remove redundant localStorage write before login() |
| `contexts/AuthContext.tsx` | Move token persistence inside login(); clear on bootstrap failure |

---

## Task 1: Add `SENDING` enum value + migration (Backend)

**Files:**
- Modify: `Code — Backend/lare-backend-documents/app/models/lead.py`
- Create: `Code — Backend/lare-backend-documents/alembic/versions/20260509_0007_outreach_sending_status.py`

- [ ] **Step 1: Add `SENDING` to `OutreachStatus` in the model**

In `app/models/lead.py`, add `SENDING` after `PENDING`:
```python
class OutreachStatus(str, enum.Enum):
    PENDING = "pending"
    SENDING = "sending"   # claimed by outreach worker, BSP call in-flight
    SENT = "sent"
    DELIVERED = "delivered"
    REPLIED = "replied"
    QUALIFIED = "qualified"
    BOOKING_LINK_SHARED = "booking_link_shared"
    BOOKING_CONFIRMED = "booking_confirmed"
    BOOKING_UNCONFIRMED = "booking_unconfirmed"
    DECLINED = "declined"
    OPTED_OUT = "opted_out"
    INVALID = "invalid"
```

- [ ] **Step 2: Create the migration file**

Create `alembic/versions/20260509_0007_outreach_sending_status.py`:
```python
"""add sending value to outreach_status_enum

Revision ID: 20260509_0007
Revises: 20260429_0006
Create Date: 2026-05-09 00:00:00.000000
"""
from __future__ import annotations
from alembic import op

revision = "20260509_0007"
down_revision = "20260429_0006"
branch_labels = None
depends_on = None


def upgrade() -> None:
    # PostgreSQL allows adding enum values but not removing them.
    # IF NOT EXISTS prevents failure on re-run.
    op.execute("ALTER TYPE outreach_status_enum ADD VALUE IF NOT EXISTS 'sending' AFTER 'pending'")


def downgrade() -> None:
    # Cannot remove enum values in PostgreSQL without recreating the type.
    # Leave as-is; the value is harmless if unused.
    pass
```

- [ ] **Step 3: Verify migration applies cleanly**

```bash
cd "Code — Backend/lare-backend-documents"
alembic upgrade head
```
Expected output: `Running upgrade 20260429_0006 -> 20260509_0007, add sending value to outreach_status_enum`

- [ ] **Step 4: Run smoke tests**

```bash
cd "Code — Backend/lare-backend-documents"
python3 -m pytest smoke_test.py -v --tb=short
```
Expected: All 63 tests pass (new enum value doesn't affect existing tests).

- [ ] **Step 5: Commit**

```bash
cd "Code — Backend/lare-backend-documents"
git add app/models/lead.py alembic/versions/20260509_0007_outreach_sending_status.py
git commit -m "fix: add SENDING state to OutreachStatus enum with migration"
```

---

## Task 2: Fix outreach double-send with atomic lead claiming (Backend)

> **Depends on Task 1.** `outreach.py` references `OutreachStatus.SENDING` — that enum value must exist in the model and the database before this task is started. Do not begin Task 2 until Task 1's migration has been applied.

**Files:**
- Modify: `Code — Backend/lare-backend-documents/app/services/outreach.py`

This change makes `get_next_batch` atomically claim leads via `FOR UPDATE SKIP LOCKED` + immediate status flip to `SENDING`. Concurrent workers skip locked rows. Failed sends revert to `PENDING` unless max attempts reached.

- [ ] **Step 1: Update `get_next_batch` to claim leads atomically**

Replace the existing `get_next_batch` function (lines ~95–120):
```python
async def get_next_batch(
    campaign_id: uuid.UUID,
    client_id: uuid.UUID,
    batch_size: int,
    db: AsyncSession,
) -> list[Lead]:
    """
    Fetch and atomically claim the next batch of pending leads.
    Uses FOR UPDATE SKIP LOCKED so concurrent workers cannot pick the same rows.
    """
    tier_order = case(
        (Lead.priority_tier == PriorityTier.A, 0),
        (Lead.priority_tier == PriorityTier.B, 1),
        (Lead.priority_tier == PriorityTier.C, 2),
        else_=3,
    )
    result = await db.execute(
        select(Lead)
        .where(
            Lead.campaign_id == campaign_id,
            Lead.client_id == client_id,
            Lead.outreach_status == OutreachStatus.PENDING,
        )
        .order_by(tier_order.asc(), Lead.created_at.asc())
        .limit(batch_size)
        .with_for_update(skip_locked=True)
    )
    leads = list(result.scalars().all())
    # Claim immediately — flush writes SENDING to DB while the transaction
    # holds the row locks, preventing any second worker from reading these rows.
    for lead in leads:
        lead.outreach_status = OutreachStatus.SENDING
    if leads:
        await db.flush()
    return leads
```

- [ ] **Step 2: Update failure handling in `send_batch` to revert unclaimed leads**

In `send_batch`, find the failure branch (currently `lead.send_attempts += 1; if lead.send_attempts >= 3: lead.outreach_status = OutreachStatus.INVALID`). Replace it with:
```python
        lead.send_attempts += 1
        if lead.send_attempts >= 3:
            lead.outreach_status = OutreachStatus.INVALID
        else:
            lead.outreach_status = OutreachStatus.PENDING  # back in queue for retry
        failed_count += 1
        logger.warning(
            "Failed to send WhatsApp message to %s for campaign %s: %s",
            lead.phone,
            campaign_id,
            send_result.error,
        )
```

- [ ] **Step 3: Update the `remaining` count query to exclude SENDING leads**

In `send_batch`, the `remaining_result` query counts `PENDING` leads. SENDING leads are in-flight in this same batch — they'll be SENT or back to PENDING by commit time. The count is taken after commit so SENDING leads have already transitioned. No change needed here — verify by reading the count query:
```python
    remaining_result = await db.execute(
        select(func.count())
        .select_from(Lead)
        .where(
            Lead.campaign_id == campaign_id,
            Lead.client_id == client_id,
            Lead.outreach_status == OutreachStatus.PENDING,
        )
    )
```
This is correct as-is (SENDING leads are resolved before this count runs).

- [ ] **Step 4: Run smoke tests**

```bash
cd "Code — Backend/lare-backend-documents"
python3 -m pytest smoke_test.py -v --tb=short
```
Expected: All 63 tests pass.

- [ ] **Step 5: Commit**

```bash
cd "Code — Backend/lare-backend-documents"
git add app/services/outreach.py
git commit -m "fix: claim leads atomically with FOR UPDATE SKIP LOCKED to prevent double-send"
```

---

## Task 3: Fix webhook phone lookup — prefer active conversations, log ambiguity (Backend)

**Files:**
- Modify: `Code — Backend/lare-backend-documents/app/api/webhooks.py`

- [ ] **Step 1: Add missing imports to webhooks.py**

At the top of `app/api/webhooks.py`, ensure these imports are present (add if missing):
```python
from app.models.conversation import Conversation, ConversationStatus
```

- [ ] **Step 2: Replace `_find_recent_lead_by_phone` with `_find_lead_by_phone`**

Remove the old `_find_recent_lead_by_phone` function and replace with:
```python
async def _find_lead_by_phone(phone: str, db: AsyncSession) -> Lead | None:
    """
    Find the best-matching active lead for an inbound phone number.

    Priority order:
    1. Lead that already has an active conversation (most specific — we were talking to them).
    2. Most recently updated active lead (fallback). Logs a warning if ambiguous.
    """
    # Prefer a lead with an ongoing active conversation
    conv_result = await db.execute(
        select(Lead)
        .join(Campaign, Lead.campaign_id == Campaign.id)
        .join(Conversation, Conversation.lead_id == Lead.id)
        .where(
            Lead.phone == phone,
            Lead.outreach_status != OutreachStatus.INVALID,
            Campaign.status == CampaignStatus.ACTIVE,
            Conversation.status == ConversationStatus.ACTIVE,
        )
        .order_by(Lead.updated_at.desc())
        .limit(1)
    )
    lead = conv_result.scalars().first()
    if lead is not None:
        return lead

    # Fallback: most recently updated active lead
    result = await db.execute(
        select(Lead)
        .join(Campaign, Lead.campaign_id == Campaign.id)
        .where(
            Lead.phone == phone,
            Lead.outreach_status != OutreachStatus.INVALID,
            Campaign.status == CampaignStatus.ACTIVE,
        )
        .order_by(Lead.updated_at.desc())
        .limit(2)
    )
    leads = list(result.scalars().all())
    if not leads:
        return None
    if len(leads) > 1:
        logger.warning(
            "Ambiguous inbound: phone %s matches %d active leads; routing to lead_id=%s. "
            "Consider adding a BSP conversation ID to disambiguate.",
            phone,
            len(leads),
            leads[0].id,
        )
    return leads[0]
```

- [ ] **Step 3: Update the call site in `whatsapp_webhook`**

In `whatsapp_webhook`, replace `await _find_recent_lead_by_phone(normalized_phone, db)` with:
```python
            lead = await _find_lead_by_phone(normalized_phone, db)
```

- [ ] **Step 4: Run smoke tests**

```bash
cd "Code — Backend/lare-backend-documents"
python3 -m pytest smoke_test.py -v --tb=short
```
Expected: All 63 tests pass.

- [ ] **Step 5: Commit**

```bash
cd "Code — Backend/lare-backend-documents"
git add app/api/webhooks.py
git commit -m "fix: scope webhook phone lookup to active conversations first, log ambiguity"
```

---

## Task 4: Fix webhook silent failure — re-raise processing errors (Backend)

**Files:**
- Modify: `Code — Backend/lare-backend-documents/app/api/webhooks.py`

The broad `except Exception` swallows all failures and returns 200, causing permanent message loss. The idempotency check (`bsp_message_id` dedup) makes re-raising safe — BSP retries will not double-process.

- [ ] **Step 1: Change the exception handler to re-raise**

In `whatsapp_webhook`, find the `except Exception` block at the end of the processing try/except (currently lines ~95–97):
```python
    except Exception:
        logger.exception("Unhandled WhatsApp webhook processing error")
        return {"status": "ok"}
```

Replace with:
```python
    except Exception:
        logger.exception("Unhandled WhatsApp webhook processing error")
        raise
```

The `raise` causes FastAPI to return a 500, which prompts the BSP to retry delivery. The idempotency check at the top of inbound processing (`bsp_message_id` lookup) prevents double-processing on retry.

- [ ] **Step 2: Run smoke tests**

```bash
cd "Code — Backend/lare-backend-documents"
python3 -m pytest smoke_test.py -v --tb=short
```
Expected: `test_webhook_idempotency_check_present` passes (idempotency guard still in place).

- [ ] **Step 3: Commit**

```bash
cd "Code — Backend/lare-backend-documents"
git add app/api/webhooks.py
git commit -m "fix: re-raise webhook processing exceptions so BSP retries on failure"
```

---

## Task 5: Fix campaign launch atomicity — enqueue before committing ACTIVE state (Backend)

**Files:**
- Modify: `Code — Backend/lare-backend-documents/app/api/campaigns.py`

- [ ] **Step 1: Restructure `launch_campaign` to enqueue first**

Replace the `launch_campaign` endpoint body. Find these lines:
```python
    campaign.status = CampaignStatus.ACTIVE
    campaign.started_at = datetime.now(timezone.utc)
    await db.commit()
    await db.refresh(campaign)
    await advance_onboarding_status(current_client, OnboardingStatus.LAUNCHED.value, db)
    await enqueue_outreach_batch(str(campaign.id), str(current_client.id))
```

Replace with:
```python
    # Enqueue BEFORE committing state. If Redis is unavailable, this raises and
    # the campaign stays in DRAFT — the caller gets a 503 and can retry safely.
    try:
        await enqueue_outreach_batch(str(campaign.id), str(current_client.id))
    except Exception as exc:
        logger.exception("Failed to enqueue outreach batch for campaign %s", campaign.id)
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="Outreach scheduling is unavailable. Please try launching again.",
        ) from exc

    campaign.status = CampaignStatus.ACTIVE
    campaign.started_at = datetime.now(timezone.utc)
    await db.commit()
    await db.refresh(campaign)
    await advance_onboarding_status(current_client, OnboardingStatus.LAUNCHED.value, db)
```

You'll also need `import logging` and `logger = logging.getLogger(__name__)` at the top of `campaigns.py` if not already present:
```python
import logging
logger = logging.getLogger(__name__)
```

- [ ] **Step 2: Run smoke tests**

```bash
cd "Code — Backend/lare-backend-documents"
python3 -m pytest smoke_test.py -v --tb=short
```
Expected: All 63 tests pass.

- [ ] **Step 3: Commit**

```bash
cd "Code — Backend/lare-backend-documents"
git add app/api/campaigns.py
git commit -m "fix: enqueue outreach job before committing ACTIVE state to prevent stuck campaigns"
```

---

## Task 6: Fix tier key mismatch + lock campaign name post-upload (Frontend)

**Files:**
- Modify: `/tmp/lare-review/frontend/src/pages/NewCampaignPage.tsx`

The backend returns `tier_a / tier_b / tier_c` but the UI reads `A / B / C`. Also, the campaign name field is editable after upload but is never saved — lock it to read-only after step 1.

- [ ] **Step 1: Add a tier key mapping near the top of the component**

Find the top of `NewCampaignPage.tsx` (after imports). Add this constant before the component function:
```typescript
const TIER_KEY: Record<string, 'tier_a' | 'tier_b' | 'tier_c'> = {
  A: 'tier_a',
  B: 'tier_b',
  C: 'tier_c',
}
```

- [ ] **Step 2: Replace all tier breakdown accesses**

Search the file for patterns like `tier_breakdown[tier]`, `tier_breakdown.A`, `tier_breakdown.B`, `tier_breakdown.C`, `tier_breakdown['A']`, etc. Replace every instance with:
```typescript
// Instead of: uploadResult.tier_breakdown[tier]  or  tier_breakdown.A
uploadResult.tier_breakdown[TIER_KEY[tier]]

// Instead of: tier_breakdown.A  (hardcoded)
uploadResult.tier_breakdown.tier_a

// Instead of: tier_breakdown.B
uploadResult.tier_breakdown.tier_b

// Instead of: tier_breakdown.C
uploadResult.tier_breakdown.tier_c
```

- [ ] **Step 3: Make campaign name read-only after upload**

Find the campaign name input field that is shown during the preview/launch steps (step >= 2). The input should already exist for step 1 (upload). For steps 2 and 3, change the input to a static display:

Find the JSX for the campaign name field. It will look something like:
```tsx
<input
  value={campaignName}
  onChange={(e) => setCampaignName(e.target.value)}
  ...
/>
```

Wrap it with a conditional based on `currentStep`:
```tsx
{currentStep === 1 ? (
  <input
    value={campaignName}
    onChange={(e) => setCampaignName(e.target.value)}
    // ... existing props
  />
) : (
  <p className="/* existing name text styles */">{campaignName}</p>
)}
```

Match whatever CSS classes and styles are already used for text display in the component (check nearby text elements for the pattern).

- [ ] **Step 4: Run TypeScript build check**

```bash
cd /tmp/lare-review/frontend
npm install
npm run build
```
Expected: Build succeeds with no TypeScript errors.

- [ ] **Step 5: Commit and push**

```bash
cd /tmp/lare-review/frontend
git add src/pages/NewCampaignPage.tsx
git commit -m "fix: correct tier_breakdown key mapping (tier_a/b/c) and lock name post-upload"
git push origin main
```

---

## Task 7: Fix onboarding document upload idempotency (Frontend)

**Files:**
- Modify: `/tmp/lare-review/frontend/src/pages/OnboardingPage.tsx`

Retrying the documents step re-posts all files including already-uploaded ones. Fix: track which entries uploaded successfully; skip them on retry.

- [ ] **Step 1: Add `uploadedKeys` state to the onboarding component**

Find the component's state declarations (the `useState` calls near the top). Add:
```typescript
const [uploadedKeys, setUploadedKeys] = useState<Set<string>>(new Set())
```

- [ ] **Step 2: Build a stable key for each document entry**

Each document entry has a `type` and a `file`. The key uniquely identifies the entry:
```typescript
// Helper — add near the state declarations or at the top of the component
const entryKey = (type: string, fileName: string) => `${type}__${fileName}`
```

- [ ] **Step 3: Skip already-uploaded entries in the upload loop**

Find the document upload loop (around lines 74–83 in `OnboardingPage.tsx`). It will look something like:
```typescript
for (const entry of documentEntries) {
  await api.uploadDocument(entry.type, entry.file)
}
```

Replace with:
```typescript
for (const entry of documentEntries) {
  const key = entryKey(entry.type, entry.file.name)
  if (uploadedKeys.has(key)) {
    continue  // already uploaded successfully; skip on retry
  }
  await api.uploadDocument(entry.type, entry.file)
  setUploadedKeys(prev => new Set([...prev, key]))
}
```

- [ ] **Step 4: Reset `uploadedKeys` when the user changes files**

Find where document entries are added/removed (the file input change handler). After clearing or changing entries, also reset the uploaded set:
```typescript
setUploadedKeys(new Set())
```

- [ ] **Step 5: Run TypeScript build check**

```bash
cd /tmp/lare-review/frontend
npm run build
```
Expected: Build succeeds.

- [ ] **Step 6: Commit and push**

```bash
cd /tmp/lare-review/frontend
git add src/pages/OnboardingPage.tsx
git commit -m "fix: skip already-uploaded documents on retry in onboarding step 2"
git push origin main
```

---

## Task 8: Fix settings form data loss on slow or failed fetch (Frontend)

**Files:**
- Modify: `/tmp/lare-review/frontend/src/pages/SettingsPage.tsx`

The form hydrates from the API response but overwrites in-progress edits on slow loads, and allows saving blank data on failed fetches.

- [ ] **Step 1: Add `hasHydrated` state to the BusinessContextSection component**

Find the `BusinessContextSection` component (or wherever the business context form state is managed). Add:
```typescript
const [hasHydrated, setHasHydrated] = useState(false)
```

- [ ] **Step 2: Hydrate form state only once, while pristine**

Find the `useEffect` that populates the form from the API response. It will look like:
```typescript
useEffect(() => {
  if (existing) {
    setForm(existing)  // or setForm({ ...existing })
  }
}, [existing])
```

Replace with:
```typescript
useEffect(() => {
  if (existing && !hasHydrated) {
    setForm({
      description: existing.description ?? '',
      primary_product_name: existing.primary_product_name ?? '',
      pricing_overview: existing.pricing_overview ?? '',
      usp: existing.usp ?? '',
      current_offers: existing.current_offers ?? '',
      // include all fields that exist on the form
    })
    setHasHydrated(true)
  }
}, [existing, hasHydrated])
```

Adjust the field names to match what `existing` actually contains (check the `BusinessContextResponse` type in `src/types/api.ts`).

- [ ] **Step 3: Disable the form and save button until loaded**

Find the TanStack Query call that fetches the existing business context. It returns `{ data: existing, isLoading, isError }`. Update the form's JSX to:

```tsx
{isLoading && (
  <p className="/* use existing muted text class */">Loading your settings…</p>
)}
{isError && (
  <p className="/* use existing error text class */">
    Failed to load settings. Refresh to try again.
  </p>
)}
{/* Wrap the entire form in: */}
<fieldset disabled={isLoading || isError || !hasHydrated} style={{ border: 'none', padding: 0 }}>
  {/* existing form JSX */}
</fieldset>
```

- [ ] **Step 4: Run TypeScript build check**

```bash
cd /tmp/lare-review/frontend
npm run build
```
Expected: Build succeeds.

- [ ] **Step 5: Commit and push**

```bash
cd /tmp/lare-review/frontend
git add src/pages/SettingsPage.tsx
git commit -m "fix: guard settings form hydration to prevent data loss on slow or failed fetch"
git push origin main
```

---

## Task 9: Fix lead pagination — replace hard-coded page_size:1000 (Frontend)

**Files:**
- Modify: `/tmp/lare-review/frontend/src/hooks/useLeads.ts`
- Modify: `/tmp/lare-review/frontend/src/pages/CampaignDetailPage.tsx`

The hard-coded `page_size: 1000` silently truncates campaigns with more than 1,000 leads and presents a subset as complete.

- [ ] **Step 1: Update `useLeads` to accept page and pageSize parameters**

Replace the entire `useLeads` hook in `src/hooks/useLeads.ts`:
```typescript
export function useLeads(
  campaignId: string | undefined,
  page = 1,
  pageSize = 50,
  isActive = false,
) {
  return useQuery({
    queryKey: ['leads', campaignId, page, pageSize],
    queryFn: () => api.getLeads(campaignId!, { page, page_size: pageSize }),
    enabled: !!campaignId,
    refetchInterval: isActive ? 30_000 : false,
  })
}
```

Check `src/lib/api.ts` to see how `getLeads` is called and update the call signature there if it doesn't already accept `page` and `page_size` params:
```typescript
// In api.ts, update getLeads to accept pagination params:
getLeads: (campaignId: string, params: { page?: number; page_size?: number } = {}) =>
  axiosInstance
    .get<LeadsListResponse>(`/api/leads/`, { params: { campaign_id: campaignId, ...params } })
    .then(r => r.data),
```

- [ ] **Step 2: Add `page` state and pagination controls to CampaignDetailPage**

Find the leads section in `CampaignDetailPage.tsx`. It currently uses `useLeads(campaignId)`. Update it:

```typescript
const [leadsPage, setLeadsPage] = useState(1)
const PAGE_SIZE = 50

const { data: leadsData } = useLeads(campaignId, leadsPage, PAGE_SIZE)
const leads = leadsData?.leads ?? []
const totalLeads = leadsData?.total ?? 0
const hasMore = leadsPage * PAGE_SIZE < totalLeads
```

- [ ] **Step 3: Add prev/next pagination UI below the leads list**

Use explicit page controls — do NOT use "load more" (it silently replaces the current page's data with the next page instead of accumulating, because `useQuery` overwrites `data` on each fetch):

```tsx
<div style={{ display: 'flex', alignItems: 'center', gap: 16, marginTop: 16 }}>
  <button
    className="btn-ghost btn-sm"
    onClick={() => setLeadsPage(p => Math.max(1, p - 1))}
    disabled={leadsPage === 1}
  >
    ← Previous
  </button>
  <span className="t-label t-muted">
    Page {leadsPage} · {totalLeads} leads total
  </span>
  <button
    className="btn-secondary btn-sm"
    onClick={() => setLeadsPage(p => p + 1)}
    disabled={!hasMore}
  >
    Next →
  </button>
</div>
```

Adjust class names to match the design system in use (check existing buttons in the file for exact class names).

- [ ] **Step 4: Run TypeScript build check**

```bash
cd /tmp/lare-review/frontend
npm run build
```
Expected: Build succeeds with no type errors.

- [ ] **Step 5: Commit and push**

```bash
cd /tmp/lare-review/frontend
git add src/hooks/useLeads.ts src/pages/CampaignDetailPage.tsx src/lib/api.ts
git commit -m "fix: replace hard-coded page_size:1000 with real pagination (50/page)"
git push origin main
```

---

## Task 10: Fix auth token race — persist token only after /api/auth/me succeeds (Frontend)

**Files:**
- Modify: `/tmp/lare-review/frontend/src/pages/LoginPage.tsx`
- Modify: `/tmp/lare-review/frontend/src/pages/SignupPage.tsx`
- Modify: `/tmp/lare-review/frontend/src/contexts/AuthContext.tsx`

Token is stored in localStorage before `/api/auth/me` succeeds. If `me` fails, a stale token is left behind, creating half-authenticated state on next load.

- [ ] **Step 1: Read AuthContext.tsx to understand the login flow**

Read `/tmp/lare-review/frontend/src/contexts/AuthContext.tsx` in full before making changes. Identify:
- Where `setToken` / `localStorage.setItem('token', ...)` is called
- Where `/api/auth/me` is called
- The `login(token)` function signature

- [ ] **Step 2: Move token persistence inside AuthContext's `login` function**

In `AuthContext.tsx`, the `login` function should:
1. Accept the raw token
2. Call `/api/auth/me` using that token
3. **Only if `/api/auth/me` succeeds**: store the token to localStorage and update auth state
4. **If `/api/auth/me` fails**: throw the error without storing the token

The function should look like:
```typescript
const login = async (token: string) => {
  // Validate the token against the server BEFORE persisting
  const meResponse = await api.getMe(token)  // pass token explicitly or set temporarily
  // Only store after server confirms the token is valid
  localStorage.setItem('token', token)
  setUser(meResponse)
  setIsAuthenticated(true)
}
```

The exact implementation depends on how `api.getMe` works. If it reads from localStorage, temporarily set it, call `getMe`, and clear on failure:
```typescript
const login = async (token: string) => {
  localStorage.setItem('token', token)  // temp — needed for the /me call
  try {
    const meResponse = await api.getMe()
    setUser(meResponse)
    setIsAuthenticated(true)
    // token stays in localStorage — login succeeded
  } catch (err) {
    localStorage.removeItem('token')    // clear on failure
    setIsAuthenticated(false)
    throw err
  }
}
```

- [ ] **Step 3: Remove redundant localStorage calls from LoginPage and SignupPage**

In `LoginPage.tsx`, find any `localStorage.setItem('token', ...)` or `setToken(data.access_token)` calls that happen **before** `login()` is called. Remove them — `login()` now handles storage internally.

Do the same in `SignupPage.tsx`.

- [ ] **Step 4: Fix AuthContext bootstrap to clear token on non-401 failures**

Find the useEffect in `AuthContext.tsx` that runs on mount and calls `/api/auth/me` to restore session. Find the error handler:
```typescript
} catch (err) {
  // current code: may preserve token on non-401 errors
}
```

Update it to always clear the token on any bootstrap failure:
```typescript
} catch {
  localStorage.removeItem('token')
  setIsAuthenticated(false)
  setUser(null)
}
```

- [ ] **Step 5: Run TypeScript build check**

```bash
cd /tmp/lare-review/frontend
npm run build
```
Expected: Build succeeds.

- [ ] **Step 6: Commit and push**

```bash
cd /tmp/lare-review/frontend
git add src/contexts/AuthContext.tsx src/pages/LoginPage.tsx src/pages/SignupPage.tsx
git commit -m "fix: persist auth token only after /api/auth/me succeeds; clear on bootstrap failure"
git push origin main
```

---

## Final Verification

- [ ] **Backend: Run full smoke test suite**

```bash
cd "Code — Backend/lare-backend-documents"
python3 -m pytest smoke_test.py -v --tb=short
```
Expected: 63/63 pass.

- [ ] **Backend: Confirm migration chain is clean**

```bash
cd "Code — Backend/lare-backend-documents"
alembic check
```
Expected: `No new upgrade operations detected.`

- [ ] **Frontend: Final build**

```bash
cd /tmp/lare-review/frontend
npm run build
```
Expected: Build succeeds, no TypeScript errors, no warnings about undefined properties.

- [ ] **Backend: Push to Railway repo**

```bash
cd "Code — Backend/lare-backend-documents"
git push origin main
```
Railway will run `alembic upgrade head` automatically on deploy.

---

## Issue Priority Summary

| # | Severity | File | Fixed in Task |
|---|----------|------|---------------|
| 1 | Critical | `webhooks.py` — tenant isolation | Task 3 |
| 2 | High | `webhooks.py` — silent failure | Task 4 |
| 3 | High | `campaigns.py` — launch race | Task 5 |
| 4 | High | `outreach.py` — double-send | Tasks 1+2 |
| 5 | High | `NewCampaignPage.tsx` — tier keys | Task 6 |
| 6 | High | `NewCampaignPage.tsx` — rename lost | Task 6 |
| 7 | High | `OnboardingPage.tsx` — dup uploads | Task 7 |
| 8 | High | `SettingsPage.tsx` — data loss | Task 8 |
| 9 | High | `useLeads.ts` — 1k truncation | Task 9 |
| 10 | Medium | `LoginPage.tsx` — token race | Task 10 |
