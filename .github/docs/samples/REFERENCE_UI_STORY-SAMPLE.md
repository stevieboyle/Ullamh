# Reference UI Story Sample: KAN-4-102
## Title: [UI] Implementation of Async Job Status Component with Polling

> **This is a sample document.** It contains project-specific content from the **Meitheal (KAN-4)** project and is provided as a concrete reference showing the level of detail expected in a refined frontend/UI story. Do **not** use this as a starting point for new stories. Use the generic template at `.github/skills/refinement-checklist/assets/reference-ui-story.md` instead.

---

### 1. User Narrative
> **As a** Researcher,
> **I want** to see a real-time progress bar and status indicator after I upload a paper,
> **so that** I know exactly which stage of extraction (Text, Metadata, or Citations) the system is currently working on without refreshing the page.

---

### 2. Technical Context (The "Agent's Map")
* **Framework:** Next.js 16 (App Router).
* **State Management:** TanStack Query v6 (React Query) for status polling.
* **Relevant Spec:** `KAN-4-architecture-specification.md` (Section 5: Pipeline Stages).
* **Key Files to Create/Modify:**
    * `src/components/upload/JobStatus.tsx` (Main Client Component)
    * `src/hooks/useJobStatus.ts` (Custom hook for polling logic)
    * `src/app/api/proxy/jobs/[id]/route.ts` (Next.js API Route to proxy FastAPI requests)

---

### 3. Acceptance Criteria (AC)
- [ ] **Polling Logic:** Implement `useQuery` with a `refetchInterval` of 3000ms.
- [ ] **Auto-Termination:** Polling must stop automatically when the job status is `COMPLETED` or `FAILED`.
- [ ] **Visual Stepper:** Map the Backend Enums (`PENDING`, `EXTRACTING`, `RESOLVING`, `COMPLETED`) to a Tailwind-based stepper component.
- [ ] **Error States:** Display a clear error message and a "Retry" button if the job status returns `FAILED`.
- [ ] **Suspense Integration:** The component must be wrapped in a `<Suspense>` boundary with a skeleton loader in the parent page.

---

### 4. Definition of Done (DoD) for Agents
* **Component Pattern:** Use **Client Components** only for the status "leaf" nodes to keep the parent pages as **Server Components**.
* **Accessibility:** The progress bar must include `aria-valuenow`, `aria-valuemin`, and `aria-valuemax` attributes.
* **Clean Code:** Use TypeScript interfaces for the API response; no `any` types.
* **Cleanup:** Ensure the `refetchInterval` is cleared when the component unmounts to prevent memory leaks.

---

### 5. Edge Cases & Constraints
* **Slow Network:** Handle cases where the status request takes longer than the polling interval (use `refetchOnWindowFocus: false`).
* **Invalid ID:** Show a "Job Not Found" state if the job ID in the URL is malformed or missing from the DB.

---

### 6. Alignment with KAN-4 Spec
* **Backend Sync:** Ensure the polling endpoint matches the FastAPI route defined in the backend service.
* **Stage Parity:** The frontend stepper labels must exactly match the pipeline stages defined in Section 5 of the architecture spec.

---

### 7. Cross-Cutting Concerns
* **CORS/Auth impact:** N/A — status polling is proxied via the Next.js API route; no direct cross-origin call from the browser to FastAPI.
* **Rate-limiting impact:** Polling at 3000ms intervals per active job. If many concurrent users poll simultaneously, monitor the FastAPI `/api/jobs/{id}` rate-limiter thresholds.
* **Logging/Observability:** Log on the proxy route: request received, upstream status code, upstream errors.
* **Error propagation:** A 500 from FastAPI is caught by TanStack Query's `onError` handler and surfaced to the user as a generic "Processing error — please try again" message.
