# PEPE Suggest & Spell Platform – Full Technical Documentation

> Single authoritative README (supersedes previous HOW_IT_WORKS.md & README_NEW.md). Cực chi tiết: kiến trúc, flow, config, API, hiệu năng, vận hành, bảo trì, mở rộng.

## 1. Mục tiêu hệ thống
Hệ thống phục vụ gợi ý từ khóa & sửa lỗi chính tả tiếng Việt có dấu với các nguyên tắc:
- Độ chính xác dấu tiếng Việt cao, không “bẻ dấu” sai.
- Thứ tự & nội dung gợi ý ổn định (zero-diff khi bật tối ưu hiệu năng an toàn).
- Dễ mở rộng: thêm nguồn dữ liệu (embedding, synonyms, phrase dict) không phá vỡ core.
- Hiệu năng: đáp ứng ms-level nhờ progressive narrowing + caching nhiều tầng.

## 2. Kiến trúc tổng quan
```
Client ─HTTP─> Hono Server (app.ts) ──> Suggest Controller / Spell Controller
                                   │
                                   ├─ Redis (cache, snapshot, versions)
                                   ├─ MongoDB (phrases, next tokens, snapshot raw)
                                   ├─ Worker warming (register.ts)
                                   └─ (Optional) Embedding Service / Vector Store
```
### 2.1 Thành phần chính
| Thành phần | Mô tả |
|------------|------|
| Hono App | HTTP server: middlewares (CORS, compress, csrf), route `/api/*`, swagger docs |
| SuggestService | Xử lý pipeline gợi ý đa nguồn với early exit và caching |
| SpellSuggestService | Sửa chính tả & beautify tiếng Việt, bảo toàn dấu |
| SpellDictionarySnapshotService | Build/upgrade snapshot (tokenFreq, index, originalForm...) |
| Custom Phrase Index | In-memory + Redis backed narrowing cấu trúc inverted maps |
| Repositories | Truy cập Mongo: phrase dictionary, next tokens, vector, overrides |
| Redis | Cache kết quả suggest, spell snapshot, index, version invalidation |
| Worker Register | Warming (spell snapshot + custom index) + queue consumers |

### 2.2 Luồng khởi động (Startup Flow)
```
server.ts
  └─ import './workers/register'
       ├─ warmSpellDictionary()
       ├─ warmCustomIndex()
       ├─ processQueue()
       └─ processEventQueue()
app.ts
  └─ mount routes + swagger
```
Mục tiêu: giảm cold latency và tránh build snapshot trong request đầu.

## 3. Request Lifecycle (Suggest)
```
HTTP /api/suggest?q=<query>
  │
  ├─ Parse & normalize (fold + preserve)
  ├─ Cache lookup (Redis) theo key versioned
  │   └─ Hit → trả (attach headers)
  └─ Miss → pipeline:
        1. Custom Phase (index narrowing → scan → expansion → contains)
           * Early exit nếu đủ số lượng & mode cho phép
        2. Nếu query kết thúc bằng space: Phrase Dict + Next-token + Branch expansion
        3. Nếu chưa space cuối: Fallback phrase (dự đoán phần còn lại token cuối)
        4. (Optional) Embedding neighbors
        5. Merge + Dedupe + Weighting + Priority Boost + Sort + Trim
        6. Cache save + Headers + Response
```
### 3.1 Cache Key Format
`suggest:v<customVersion>:<normalizedPreserveQuery>:space=<0|1>`

### 3.2 Early Exit Conditions
- `PERF.CUSTOM_EARLY_THRESHOLD` đạt và `CUSTOM_RETURN_MODE = custom-first`.
- Custom candidate >= `TOTAL_MAX_RESULTS`.

### 3.3 Dedupe Strategy
- Dựa trên canonical form (full phrase lowercase) hoặc `(tokens|type)` tùy nguồn.
- Custom ưu tiên giữ nguyên.

## 4. Request Lifecycle (Spell)
```
HTTP /api/spell?q=<query>
  │
  ├─ Tokenize & apply overrides (protect / replace)
  ├─ Với mỗi token cần xem xét:
  │     1. Skip (len <=2 / trong protect / đã known)
  │     2. Lấy prefixIndex pool (cut MAX_PREFIX_POOL)
  │     3. Bổ sung topTokens nếu pool nhỏ
  │     4. Length pruning
  │     5. Early-exit Levenshtein (maxDist động)
  │     6. Chọn bestTerm (dist asc, freq desc)
  ├─ Beautify (originalForm / override display / giữ dấu người dùng nếu hợp lệ)
  └─ Kết hợp thành did_you_mean (null nếu không khác)
```
### 4.1 Early-Exit Levenshtein
- Row-level pruning: nếu minRow > maxDist → break.
- Length gate: |len(a)-len(b)| > maxDist → skip.

### 4.2 Accent Integrity
- Nếu user version có dấu đúng normalization trùng với candidate không dấu → ưu tiên giữ bản có dấu user.
- `originalForm` mapping đảm bảo phục hồi form đẹp (ví dụ “điện thoại”, không phải “dien thoai”).

## 5. Data Structures
### 5.1 Spell Snapshot
| Field | Vai trò |
|-------|---------|
| tokenFreq | Tần suất token normalize (fold + lowercase) |
| topTokens | Pool seed & protect baseline |
| originalForm | foldKey → 1 canonical accented form |
| prefixIndex | prefix (1–3 ký tự) → candidate list |
| lengthBuckets | length → token list (để prune) |
| version / built_at / size_bytes | Meta theo dõi |

Auto-upgrade rebuild nếu thiếu field mới.

### 5.2 Custom Phrase Index (Suggest)
| Map | Nội dung |
|-----|----------|
| byFirstChar | char đầu → danh sách phrase ids/entries |
| byFirstTok2 | 1-2 ký tự đầu token đầu |
| foldByFirstChar | phiên bản fold để match không dấu |
| foldByFirstTok2 | tương tự nhưng fold |

### 5.3 Next / Phrase Dict Data
| Repo | Mục đích |
|------|----------|
| keywordNextRepo | bigram/trigram → danh sách tokens tiếp theo (freq/score) |
| keywordPhraseRepo | PMI / n-gram tail phrases |
| vectorRepo | tokenId → text hiển thị (beautify phrase dict) |

### 5.4 Embedding (Optional)
| Thành phần | Mô tả |
|------------|------|
| embeddingNeighborService | Lấy K vector gần nhất (semantic) |
| disable length gate | `PERF.DISABLE_EMBEDDING_MAX_LEN` |

## 6. Normalization & Matching
| Loại | Mô tả |
|------|------|
| Preserve | Lowercase giữ dấu (dùng hiển thị base) |
| Fold | Bỏ toàn bộ dấu + đ→d (matching không dấu) |
| Tokens | Segmenter cắt từ input (chưa fold) |

Matching custom:
- Token prefix: mỗi query token phải là prefix của phrase token tương ứng.
- Fold contains: cho phép query không dấu tìm phrase có dấu.
- Contains augmentation: thêm phrase chứa chuỗi đầy đủ khi candidate ít.

## 7. Scoring & Ranking (Suggest)
### 7.1 Base Scoring Components
| Thành phần | Ảnh hưởng |
|-----------|-----------|
| priority | Điểm ưu tiên manual (custom) |
| lengthBonus | Ưu tiên phrase dài hợp lý (không spam) |
| startsBoost | Boost nếu phrase bắt đầu chính xác query |

### 7.2 Final Score Formula
```
normalizedScore = normalizer(rawScore)  // ví dụ log1p
weighted = normalizedScore * weight(type)
if priorityBoostEnabled: weighted += priority * CUSTOM_PRIORITY_BOOST
```
### 7.3 Merge Modes
| Mode | Hành vi |
|------|---------|
| custom-first (default) | Early exit cho phép; non-custom chỉ bổ sung nếu thiếu quota |
| FULL_MERGE_MODE | Gom tất cả nguồn trước rồi trim |

## 8. Caching Chi Tiết
| Lớp | Giải thích |
|-----|-----------|
| In-process ephemeral | Map reuse trong request (avoid rebuild) |
| Process-wide LRU | Next-token results (context key) |
| Redis Short TTL | Suggest output (20s), custom phrases list (60s), index (~5m) |
| Redis Long TTL | Spell snapshot (30d) |

Invalidation: thay đổi custom phrases → bump `suggest:custom_version_v1` → cache miss.

## 9. Redis Keys
| Key Pattern | TTL | Mục đích |
|-------------|-----|----------|
| `spell:dict:latest` | 30d | Spell snapshot JSON |
| `suggest:custom_version_v1` | 1d | Version marker (bump để vô hiệu hóa cache) |
| `suggest:custom_phrase_index:v2` | ~5m | Serialized index cache |
| `suggest:custom_phrases:all_v2` | 60s | Danh sách custom phrases full |
| `suggest:v<ver>:<norm>:space=0|1` | 20s | Kết quả suggest |

## 10. Config Reference (Full)
### 10.1 Spell (`spell.config.ts`)
| Field | Default | Role | Tuning |
|-------|---------|------|--------|
| ENABLE_SPELL_DICT | true | Bật pipeline | Tắt khi bảo trì |
| ENABLE_TELEX_CORRECTION | true | Hook tương lai | Giữ |
| ENABLE_DEBUG_LOGS | false | Verbose debug | Bật tạm |
| SPELL_MAX_DISTANCE | 2 | maxDist len>4 | 3 nếu cần recall |
| SPELL_PER_TOKEN_LIMIT | 5 | (dự phòng) | Giữ |
| CORRECTION_TRIGGER_MIN_RESULTS | 0 | Ngưỡng min candidate để apply | >0 giảm over-correct |
| APPLY_IF_SCORE_BETTER | 0 | Gate (nếu scoring thêm) | 0 |
| MAX_CANDIDATE_POOL | 150 | Pool cuối distance | 120–180 |
| MAX_PREFIX_POOL | 400 | pool prefixIndex | Giảm nếu CPU cao |

### 10.2 Suggest Core Toggles (`suggest.config.ts`)
| Field | Default | Mô tả |
|-------|---------|-------|
| ENABLE_PREFIX | true | Prefix items (nếu tồn tại) |
| ENABLE_NEIGHBOR | true | Neighbor source |
| ENABLE_NEXT | true | Next-token predictions |
| ENABLE_PHRASE | true | Branch expansion |
| ENABLE_PHRASE_DICT | true | Phrase dictionary |
| FULL_MERGE_MODE | true | Merge toàn nguồn rồi trim |
| FULL_MERGE_MAX | 8 | Cap merge sơ bộ |
| CUSTOM_MAX_RETURN | 50 | Giới hạn custom trước merge |
| CUSTOM_PHRASE_CACHE_TTL | 60 | TTL list custom |
| CUSTOM_RETURN_MODE | custom-first | Early exit strategy |
| SOURCE_FIELDS | ['name','title'] | Build offline ref |

### 10.3 Phrase Expansion & Limits
| Field | Default | Vai trò |
|-------|---------|--------|
| PHRASE_FIRST_LIMIT | 3 | Seed width |
| PHRASE_SECOND_LIMIT | 2 | Fanout depth>1 |
| MAX_PHRASE_TOKENS | 3 | Max appended |
| PHRASE_MAX_NGRAM | 3 | PMI tra cứu max |
| MIN_PHRASE_COUNT | 2 | Lọc tần suất |
| MIN_PHRASE_PMI | 1.5 | Lọc cohesion |

### 10.4 Retrieval Limits
| Field | Default | Applies |
|-------|---------|--------|
| LIMITS.PREFIX | 6 | prefixItems |
| LIMITS.NEIGHBOR | 6 | neighborItems |
| LIMITS.PHRASE_DICT | 6 | phraseDictItems |
| LIMITS.NEXT | 8 | nextTokenItems + seeds |

### 10.5 Scoring & Embeddings
| Field | Default | Vai trò |
|-------|---------|--------|
| WEIGHTS.* | varies | Type weighting |
| SCORE_NORMALIZER | log1p | Compress outliers |
| CUSTOM_PRIORITY_BOOST | 0 | Priority-based bump |
| EMBEDDINGS.ENABLED | false | Bật semantic neighbors |
| EMBEDDINGS.K | 8 | Top K neighbors |

### 10.6 PERF Flags
| Field | Default | Tác dụng |
|-------|---------|--------|
| CUSTOM_EARLY_THRESHOLD | 6 | Early exit custom |
| DISABLE_EMBEDDING_MAX_LEN | 2 | Skip embed if short |
| ENABLE_PHASE_TIMING | false | Emit timing headers |
| PHRASE_BUILD_TIME_BUDGET_MS | 25 | Branch time budget |
| CUSTOM_INDEX_ENABLED | true | Bật index narrowing |
| CUSTOM_INDEX_FALLBACK_MIN | 30 | Fallback full list |
| CUSTOM_INDEX_MEM_TTL | 180 | TTL in-process index |
| CUSTOM_MAP_REUSE | true | Reuse grouping maps |
| NEXT_LRU_ENABLED | true | LRU next-token |
| NEXT_LRU_SIZE | 400 | Capacity |
| TOTAL_MAX_RESULTS | 8 | Final result size |

### 10.7 Optional Normalization Assets
| Asset | Vai trò |
|-------|--------|
| STOPWORDS | Loại bỏ/no-weight tokens |
| SYNONYMS | Normalize synonyms vào canonical form |

## 11. Tuning Cheat Sheet
### Latency ↓
- Giảm `PHRASE_BUILD_TIME_BUDGET_MS`, `LIMITS.NEXT`.
- Bật early exit & giữ `CUSTOM_MAP_REUSE` + `NEXT_LRU_ENABLED`.

### Recall ↑
- Tăng `CUSTOM_MAX_RETURN`, `LIMITS.PHRASE_DICT`, `SPELL_MAX_DISTANCE`.

### Memory ↓
- Giảm `NEXT_LRU_SIZE`, shorten TTL caches.

### Ưu tiên Custom ↑
- Tăng `WEIGHTS.custom` hoặc `CUSTOM_PRIORITY_BOOST`.

### Giảm Over-Correction Spell
- Set `CORRECTION_TRIGGER_MIN_RESULTS > 0` hoặc giảm `SPELL_MAX_DISTANCE`.

## 12. API Overview
| Endpoint | Method | Query Params | Mô tả |
|----------|--------|--------------|------|
| `/api/suggest` | GET | q=string | Trả danh sách gợi ý |
| `/api/spell` | GET | q=string | Trả did_you_mean & meta |
| `/api/health` | GET | - | (Nếu có) healthcheck |
| `/api/rebuild-spell` | POST | secret? | (Tùy) rebuild snapshot |
| `/api/custom/invalidate` | POST | - | Trigger invalidation |

### 12.1 Suggest Response (ví dụ)
```json
{
  "query": "dien thoa",
  "results": [
    {"full": "điện thoại", "type": "custom", "score": 12.4, "priority": 5},
    {"full": "điện thoại samsung", "type": "phrase", "score": 10.2},
    {"full": "điện thoại iphone", "type": "phrase_dict", "score": 9.8}
  ],
  "meta": {
    "version": 42,
    "elapsed_ms": 7,
    "early_exit": false
  }
}
```
### 12.2 Spell Response (ví dụ)
```json
{
  "query": "an cam",
  "did_you_mean": "ăn cam",
  "tokens": [
    {"orig": "an", "corrected": "ăn", "distance": 0, "action": "beautify"},
    {"orig": "cam", "corrected": "cam", "action": "known"}
  ],
  "snapshot_version": 17
}
```

## 13. Headers & Instrumentation
Khi bật `PERF.ENABLE_PHASE_TIMING`:
| Header | Ý nghĩa |
|--------|--------|
| x-suggest-latency-ms | Tổng thời gian xử lý |
| x-prof-custom-scan-ms | Thời gian quét custom core |
| x-prof-custom-sort-ms | Thời gian sort custom |
| x-prof-custom-build-map-ms | Build grouping maps |
| x-prof-custom-candidates | # raw candidate trước lọc |
| x-prof-custom-match-raw | # match sau filter |
| x-prof-priority-mode | sorting mode (priority-first / score-first) |
| x-prof-custom-fold-aug | # entries thêm từ fold augmentation |
| x-prof-custom-expand-added | # expansion results |
| x-early-exit-stage | custom / phrase_dict_next / fallback_phrase |

Định hướng bổ sung: LRU hit/miss, map reuse count, embedding time.

## 14. Error Handling & Fallbacks
| Tầng | Lỗi | Fallback |
|------|-----|----------|
| Spell snapshot missing field | null/legacy snapshot | Auto rebuild mới |
| Custom index quá ít candidate | Dưới threshold | Load full custom list |
| Redis timeout | Cache miss | Đọc trực tiếp DB repos |
| Embedding service lỗi | Exception | Bỏ qua nguồn embedding |
| Phrase expansion quá time budget | > PHRASE_BUILD_TIME_BUDGET_MS | Dừng mở rộng, dùng partial |

## 15. Deployment Checklist
1. Staging: bật timing; ghi p50/p95.
2. Script diff output khi bật/tắt PERF flags (đảm bảo zero-diff ranking Top N).
3. Canary 10% traffic với weight thay đổi.
4. Giảm rủi ro rollback: flags thuần cấu hình → tắt là xong.
5. Theo dõi memory: kích thước LRU & index.

## 16. Observability Roadmap
| Item | Mục tiêu |
|------|---------|
| LRU metrics | Hiểu hit ratio |
| Snapshot size gauge | Cảnh báo > ngưỡng |
| Candidate count histogram | Tối ưu narrowing |
| Distance op counter | Phát hiện bùng nổ CPU |
| Phase timing percentiles | SLA tuning |

## 17. Security & Hardening
| Khía cạnh | Thực hành |
|----------|----------|
| Rate limiting | (Add middleware nếu bị abuse) |
| Input sanitation | Normalize & length cap trước xử lý |
| Rebuild endpoints | Yêu cầu secret / auth token |
| Redis credential | ENV secrets, không hard-code |
| Dependency audit | Thường xuyên `npm audit` |

## 18. Troubleshooting Guide
| Triệu chứng | Nguyên nhân khả dĩ | Hành động |
|-------------|--------------------|----------|
| Suggest chậm spike | Index chưa warm / Redis chậm | Kiểm tra warming log, redis latency |
| Spell không beautify dấu | `originalForm` thiếu | Rebuild snapshot; log missing mapping |
| Over-correction tiếng Việt | `SPELL_MAX_DISTANCE` cao | Giảm xuống 2 hoặc tăng CORRECTION_TRIGGER_MIN_RESULTS |
| Mất custom mới | Version chưa bump | Kiểm tra key `suggest:custom_version_v1` |
| Gợi ý mất đa dạng | FULL_MERGE_MODE=false + weights lệch | Bật full merge hoặc điều chỉnh WEIGHTS |
| CPU cao | MAX_PREFIX_POOL lớn / PHRASE_BUILD_TIME_BUDGET_MS cao | Giảm các tham số |
| Memory tăng | NEXT_LRU_SIZE quá lớn | Giảm hoặc flush LRU |

## 19. Roadmap (Chi tiết)
| Phase | Hạng mục | Mô tả | Trạng thái |
|-------|---------|-------|-----------|
| 1a | Map reuse | Reuse custom grouping maps | Done |
| 1b | Next-token LRU | Cache context queries | Done |
| 1e | Scoring precompute | Prebuild weight references | Done |
| 1c | Top-K heap | Thay full sort khi N nhỏ | Pending |
| 1d | Prefix incremental reuse | Giữ candidate list khi user thêm ký tự | Pending |
| 2a | Instrumentation+ | LRU hits, reuse counters, JSON meta | Pending |
| 2b | Async embedding | Chạy song song & merge muộn | Pending |
| 2c | Adaptive budgets | Điều chỉnh branch depth theo p95 | Pending |
| 3 | Contextual spell bigram | Tận dụng bigramFreq | Future |

## 20. Design Principles (Tóm tắt)
1. Zero-diff ưu tiên.
2. Narrowing trước, mở rộng sau.
3. Accent integrity > mọi tối ưu.
4. Stateless core, state qua Redis + snapshot version.
5. Feature flags để rollout an toàn.

## 21. Glossary
| Term | Giải thích |
|------|-----------|
| fold | Chuẩn hóa bỏ dấu + đ→d |
| beautify | Khôi phục form có dấu đẹp nhất |
| early exit | Kết thúc pipeline sớm khi đủ tiêu chí |
| expansion | Mở rộng phrase bằng next-token chaining |
| zero-diff | Tối ưu không đổi output cuối |
| PMI | Pointwise Mutual Information (độ kết dính tokens) |

## 22. Ví dụ Luồng (End-to-End)
### 22.1 Suggest: "điện tho"
1. Normalize: preserve="điện tho", fold="dien tho".
2. Cache miss.
3. Custom phase narrowing ra các phrase bắt đầu “điện” + fold “dien”.
4. Expansion thêm tail (nếu token cuối hoàn chỉnh). Ở đây token cuối chưa hoàn chỉnh → skip expansion multiple.
5. Early exit? Chưa đủ threshold.
6. Chưa space cuối → fallback phrase predict tiếp phần của "tho" (ví dụ “thoại”).
7. Merge + score + trim.
8. Cache store + headers.

### 22.2 Spell: "dien thoai ssamgsung"
1. Tokenize: ["dien", "thoai", "ssamgsung"].
2. Normalize & overrides (none).
3. "dien" → prefixIndex pool; dist=1 với “điện” beautify.
4. "thoai" → thành “thoại”.
5. "ssamgsung" → candidates thu được “samsung” (distance 2) high freq.
6. Output did_you_mean = "điện thoại samsung".

## 23. Developer Notes
| Chủ đề | Ghi chú |
|--------|--------|
| Thread safety | Node single-thread; index reuse cần version guard |
| Memory tuning | Theo dõi size(index) + LRU; dump process RSS khi deploy |
| Unicode | Xử lý NFC/NFD: đảm bảo fold chạy trên chuỗi chuẩn hóa |
| Telex extension | ENABLE_TELEX_CORRECTION future pipeline step |
| Tests (khuyến nghị) | Snapshot test top-K suggestions dưới các flag combos |

## 24. Maintenance Playbooks
### 24.1 Rebuild Spell Snapshot
1. Delete `spell:dict:latest` hoặc hit endpoint rebuild.
2. Chờ worker rebuild (log: "spell snapshot rebuilt").
3. Verify size & fields (prefixIndex, originalForm) trong Redis.

### 24.2 Add New Custom Phrase
1. Insert vào DB (collection custom_phrases).
2. Debounce invalidation → version bump.
3. Gọi suggest với query liên quan để warm.

### 24.3 Điều tra Over-Correction
1. Log per-token distance.
2. Check token presence trong protect set.
3. Có thể thêm token vào override protect.

### 24.4 Điều chỉnh Latency p95 cao
| Bước | Thao tác |
|------|----------|
| 1 | Bật phase timing headers |
| 2 | Xác định phase nặng (custom scan / expansion / next) |
| 3 | Giảm limit hoặc time budget tương ứng |
| 4 | Kiểm tra diff output (script) |

## 25. Future Ideas
- Bloom filter cho known tokens (spell) thay vì tokenFreq key lookup.
- Adaptive prefixIndex: dynamic shrink với frequency tiers.
- Weighted accent error model (ưu tiên sửa dạng phổ biến). 

## 26. License / Internal Use
Internal proprietary. Không phân phối bên ngoài.

## 27. Summary
Hệ thống gợi ý & sửa chính tả tiếng Việt: nhanh, chính xác dấu, dễ tuning. Kiến trúc module hóa, mọi tối ưu đặt trên nguyên tắc zero-diff & flag-driven. README này là nguồn duy nhất – cập nhật khi thêm field hoặc phase mới.

_End of README_
