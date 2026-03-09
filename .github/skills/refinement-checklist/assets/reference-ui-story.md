# Reference UI Story: [EPIC_ID]-[STORY-NUMBER]
## Title: [UI] Implementation of Async Job Status Component with Polling

> **This is a generic template.** Replace all `[PLACEHOLDER]` values with your project's domain entities, framework names, and spec section references before use. See `.github/docs/samples/REFERENCE_UI_STORY-SAMPLE.md` for a fully-populated example.

---

### 1. User Narrative
> **As a** [USER_PERSONA],
> **I want** to see a real-time progress indicator after I submit a [CORE_DOMAIN_ENTITY],
> **so that** I know exactly which stage of processing the system is currently working on without refreshing the page.

---

### 2. Technical Context (The "Agent's Map")
* **Framework:** [FRONTEND_FRAMEWORK] ([routing approach, e.g. App Router]).
* **State Management:** [STATE_MANAGEMENT_LIBRARY] for status polling.
* **Relevant Spec:** `[EPIC_ID]-architecture-specification.md` (Section 5: Pipeline Stages).
* **Key Files to Create/Modify:**
    * `[COMPONENTS_DIR]/[domain]/JobStatus.tsx` (Main Client Component)
    * `[HOOKS_DIR]/useJobStatus.ts` (Custom hook for polling logic)
    * `[API_PROXY_DIR]/jobs/[id]/route.ts` ([FRONTEND_FRAMEWORK] API Route to proxy [BACKEND_FRAMEWORK] requests)

---

### 3. Acceptance Criteria (AC)
- [ ] **Polling Logic:** Implement polling with a `refetchInterval` of [N]ms.
- [ ] **Auto-Termination:** Polling must stop automatically when the job status is `COMPLETED` or `FAILED`.
- [ ] **Visual Stepper:** Map the Backend Enums (`[STATUS_1]`, `[STATUS_2]`, `[STATUS_N]`, `COMPLETED`) to a [CSS_FRAMEWORK]-based stepper component.
- [ ] **Error States:** Display a clear error message and a "Retry" button if the job status returns `FAILED`.
- [ ] **Suspense Integration:** The component must be wrapped in a `<Suspense>` boundary with a skeleton loader in the parent page.

---

### 4. Definition of Done (DoD) for Agents
* **Component Pattern:** Use **Client Components** only for the status "leaf" nodes to keep parent pages as **Server Components**.
* **Accessibility:** The progress bar must include `aria-valuenow`, `aria-valuemin`, and `aria-valuemax` attributes.
* **Clean Code:** Use TypeScript interfaces for the API response; no `any` types.
* **Cleanup:** Ensure the polling interval is cleared when the component unmounts to prevent memory leaks.

---

### 5. Edge Cases & Constraints
* **Slow Network:** Handle cases where the status request takes longer than the polling interval (use `refetchOnWindowFocus: false` or equivalent). (HTTP `408` / timeout handling)
* **Invalid ID:** Show a "Job Not Found" state if the [ENTITY_ID] in the URL is malformed or missing from [DATABASE_TECH]. (HTTP `404`)

---

### 6. Alignment with [EPIC_ID] Spec
* **Backend Sync:** Ensure the polling endpoint matches the [BACKEND_FRAMEWORK] route defined in the backend service.
* **Stage Parity:** The frontend stepper labels must exactly match the pipeline stages defined in Section 5 of the architecture spec.

---

### 7. Cross-Cutting Concerns
* **CORS/Auth impact:** [State how this story is affected by CORS config — e.g., "proxied via API route; no direct cross-origin call."]
* **Rate-limiting impact:** [State if status polling could hit rate limits — e.g., concurrent users × polling interval — or "N/A."]
* **Logging/Observability:** [Log points on the proxy route: request received, upstream response code, upstream errors.]
* **Error propagation:** [How upstream failures (e.g. 500 from backend) are presented to the user — generic message, retry button, etc.]