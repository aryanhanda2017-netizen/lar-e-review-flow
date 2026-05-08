# Frontend Rules — LAR-E

## ⚠️ Two Frontends — Only Touch One

| Frontend | Status | Location |
|----------|--------|----------|
| Vite + React 19 (NEW) | **Active** | `LARE-FRONTEND-main/` · GitHub: `aryanhanda2017-netizen/LARE-FRONTEND` |
| Next.js 14 / Bolt (OLD) | **DEPRECATED** | `https://lar-e-frontend-desig-f3lg.bolt.host` — do not touch |

## Stack

Vite + React 19 · TypeScript strict (`noImplicitAny`) · TanStack Query v5 · Tailwind + shadcn/ui · Axios · React Router v6.

---

## ⚠️ CRITICAL — campaign_id Must Come from `useParams()` (Bug 12)

```tsx
// CORRECT
const { campaignId } = useParams()
// WRONG — never use global state, context, or localStorage for campaign_id
```

Every campaign-scoped page must have `campaignId` in its URL: `/campaigns/:campaignId/leads`, `/campaigns/:campaignId/leads/:leadId`, etc.

**Why:** Without it, page reload drops the ID → 422 → `[object Object]` on screen.

---

## TanStack Query v5

```tsx
const { data } = useQuery({
  queryKey: ['leads', campaignId],
  queryFn: () => api.getLeads(campaignId),
  enabled: !!campaignId,          // always guard campaign-scoped queries
})

const mutation = useMutation({
  mutationFn: (data) => api.update(data),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['leads'] }),
})
```

---

## API Response Shapes — Unwrap Correctly

```tsx
// Leads:        { leads: LeadPreviewItem[], total, page, page_size }
const leads = data?.leads ?? []

// Campaigns:    array OR { campaigns: [] }
const campaigns = Array.isArray(data) ? data : (data?.campaigns ?? [])

// Appointments: { appointments: AppointmentListItem[], total, page, page_size }
const appointments = data?.appointments ?? []
```

## Dashboard Field Mapping

| UI label | Backend field |
|----------|--------------|
| Contacted | `campaign.leads_contacted` |
| Replied | `overview.replied` |
| Qualified | `overview.qualified` |
| Booked | `overview.appointments` |
| Knowledge Gaps | `knowledge_gap_count` |

---

## Auth Pattern

- Token: `localStorage` key `token`
- Session restore: `GET /api/auth/me` (not from stored user object)
- Post-login: `business_name` present → `/leads`, else → `/onboarding`
- Sign out: `localStorage.removeItem('token')` + `window.location.href = '/login'`

---

## Conversation Polling

```tsx
useQuery({
  queryKey: ['conversation', leadId],
  queryFn: () => api.getConversation(leadId),
  refetchInterval: isActive ? 15000 : false,
})
```

---

## TypeScript Rules

- `npm run build` before pushing — dev mode hides type errors that build rejects
- No `any`. Define interfaces in `src/types/` matching backend schemas.

---

## Design System

Base bg: `#EDE7DC`. See `.claude/rules/design.md` for full token set. Design prototype: `docs/design/lare-design-prototype.html`.

---

## Deployment

```bash
# 1. Commit source
git add -A && git commit -m "..." && git push origin feature/landing-page
# 2. Deploy to Vercel (required — do not skip)
git subtree push --prefix=frontend front main
```
