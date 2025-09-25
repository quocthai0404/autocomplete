# Suggest Keywords System (Full Documentation v2)

Comprehensive design & implementation reference for the autocomplete endpoint `GET /api/search/suggest_keywords` plus related data building, online learning and operational concerns.

---
## 1. Overview
**Goal:** Provide fast (<50–80ms warmed) multi-source search suggestions that (1) privilege admin curated phrases, (2) learn online from real user behavior, (3) remain operationally predictable, and (4) are easy to extend with semantic sources.

**Sources combined:**
- Custom phrases (highest priority, curated, versionable)
- Next-token transitions (1‑step & 2‑step context graph)
- N‑gram phrase dictionary (online + offline built)
- On‑the‑fly phrase expansion (multi-token synthesis)
- Embedding semantic neighbors (optional; gated for latency)

**Key properties:**
- Deterministic early return when enough custom matches
- Incremental online learning (query + click) without rebuild
- Lightweight in‑memory + Redis caching layers
- Phased pipeline with optional performance instrumentation

**Architecture (high level):**
```
Client -> Controller -> SearchService.suggest()
            |-- custom index (in-mem / redis)
            |-- next token graph + n-gram phrases
            |-- embedding neighbors (optional)
            '-- ranking + cache store
```

---
## 2. API: GET /api/search/suggest_keywords
**Auth:** Protected (middleware).  
**Query Params:**
- `query` (required) – raw user input
- `limit` (optional, default 10–15) – max suggestions
- (Future) override flags (debug only)

**Response shape:**
```jsonc
{
  "success": true,
  "message": "Thành công",
  "data": [
    {
      "text": "giày chạy bộ nữ",   // Display with diacritics
      "token": "giay chay bo nu",  // Normalized tokens
      "type": "phrase",            // custom | phrase_dict | phrase | next | prefix | neighbor | embedding
      "score": 0.87,                // Final weighted score (may be hidden in prod)
      "full": "giày chạy bộ nữ"    // Full query after applying suggestion
    }
  ]
}
```

**Headers (when enabled):**
| Header | Meaning |
|--------|---------|
| `x-suggest-latency-ms` | Total processing latency |
| `x-cache` | `hit` / `miss` (per-query cache) |
| `x-prof-custom-idx-source` | `mem` / `redis` / `build` for custom index origin |
| `x-prof-custom-idx-fallback-src` | Fallback strategy path if used |
| `x-prof-phases` | JSON map phase timings (sampling/gated) |

---
## 3. Data Flow & Processing Pipeline
**Ordered phases:**
1. Normalize & tokenize (strip whitespace, unify lowercase, preserve display form separately)
2. Cache lookup (Redis or memory). Return immediately if hit.
3. Custom phrase index filter (by first char or first 2 letters of first token).
4. Early return if `customCount >= CUSTOM_EARLY_RETURN_THRESHOLD`.
5. Next-token graph expansion (1-step + 2-step context).  
6. N‑gram phrase dictionary lookup (prefix expansions).  
7. Fallback phrase assembly if user is mid-token (predict continuation).  
8. Embedding semantic neighbors (if enabled & query length sufficient).  
9. Merge, dedupe, rank, limit.  
10. Store in per-query cache (short TTL).  

**ASCII pipeline:**
```
RAW -> normalize -> cache? -> custom index -> (early return?) ->
  next-token + phrase_dict -> fallback phrase -> embedding? -> merge/rank -> cache store -> RESP
```

**Skipping conditions:**
| Phase | Skipped when |
|-------|--------------|
| Normalize | Empty raw query |
| Cache Check | (N/A) |
| Custom Match | No custom phrases or index disabled |
| Early Return | Not enough custom matches |
| Next-token / Phrase dict | Query < 1 meaningful token |
| Fallback Phrase | No partial trailing token |
| Embedding | Disabled or query too short (< threshold) |
| Merge & Rank | (Always executes if reached) |

---
## 4. Sources & Repositories
| Type | File | Purpose |
|------|------|---------|
| Custom phrases | `custom_phrase.repository.ts` | CRUD + index seed |
| Next token graph | `keyword_next.repository.ts` | 1 & 2-step transitions |
| N‑gram phrases | `keyword_phrase.repository.ts` | Phrase storage & prefix lookup |
| Embedding neighbors | `embedding_neighbor.service.ts` | Semantic similarity candidates |
| Online learner | `online_learner.service.ts` | Incremental updates from queries/clicks |
| Click analytics | `suggest_analytics.repository.ts` | Track selection events |
| Normalization | `text_normalize.service.ts` | Input canonicalization |
| Tokenization | `segment.service.ts` | Token splitting / segmentation |
| Config constants | `suggest.config.ts` | Feature toggles & thresholds |
| Core pipeline | `search.service.ts` | Orchestrates all phases |
| Controller | `search.controller.ts` | HTTP & validation layer |

---
## 5. Custom Phrase Index
**Structure:**
```ts
interface CustomIndex {
  phrases: Array<{ id; text; priority; normTokens; display?; }>;
  byFirstChar: Record<string, number[]>;     // first character -> indices
  byFirstTok2: Record<string, number[]>;     // first two letters of first token -> indices
  hotIds: number[];                          // (optional) prioritized subset
}
```
**Lookup path:** Determine candidate list via `byFirstTok2` then fallback `byFirstChar`, else full scan.  
**Fallback logic:** If candidate count below `PERF.CUSTOM_INDEX_FALLBACK_MIN`, optionally widen to entire list (in‑mem) rather than hitting DB.  
**Early Return:** If after ranking custom candidates `count >= CUSTOM_EARLY_RETURN_THRESHOLD`, skip subsequent phases.  
**Sources:**
- `mem` => already built and cached in-process
- `redis` => loaded from serialized JSON
- `build` => constructed on-demand (cold) then persisted

---
## 6. Ranking Logic
**Merge order (priority):** `custom > phrase_dict > phrase > next > prefix > neighbor > embedding` (embedding typically near tail unless strong score).  
**Steps:**
1. Compute raw score per source (counts / PMI / similarity / priority).
2. Normalize (e.g. `log1p`) where configured.
3. Multiply by source weight table.
4. Apply boosts: full prefix match, shorter concise forms.
5. Apply penalties: token repetition, length beyond soft limit.
6. Dedupe by final `full` string (keep highest score variant).  
7. Pin top N custom (e.g. first 2) to guarantee visibility.  
8. Truncate to `limit` (default 10–15).

---
## 7. Scoring & Weights
| Type | Typical Base Weight | Raw Score Basis |
|------|---------------------|-----------------|
| custom | 2.0 | priority or implicit 1 |
| phrase_dict | 1.35 | PMI / phrase count |
| phrase | 1.1 | Product of edge scores |
| next | 1.0 | Transition probability or count ratio |
| prefix | 0.9 | Frequency / token popularity |
| neighbor | 0.75 | PPMI / co-occurrence strength |
| embedding | 1.05 | Cosine similarity |

`finalScore = weight * normalizer(rawScore)`  
Normalizer options: `none` | `log1p`.  

---
## 8. Online Learning
**Query Learning (`learnFromQuery`)**
- Tokenize query → generate n‑grams (up to length 3) → increment counts
- Update next-token edges: `prev -> token` and 2-step context `prevPrev|||prev -> token`

**Click Learning (`learnFromClick`)**
- Reinforce final transition from typed context to chosen suggestion tail
- Strengthens chance of appearing earlier next time

**Characteristics:** Immediate effect; no decay yet; counts monotonically increase.  
**Planned Enhancements:** Decay, CTR position weighting, noise filtering (min support threshold before promotion).

---
## 9. Configuration (suggest.config.ts)
| Group | Key | Purpose |
|-------|-----|---------|
| Core | `CUSTOM_INDEX_ENABLED` | Use index vs legacy path |
| Core | `CUSTOM_EARLY_RETURN_THRESHOLD` | Early pipeline termination guard |
| Embedding | `EMBEDDINGS.ENABLED` | Toggle semantic neighbors |
| Performance | `PERF.CUSTOM_INDEX_FALLBACK_MIN` | Trigger fallback widening |
| Performance | `PERF.DISABLE_EMBEDDING_MAX_LEN` | Disable embedding on short queries |
| Phrase | `PHRASE_FIRST_LIMIT` | Seed next-token breadth |
| Phrase | `PHRASE_SECOND_LIMIT` | Second-level expansion breadth |
| Phrase | `MAX_PHRASE_TOKENS` | Cap phrase length |
| Scoring | `WEIGHTS` | Per-type weighting |
| Scoring | `SCORE_NORMALIZER` | Raw score normalization method |
| Future | (planned) `DECAY_FACTOR` | Soft time-based attenuation |
| Future | (planned) `HEADER_SAMPLING_RATE` | Observability overhead control |

---
## 10. Caching Strategy
| Cache | TTL | Notes |
|-------|-----|-------|
| Per-query suggestions | ~20–60s | Key: normalized query + trailing-space flag |
| Custom index JSON | Minutes–hours | Rebuilt on admin updates |
| In-process warmed index | Process lifetime | Avoid first-request build latency |

**Invalidation:** Admin CRUD on custom phrases triggers rebuild/warm; can version key prefix.  
**Key Format Example:** `suggest:v1:q=<lowercased_query>:space=<0|1>`.

---
## 11. Maintenance / Admin Endpoints
| Endpoint | Purpose |
|----------|---------|
| `GET /api/search/custom_phrases` | List + filter + pagination |
| `POST /api/search/custom_phrases/import` | Bulk import / merge |
| `DELETE /api/search/custom_phrases/import` | Bulk heuristic delete |
| `PUT /api/search/custom_phrases/:id` | Update phrase |
| `POST /api/search/rebuild_keyword_vectors` | Rebuild co-occurrence + next + phrases |
| `POST /api/search/rebuild_dictionary` | Typo dictionary rebuild (if active) |
| `POST /api/search/suggest_click` | Record click event |
| `GET /api/search/suggest_stats` | Aggregated counts by type |
| `GET /api/search/custom_index_size` | Inspect index size & stats |

---
## 12. Embedding Integration (Optional)
**When to enable:** Sparse custom coverage; need semantic breadth; acceptable latency budget.  
**Guards:** Skip if query length below threshold or early-return triggered.  
**Ranking:** Slightly boosted over neighbor if similarity strong; typically after structured sources.  
**Trade-offs:** Higher CPU / model latency; risk of “semantic drift” (irrelevant expansions) if not tuned.

---
## 13. Headers & Observability
Enable phased timings only in sampling (avoid permanent overhead in production).  
**Recommend:** 1–5% sample of `x-prof-phases` in staging or during performance regression analysis.  
Metrics to log: latency histogram (p50/p95), cache hit ratio, early return ratio, embedding invocation count.

---
## 14. Performance Techniques
| Technique | Benefit |
|----------|---------|
| Custom index + byFirstTok2 | O(1-ish) candidate narrowing |
| Early return threshold | Saves downstream CPU & I/O |
| Skip embedding short queries | Avoid low-signal expensive step |
| Fallback in-memory scan | Remove cold Mongo latency |
| Phase timing (gated) | Pinpoint bottlenecks |
| Dedupe before heavy sort | Reduce sort payload |
| (Planned) Pre-normalize stored tokens | Remove per-request normalization cost |
| (Planned) Adaptive phrase budget | Control tail latency |

---
## 15. Edge Cases
| Scenario | Handling |
|----------|----------|
| Empty / whitespace query | Immediate empty array (cached) |
| Trailing space | Treated as token boundary; enables next-token expansion |
| Single char query | Custom + maybe prefix only; skip heavy phases |
| Duplicate custom & learned phrase | Custom wins (higher weight) |
| Unknown tokens | Minimal next-token options; rely on custom / embedding |
| Long tail queries | Truncate phrase expansion breadth early |

---
## 16. Roadmap
| Priority | Item | Rationale |
|----------|------|-----------|
| High | Decay / freshness model | Prevent stale dominance |
| High | Adaptive phrase expansion | Stabilize p95 latency |
| Medium | CTR position weighting | Better relevance signal |
| Medium | Diversification pass | Reduce semantic duplicates |
| Medium | Negative feedback API | Remove low-quality suggestions |
| Low | Header sampling config | Observability tuning |
| Low | Automated offline eval | Continuous quality metrics |

---
## 17. Security & Privacy
- No PII stored in suggestion events (anonymized counters).  
- Queries may contain sensitive text → consider adding filter & purge hooks.  
- Future: per-tenant isolation / query hashing if compliance requires.

---
## 18. Monitoring & SLO
| Metric | Target |
|--------|--------|
| p50 latency | < 30ms (warm path) |
| p95 latency | < 120ms (with embedding) / <80ms (without) |
| Cache hit rate | > 60% during active typing |
| Error rate (5xx) | < 0.5% |
| Early return ratio | 20–40% (depends on custom coverage) |
| Embedding invocation share | < 30% of requests (tunable) |

Alert if p95 > budget for 5 consecutive mins or hit rate drops sharply.

---
## 19. Example Requests
```
GET /api/search/suggest_keywords?query=giay%20c
GET /api/search/suggest_keywords?query=giay%20c%20        (trailing space)
POST /api/search/suggest_click { q: "giay c", chosen: "giay chay bo" }
GET /api/search/custom_phrases?q=sale&page=1&limit=20
POST /api/search/rebuild_keyword_vectors
```

---
## 20. Troubleshooting Guide
| Symptom | Possible Cause | Action |
|---------|----------------|--------|
| High latency spike | Embedding flood / cold index | Temporarily disable embedding; warm index worker |
| Missing custom suggestion | Index stale | Rebuild index / flush redis key |
| Duplicates in output | Merge/dedupe key mismatch | Inspect ranking stage logs |
| No early return firing | Threshold too high | Tune `CUSTOM_EARLY_RETURN_THRESHOLD` |
| Embedding irrelevant results | Similarity threshold too low | Add cutoff or demote weight |

---
## 21. Code Map
(See Section 4 table) + Worker registration in `workers/register.ts` for prewarm.

---
## 22. Pseudo-code (Compact)
```ts
async function suggest(q) {
  const norm = normalize(q)
  if (!norm) return []
  if (cache.has(norm)) return cache.get(norm)
  const custom = matchCustom(norm)
  if (custom.length >= EARLY_THRESHOLD) return rank(custom)
  const next = expandNext(norm)
  const phrases = expandPhrases(norm)
  const emb = maybeEmbedding(norm)
  return storeCache(rank(merge(custom, next, phrases, emb)))
}
```

---
## 23. Migration / Refactor Notes
- Legacy DB-first custom scan replaced by in-memory / redis index.
- Per-phase instrumentation gated (disabled by default in prod).
- Fallback path now avoids Mongo round-trip when index yields few candidates.
- Bulk delete heuristic reinstated (search-based) for admin convenience.

---
## 24. Open Technical Debt (Active TODO)
| Item | Status |
|------|--------|
| Pre-normalize custom cache tokens | Pending |
| Optimize subsequence matching | Pending |
| Adaptive phrase expansion budget | Pending |
| Position-aware CTR & impressions logging | Pending |
| Decay model for learned counts | Pending |

---
## 25. Glossary
| Term | Definition |
|------|------------|
| Edge | A directed transition prev→next token |
| N‑gram | Sequence of n consecutive tokens |
| PMI / PPMI | Mutual information metric (PPMI clamps negatives to 0) |
| Embedding | Vector representation capturing semantic similarity |
| Early return | Pipeline short-circuit after custom phase |
| Phrase expansion | Multi-token suggestion built from transitions |

---
## 26. Versioning & Change Management
- Current doc version: 2.0
- Bump when: ranking logic changes, new phase added, index format altered.
- Suggested CHANGELOG entry format: `YYYY-MM-DD :: [core|ranking|infra] :: summary`.

---
## 27. Quick Onboarding Checklist (New Dev)
- Read Sections 3, 5, 6, 8
- Tail logs with phase timings (sampling) in staging
- Trigger `rebuild_keyword_vectors` locally for sandbox data
- Add a custom phrase & verify early return path
- Experiment toggling embedding to observe latency delta

---
End of Full Documentation v2.
