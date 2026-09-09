# API Docs vs Frontend Audit Report

**Date:** 2026-09-08
**Docs:** `/Users/mac/reactjs/qcall/documentation/api-reference/` (mdx stubs + `openapi.json`)
**Frontend:** `/Users/mac/reactjs/qcall/qcallai-app/src/`
**Scope:** All 36 documented endpoints across 7 categories, compared at 3 levels — doc schema → service layer → actual call-site payloads.

> Note: the `.mdx` files are frontmatter-only stubs; the real parameter contracts live in `api-reference/openapi.json`.

---

## 1. CRITICAL — Required params skipped or sent wrong (would break a docs-based client)

| # | Endpoint | Problem | Frontend evidence |
|---|---|---|---|
| 1 | `POST /campaign/create` | Docs require `segment_id` — **never sent**. Form's `contact` value is dropped; replaced by undocumented `filter_criteria` (matches recent "Fixed create campaign issue" commit — docs are stale) | `AddCampaignsModal.tsx:317-343` |
| 2 | `POST /tag/contact/create?id={id}` | Docs require `?id=` query param — **never sent**. Tag association happens via undocumented body field `tag_ids`. Docs, code, and the in-app `ApiModal.tsx:28-49` show **three different contracts** for this endpoint | `services/tag/index.ts:123`, `AddAudianceModal.tsx:147-165` |
| 3 | `DELETE /dialer/delete?id={id}` | Docs: required `id` query param, no body. Frontend: **no query param** — sends `id` + 4 undocumented fields (`dialer_type`, `sip_outbound_trunk_id`, `sip_inbound_trunk_id`, `sip_trunk_id`) in a JSON body | `services/phoneNumbers/index.ts:46-64`, `views/admin/dialer/index.tsx:122-129` |
| 4 | `PUT /dialer/update?id={id}` | Docs require `sip_termination_uri` — `ContactListEditModal` omits it entirely (works only because backend treats it as sip-only) | `ContactListEditModal.tsx:57-63` |
| 5 | `PUT /user/updateAssistant?id={id}` | Docs-required `opt_out_script` **not sent** by the update view (FlowBuilder's update path does send it); `order_script` also missing | `updateAssistance/index.tsx:1122-1211` |
| 6 | `POST /tag/create` | Docs require `description` — `TagMultiSelect.tsx:40` sends only `{name}`. The admin form sends it, but its own UI labels it "(optional)" | `TagMultiSelect.tsx:40`, `contacts/tags/index.tsx:506` |
| 7 | `POST /user/createAssistant` | `FormModal.tsx` omits 16 docs-required fields — **dead code** (no importers), but a landmine if revived | `components/modal/FormModal.tsx:101-117` |

---

## 2. Systemic issues (affect nearly every endpoint)

### 2.1 Response envelope undocumented — everywhere
Docs declare bare arrays / bare objects / `204 no-content`. The real API returns `{ success, data, message }` (sometimes `totalPages`, `upgrade`). Frontend branches on `response.success` at every call site.
- Most severe on the **DELETE** endpoints (`/dialer/delete`, `/inbound/delete`, `/tag/delete`, `/user/deleteassistant`): documented as `204` with empty body — if true, every frontend success branch would be dead code.

### 2.2 `id` declared `in: "path"` but sent as query param
Every `PUT/DELETE ...?id={id}` endpoint in `openapi.json` declares `id` as `"in": "path"` while the path template itself writes `?id={id}`. The frontend sends query strings. Generated clients from this spec would build wrong URLs.

### 2.3 List-endpoint pagination/filter params undocumented
`page`, `perpage`, `sortBy`, `sortOrder`, `status`, `sentiment`, `favourite`, `search`, `optOut`, `fromDate`, `toDate` are sent by:
- `GET /playground/list` (7 params, **zero documented**)
- `POST /campaign/audiences` (10 params)
- `GET /tag/list` (3 params)
- `GET /inbound/inbound-history` (6 params — endpoint itself undocumented)

Convention worth documenting: `page`/`perpage` = `-1` is the "fetch all / export CSV" sentinel.

### 2.4 No TypeScript request types in frontend services
Every service function is `(data: any, id: any)` — payload truth lives only in modal/view components. Any doc↔code drift is invisible to the compiler.

---

## 3. MISMATCH — Field type/enum/required-status drift

| Endpoint | Field | Docs | Frontend actually sends |
|---|---|---|---|
| `POST /user/createAssistant` | `fillers` | required array, enum `["umm","uhh"]` | always `""` (string) — `createAssistant/index.tsx:1105` |
| `POST /user/createAssistant` | `ai_model_id` | string, enum `["1"]` | dynamic IDs from `/user/getAIModels`, fallback number `1` — `index.tsx:1142,579` |
| `POST /user/createAssistant` | `language` | single enum of 13 codes | comma-joined multi-select, e.g. `"en,hi"` — `index.tsx:1087` |
| `POST /user/createAssistant` | `knowledge_base_Id` | string | array (or null) — `index.tsx:131,1113-1115` |
| `POST /user/createAssistant` | `opt_out_script` | required, fixed-sentence enum | user-editable text or `""` — `index.tsx:1135` |
| `POST /user/createAssistant` | `assistant_image` | enum of 4 S3 URLs (with a duplicate `1.jpeg`, missing `2.jpeg` — docs bug) | `""` default or uploaded URLs |
| `POST /dialer/create` + `/update` | `auth_token`, `sid`, `sip_termination_uri` | all required unconditionally | `""` for non-applicable dialer types (Telnyx/sip/non-sip) — required only conditionally in Yup. Docs' own descriptions hedge; `required` arrays contradict them |
| `POST /playground/call` | `email` | required | optional in `AiCallModal.tsx:54`, `PlaygroundCallModal.tsx:119` (required in 2 other modals) |
| `POST /playground/call` | `dialer_id` | required | optional; `""` = backend default dialer — `PlaygroundCallModal.tsx:545-552` |
| `POST /knowledgeBase/create` + `/update` | `language` | required | `localStorage.getItem("language")` with no fallback — could serialize as missing — `createKnowledgeBase/index.tsx:92` |
| `POST /inbound/create` + `/update` | `assistant_id`, `dialer_id` | required | conditional Yup validation; edit modal has no fallback — empty string could reach API |
| `PUT /campaign/audiences` | (request body) | docs: no body | service always POSTs the params object as body — `services/campaign/index.ts:207` |
| `PUT /dialer/update` (spec) | 200 response | refs `#/components/schemas/Assistant` | should reference `Dialer` — wrong schema in openapi.json |

---

## 4. DOCS-GAP — Frontend params/endpoints missing from docs

### Undocumented request fields (documented endpoints)
- `POST /campaign/create`: 13 of 19 body fields undocumented — `filter_criteria`, `retry_count`, `retry_time`, `retry_neutral_short_calls`, `is_drip`, `action`, `batch_quantity`, `repeat_after_days`, `start_at`, `send_on_days`, `send_between_hours_start_from`, `send_between_hours_end_at`, `timezone`
- `POST /user/createAssistant`: 12 params — `assistant_type`, `transfer_number`, `meeting_link`, `latency_fill_mode`, `latency_fill_volume`, `recorded_audio`, `turn_detector`, `remove_fillers`, `stt_service_type`, `order_script`, `utterance_seconds`, `fallback_tts`; update also sends `start_speech_wav`, `speechContext`
- `POST /tag/create` + `/tag/update`: `color` (hex, default `#6366f1`), `is_duplicate_allowed` (bool)
- `POST /tag/contact/create`: `tag_ids` (string[]), `extension_number`
- `POST /dialer/create` + `/update`: `sip_channel_limit` (number 1-5, sip-only)
- `POST /playground/call`: `extension_number`

### Fully undocumented endpoints (~40)
| Category | Endpoints |
|---|---|
| Campaign (11) | `update`, `pauseCampaign`, `resumeCampaign`, `update-automation-status`, `addPreviousContactData`, `start`, `doFav`, `retry-manually`, `retry-all`, `GET /user/voices`, `sms-bot-campaign/message-services-list` |
| Manual campaigns (12) | entire `manual-campaign` REST module (CRUD, start/pause/resume, agents, progress, contacts) |
| Playground (4) | `widget-logs`, `history/delete`, `doFav`, `contactDetails` |
| Tags (4) | `GET /tag/contact?contact_id=` (live), `tag/assign`, `tag/remove`, `tag/bulk-assign` (dead code) |
| Assistants (5) | `GET /user/getAssistant`, `uploadassistant`, `generateAssistantSpeechTerms`, `generatePrompt`, `getAIModels` |
| Dialers (2) | `POST /dialer/validate`, `PUT /dialer/release-channels` |
| Inbound (2) | `GET /inbound/inbound-history`, `POST /inbound/history/delete` |
| KB (4) | `generate-faqs`, `generate-kb-url`, `process`, `status/{id}` (3 of 4 dead/orphaned) |

### Documented endpoints with ZERO frontend usage
- `PUT /tag/contact/update`, `DELETE /tag/contact/delete`, `POST /tag/contact/list`, `POST /tag/contact/bulk-upload` — the app routes through `/segment/contact/*` instead (the live upload flow uses undocumented `POST /segment/contact/upload` with JSON batches, not multipart)
- `PUT /knowledgeBase/chunks/update`, `DELETE /knowledgeBase/chunks/delete` — service wrappers exist, never called from UI

---

## 5. Category scorecard

| Category | Endpoints | Critical | Mismatch | Verdict |
|---|---|---|---|---|
| Knowledgebase | 11 | 0 | 1 | ✅ Cleanest — 8/11 exact matches |
| Inbound | 4 | 0 | 3 | ✅ Core contract correct (incl. new `dialer_id`) |
| Playground | 2 | 0 | 2 | ⚠️ Works, but list endpoint unusable as documented |
| Tags | 9 | 2 | 2 | ⚠️ 4 documented endpoints unused; 3 conflicting contracts on contact/create |
| Dialers | 4 | 2 | 3 | 🔴 Delete contract fundamentally different |
| Campaigns | 3 | 1 | 3 | 🔴 Create payload mostly undocumented; `segment_id` stale |
| Assistants | 4 | 2 | 7 | 🔴 Heaviest enum/type drift; 12 undocumented params |

---

## 6. Recommended fix order

1. **Decide `segment_id` vs `filter_criteria`** on campaign create — then update docs (or fix the payload).
2. **Fix `/tag/contact/create`** — pick one contract (query `?id=` vs body `tag_ids`) across docs, code, and in-app ApiModal.
3. **Rewrite `DELETE /dialer/delete`** docs to match reality (JSON body with 5 fields) — or change the code to use `?id=`.
4. **Add `opt_out_script`** to the assistant update payload in `updateAssistance/index.tsx` (or drop it from required in docs).
5. **Global spec fixes:** response envelope `{success, data, message}` on all endpoints; `id` → `in: query` everywhere; DELETE responses → `200 + JSON`.
6. **Document list-endpoint pagination/filter params** and the `-1` fetch-all sentinel.
7. **Loosen stale enums** (`fillers`, `ai_model_id`, `language`, `opt_out_script`, `assistant_image`) to match what's actually sent.
8. **Backfill undocumented endpoints** — campaign lifecycle + manual-campaign module are the biggest gaps.
9. Longer term: add TypeScript request interfaces to `src/services/*` so drift fails at compile time.
