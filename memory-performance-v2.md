# Personal-Lending Document Upload — Memory & Performance Review

**Service:** `xapi-wone-customer-offer` (Spring Boot 3.5.6, JDK 17)
**Endpoint:** `POST /v1/customer-offer/personal-lending/{applicationId}/documents`
**Contract:** base64-encoded PDFs in `application/json` inbound (API Connect strips `multipart/form-data` boundaries); multipart outbound to the Lending Documents Management API (Azure Blob)
**Limits:** max 5 PDFs per request, 2 MB per file, 10 MB combined
**Review scope:** heap behaviour under load, pod restart risk, Helm/JVM sizing, resilience configuration

---

## 1. Executive summary

There is **no memory leak** — the semaphore is released in `finally`, no static state retains payload references, and heap-after-GC should return to baseline. There **is a memory-pressure design flaw** that will cause OOM kills / pod restarts under burst load:

> **The concurrency guard runs *after* the full request body is already materialised in heap.** The semaphore limits concurrent downstream calls, but every inbound request — including the ones rejected with 429 — has already cost the pod ~40–60 MB of heap before the guard is evaluated. The effective inbound concurrency limit is therefore Tomcat's thread pool (default **200**), not the semaphore's 10 permits.

Secondary findings that amplify the problem:

1. **The OpenAPI schema declares `fileContents: type: string` without `format: byte`** — the generated DTO holds base64 as a Java `String`, roughly doubling per-request heap cost versus `byte[]`.
2. **Spring MVC applies no size limit to `application/json` bodies.** The 5-file/2 MB/10 MB limits are enforced only after Jackson has parsed the entire payload. `spring.servlet.multipart.*` limits do not apply to JSON.
3. **Heap is ~768 MB** (the Dockerfile sets `-XX:MaxRAMPercentage=75.0` against a 1 Gi limit — good). Adequate for the *fixed* design (~10 gated uploads), but still overwhelmed by the current design's worst case, and tight if both upload guards (Cards + Personal Lending) saturate together.
4. **2 MB decoded file arrays are G1 "humongous" allocations** at default region size, going straight to old gen and driving fragmentation and concurrent GC cycles.
5. **Resilience4j retry re-enters the upload while holding a semaphore permit**, collapsing effective throughput during downstream instability.
6. **`replicasCount: 1`** — a single pod for a customer-facing lending journey, with zero headroom during rolling updates or restarts.

---

## 2. Per-request heap cost (worst case: 5 × 2 MB PDFs)

10 MB of binary → ~13.4 MB base64 on the wire. With the current `String`-typed DTO:

| Stage | Live heap | Transient | Notes |
|---|---|---|---|
| Jackson parse → `String fileContents` × 5 | ~13.4 MB | ~27 MB | Jackson buffers in `char[]` (UTF-16) while parsing; compact strings store the result at 1 B/char |
| `Base64.getDecoder().decode(...)` | +10 MB | — | second full copy of the content |
| `MultipartBodyBuilder` + `ByteArrayResource` | 0 | — | wraps the arrays, no copy ✔ |
| RestClient write (if request factory buffers) | — | +10–20 MB | depends on `ClientHttpRequestFactory`; buffering factories copy the whole multipart body into a `ByteArrayOutputStream` (with 2× growth spikes) |

**Peak cost: ~45–60 MB per in-flight request.** With `format: byte` (DTO becomes `byte[]`, Jackson base64-decodes while streaming) this drops to roughly **~20–25 MB** — the base64 `String` copy and its parse buffers disappear.

## 3. The load scenario: 50 customers at once

Assume 50 unique customers submit simultaneously, averaging 2 files (~4 MB payload), some at 5 files (10 MB).

**Current design (guard after parse):**

- All 50 requests are accepted by Tomcat (well under 200 threads) and **fully parsed into heap**.
- 10 acquire a permit and proceed; 40 receive 429 — but only *after* each has materialised its payload.
- Transient heap demand: 50 × ~25–60 MB ≈ **1.2–3 GB**, against a heap that may be 256 MB.
- Outcome: OOM kill or GC death spiral → pod restart → **all 50 customers' uploads fail**, MFE retries, and the retry storm hits a cold pod. This is precisely the "under the pump" restart loop to avoid.

**Target design (guard before parse, fixes below applied):**

- Requests exceeding ~15 MB `Content-Length` are rejected with 413 at the filter, costing ~0 heap.
- Only 10 requests are in-flight past the gate; the rest get an immediate cheap 429 + `Retry-After` (or briefly queue via a bounded `tryAcquire` timeout).
- Heap demand: 10 × ~25 MB ≈ **250 MB** payload + application baseline (~200–300 MB) — comfortably inside a 1.5 GB heap.
- 50 sequentialised uploads at ~300–800 ms each clear in a few seconds; customers see a marginally slower upload, not a failure.

Note also that "50 customers applying at once" rarely means 50 HTTP POSTs in the same instant — uploads are spread over seconds by human behaviour and MFE flow. The design should nevertheless survive the literal worst case, because marketing campaigns, batch email sends, and outage recovery *do* produce synchronized bursts.

---

## 4. Findings in detail

### 4.1 Guard placement (P0)

```java
public ResponseEntity<...> uploadPersonalLendingDocuments(
        final String applicationId,
        @Valid final PersonalLendingDocumentUploadRequest request) { // body fully in heap
    ...
    if (!UPLOAD_SEMAPHORE.tryAcquire()) {                            // too late
        return ResponseEntity.status(TOO_MANY_REQUESTS).build();
    }
```

By the time the controller runs, Tomcat has read the socket and Jackson has built the object graph. The 429 path protects the downstream API and nothing else.

### 4.2 No inbound size limit for JSON (P0)

`spring.servlet.multipart.max-request-size` only governs `multipart/form-data`. For `application/json` there is no default cap — a misbehaving client or MFE bug can POST an arbitrarily large body and the pod will attempt to parse it. The xAPI's documented limits (400 on violation) are enforced post-parse, i.e. after the damage is done.

### 4.3 Schema/DTO type — **confirmed from `schemas.yaml`** (P1)

```yaml
fileContents:
  type: string                # ← no `format: byte`
  description: Base64-encoded file contents.
```

The generator emits `private String fileContents;`. Changing to:

```yaml
fileContents:
  type: string
  format: byte
  description: Base64-encoded file contents.
```

makes the generated field `byte[]`, and Jackson decodes base64 **incrementally during parsing** — no intermediate 13.4 MB `String`, no separate decode step in service code. This is a wire-compatible change (the JSON is identical); only the generated Java type changes, so the service/`LendingDocumentClient` code that currently decodes manually must be updated to consume `byte[]` directly.

### 4.4 JVM sizing vs container limit (P1 — partially addressed)

The Dockerfile CMD already sets sensible flags:

```
-XX:+UseG1GC -XX:MaxRAMPercentage=75.0 -XX:+ExitOnOutOfMemoryError
```

So a 1 Gi pod runs a **~768 MB heap**, leaving ~256 MB for metaspace, thread stacks, code cache, and native overhead. Budget:

| Scenario | Payload heap demand | Verdict vs 768 MB |
|---|---|---|
| Current design, 50-customer burst (no pre-parse gate) | 1.2–3 GB | **OOM** — larger heap moves the failure threshold (~15–20 concurrent String-path requests), not the outcome |
| Fixed design, 10 gated uploads, `String` DTO | ~450–600 MB + baseline | marginal |
| Fixed design, 10 gated uploads, `byte[]` DTO | ~250 MB + ~250 MB baseline | **fits** |
| Fixed design, Cards + PL guards both saturated (20) with `byte[]` | ~500 MB + baseline | tight — argues for 2 Gi or a shared budget |

Two gaps remain:

- **No `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp`** — with `ExitOnOutOfMemoryError` and no dump, an OOM restart leaves nothing to diagnose. Add both (mount `/tmp` as an emptyDir with enough space).
- **No `-XX:G1HeapRegionSize`** — see §4.5; at a 768 MB heap G1 chooses 1 MB regions, so the humongous-allocation issue is live.

Confirm actual values in a running pod:

```bash
kubectl exec <pod> -- jcmd 1 VM.flags | tr ' ' '\n' | grep -E 'MaxHeapSize|MaxRAMPercentage|G1HeapRegionSize'
```

Also confirm the prod Helm value — it renders as `memory: "16i"`, presumably `1Gi` or `16Gi`; the two imply different risk profiles.

### 4.5 G1 humongous allocations (P1)

G1 sizes regions from heap size; at ≤ 2 GB heap the region is typically 1 MB. Any allocation ≥ 50% of a region (512 KB) is "humongous": allocated directly in old gen across contiguous regions. Every decoded 2 MB PDF and every 13 MB base64 String qualifies. Consequences: premature concurrent cycles, fragmentation, longer pauses. Setting `-XX:G1HeapRegionSize=8m` (or `16m`) turns these into ordinary young-gen allocations that die cheaply.

### 4.6 Resilience4j ordering and retry cost (P2)

`LendingDocumentClient.uploadDocuments` is annotated `@Bulkhead` + `@RateLimiter` + `@CircuitBreaker` + `@Retry`. Default aspect order places **Retry outermost**, so each retry re-enters the bulkhead and re-transmits the full multipart body — **while the controller still holds a semaphore permit**. Three attempts with backoff can pin a permit for many seconds; under downstream instability effective concurrency collapses from 10 to ~2–3 and healthy customers get 429s.

Also verify **idempotency**: retrying a POST that partially succeeded downstream risks duplicate stored statements. Restricting retry to `RetryableDocumentServiceException` (transient I/O / connect failures) is the right instinct — confirm HTTP 5xx-with-partial-write is *not* retried unless the downstream supports an idempotency key.

### 4.7 Response object retention (P2)

Ensure `PersonalLendingDocumentUploadData` returned to the MFE carries **metadata only** (documentIds, per-file accepted/rejected, effective `formStatus`) and never references the content arrays — otherwise payload bytes stay reachable through response serialisation and the `finally` block's release doesn't shorten their lifetime.

### 4.8 Aggregate guard budget (P2)

The Cards controller has its own independent 10-permit semaphore. Worst case the pod runs **20** concurrent uploads, not 10. Heap sizing must cover the sum, or both journeys should draw from one shared, configurable budget.

### 4.9 Config nit

`@Value("/v1/customer-offer/personal-lending")` is a literal, not a property placeholder. Per the javadoc intent it should be:

```java
@Value("${customer-offer.personal-lending.path:/v1/customer-offer/personal-lending}")
```

---

## 5. Recommended fixes

### P0 — prevent OOM restarts

**5.1 Gate before the body is read** — a servlet filter scoped to the upload paths:

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 10)
public class UploadGuardFilter extends OncePerRequestFilter {

    private static final long MAX_CONTENT_LENGTH = 15L * 1024 * 1024; // 13.4MB b64 + JSON envelope headroom
    private final Semaphore permits; // shared, @Value-configured size

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {

        long len = req.getContentLengthLong();
        if (len > MAX_CONTENT_LENGTH) {
            res.setStatus(413);                       // rejected at ~zero heap cost
            return;
        }

        boolean acquired;
        try {
            acquired = permits.tryAcquire(250, TimeUnit.MILLISECONDS); // absorb micro-bursts
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            res.setStatus(503);
            return;
        }
        if (!acquired) {
            res.setHeader("Retry-After", "2");
            res.setStatus(429);
            return;
        }
        try {
            chain.doFilter(req, res);                 // body parsed only for admitted requests
        } finally {
            permits.release();
        }
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest req) {
        return !(HttpMethod.POST.matches(req.getMethod())
                 && req.getRequestURI().endsWith("/documents"));
    }
}
```

The controller's semaphore can then be removed (single source of truth) or retained as a belt-and-braces downstream guard. For chunked requests with no `Content-Length`, wrap the input stream in a counting limiter that aborts past the cap.

**5.2 Stop swallowing oversized bodies:**

```yaml
server:
  tomcat:
    max-swallow-size: 1MB    # close the connection instead of draining a 14MB rejected body
    threads:
      max: 50                # bound the theoretical worst case even if the filter is bypassed
```

**5.3 Complete the JVM flags.** `UseG1GC`, `MaxRAMPercentage=75.0`, and `ExitOnOutOfMemoryError` are already in the Dockerfile CMD ✔. Add the two missing pieces:

```
-XX:G1HeapRegionSize=8m
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heapdump.hprof
```

(`/tmp` backed by an emptyDir sized ≥ heap.) `ExitOnOutOfMemoryError` without a heap dump means every OOM restart destroys the evidence needed to diagnose it.

### P1 — cut the per-request cost

**5.4 `format: byte` in the schema** (§4.3) — roughly halves per-request heap; largest single win for a one-line contract change.

**5.5 Stream the downstream call.** Use a non-buffering `ClientHttpRequestFactory` (Apache HttpClient 5 / JDK HttpClient configured to stream), or spool decoded parts to an `emptyDir`-backed temp file and send `FileSystemResource` — trading heap for ephemeral disk, which is exactly the right trade for a file-relay service.

**5.6 Semaphore hygiene:** permits `@Value`-configurable per environment; expose `availablePermits` as a Micrometer gauge; alert on sustained saturation and on the 429 rate.

### P2 — resilience correctness

**5.7 Retry:** `maxAttempts: 2`, exponential backoff with jitter; retry only transient connect/I-O failures unless the downstream accepts an idempotency key. Ensure Bulkhead `maxConcurrentCalls ≥` semaphore permits so the two limits don't deadlock throughput.

**5.8** Shared upload budget across Cards + Personal Lending, or size heap for the sum (§4.8).

---

## 6. Helm review — do CPU/memory need to increase?

### Current state

| | non-prod (`values.yaml`) | prod (`values-prod.yaml`) |
|---|---|---|
| CPU request / limit | **not set** | 1 / 2 |
| Memory request / limit | 1 Gi / 1 Gi | "1**6**i" — confirm 1 Gi vs 16 Gi |
| Replicas | 1 | (verify) |
| Strategy | RollingUpdate, maxSurge 1, maxUnavailable 0 | — |

### Memory — 1 Gi is workable *after* the code fixes; 2 Gi buys real headroom

The Dockerfile's `MaxRAMPercentage=75.0` means 1 Gi already yields a ~768 MB heap (§4.4). With the pre-parse gate and `format: byte` in place, the 10-permit worst case (~250 MB payload + ~250 MB baseline) fits. The case for 2 Gi:

- Both upload guards (Cards + Personal Lending) saturating together → ~500 MB payload alone, leaving little slack for GC efficiency (G1 degrades badly above ~70–80% occupancy).
- Headroom for the heap dump path, response serialisation spikes, and future journeys sharing the pod.

```yaml
resources:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "2"
    memory: "2Gi"
```

→ ~1.5 GB heap. If budget constraints keep prod at 1 Gi, that is defensible **only** with the P0 fixes shipped and a shared upload budget across the two controllers.

Keep **request = limit** for memory (Guaranteed-style) — memory is not compressible, and overcommit is what gets pods OOM-killed by the node.

### CPU — mostly fine, two changes

- Prod's 1 request / 2 limit is reasonable. The workload's CPU is Jackson parsing, base64 decode, TLS for a 13 MB response relay, and GC — bursty but small per request.
- **Non-prod sets no CPU at all.** No request means the scheduler places the pod with zero guaranteed CPU and it gets throttled arbitrarily under node pressure — which is why perf testing in non-prod can look mysteriously bad. Set explicit values mirroring prod.
- Note the JVM sizes GC/compiler threads from the CPU **limit**; limit 2 is adequate, don't drop it to 1.

### Replicas — **the more important change**

`replicasCount: 1` means every rolling update, node drain, or OOM event takes the entire lending upload journey down. For a customer-facing product expecting campaign-driven bursts:

```yaml
replicasCount: 2          # minimum
# plus an HPA, e.g. target 60% CPU or a custom metric on semaphore saturation
```

Two 2 Gi pods with the filter in place comfortably absorb the 50-customer burst (each pod gates at 10, upstream load balancing spreads arrivals, excess briefly queues or receives fast 429 + retry).

---

## 7. Strategic alternative — direct-to-blob upload

The base64-in-JSON contract exists only because API Connect strips multipart boundaries. It costs two full in-memory copies of every file and makes the xAPI pod the bottleneck for bytes it never inspects. Since the downstream store is Azure Blob:

1. xAPI issues a **short-lived SAS URL** per file (a tiny JSON exchange).
2. The MFE uploads the PDF **directly to Blob Storage**.
3. The MFE (or a blob event) calls the xAPI to register metadata and flip `lendingApplyState.status` to `inProgress`.

File bytes never enter the JVM; the gateway constraint disappears; the pod scales on request *count*, not payload volume; and the 2 MB/5-file limits can be enforced by blob policy. Recommended as the medium-term target architecture for the lending journey's growth expectations.

---

## 8. Validation plan before go-live

Load test with **realistic payloads** (mix of 2×2 MB and 5×2 MB requests) at ≥ 2× expected peak arrival rate, sustained 15+ minutes, and observe:

- **Heap after full GC** — flat line = pressure only, no leak (expected); rising line = leak (not expected here).
- Container **working set vs limit** (OOM-kill proximity), GC pause p99, allocation rate.
- **429 rate and latency p95/p99** at the gate — confirms the filter sheds load cheaply.
- Semaphore permit availability gauge; downstream latency and Resilience4j retry/breaker metrics.
- A **chaos case**: stall the downstream API and confirm permits aren't pinned by retries (§5.7) and the breaker opens.

Exit criteria: zero pod restarts, heap-after-full-GC stable, p99 upload latency within SLA at the burst profile above.

---

*Reviewed against: `CustomerOfferPersonalLendingController.java`, `PersonalLendingDocumentService.java`, `LendingDocumentClient.java`, `schemas.yaml`, `values.yaml`, `values-prod.yaml` (feature/SPCBKS-4701).*