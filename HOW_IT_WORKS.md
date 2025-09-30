# PEPE Suggest & Spell – HOW IT WORKS (Tóm tắt nhanh)

> File này là bản tóm tắt thực thi. Tài liệu đầy đủ xem ở `README.md` (nguồn duy nhất chuẩn hoá). Nếu có khác biệt: README thắng.

## 1. Mục tiêu & Nguyên tắc
- Giữ **độ chính xác dấu tiếng Việt** cao – không phá dấu.
- Kết quả gợi ý ổn định (zero-diff) khi bật/tắt tối ưu an toàn.
- Mở rộng dễ: thêm nguồn (phrase dict, embeddings) không ảnh hưởng core.
- Hiệu năng ms-level qua chiến lược: narrowing sớm → mở rộng có kiểm soát → cache nhiều tầng.

## 2. Kiến trúc Tổng Quan
```
Client → Hono HTTP API
    ├─ SuggestController → SuggestService (multi-source pipeline)
    ├─ SpellController   → SpellSuggestService (chính tả + beautify)
    │
    ├─ Redis: cache suggest, snapshot spell, index custom, version keys
    ├─ MongoDB: custom phrases, phrase dict (PMI), next-token (n-gram), overrides
    ├─ Worker: warming (snapshot + custom index), rebuild background
    └─ (Optional) Embedding / Vector service (semantic)
```

## 3. Dòng Chảy Request
### 3.1 Suggest (`/api/suggest?q=`)
1. Normalize: tạo `preserve` (giữ dấu, lowercase) + `fold` (bỏ dấu, đ→d).
2. Redis cache check (key có version custom).
3. Nếu miss → chạy pipeline:
   - Custom phase: index narrowing bằng ký tự/tiền tố; augmentation khi ít.
   - Nếu query kết thúc space: load phrase dict + next-token + branch expansion.
   - Nếu chưa có space cuối: fallback partial completion cho token cuối.
   - (Optional) Embedding neighbors.
   - Merge + dedupe + scoring + sort + trim N.
4. Lưu cache + trả kết quả + headers timing (nếu bật).

Early exit nếu đạt `CUSTOM_EARLY_THRESHOLD` và `CUSTOM_RETURN_MODE=custom-first`.

### 3.2 Spell (`/api/spell?q=`)
1. Tokenize + áp overrides (protect / replace).
2. Với token cần xét: prefix pool → bổ sung topTokens → prune theo length → early-exit Levenshtein.
3. Chọn best theo (khoảng cách ↑, freq ↓).
4. Beautify: khôi phục dấu đúng qua `originalForm`; nếu user đã nhập form có dấu hợp lệ và khớp fold → giữ nguyên.
5. Trả `did_you_mean` nếu khác bản gốc.

## 4. Cấu Trúc Dữ Liệu Chính
### 4.1 Spell Snapshot
| Trường | Vai trò |
|--------|--------|
| tokenFreq | Tần suất token fold → dùng known check + ranking |
| topTokens | Seed bảo vệ / fallback pool |
| originalForm | foldKey → 1 dạng có dấu đẹp |
| prefixIndex | prefix (1–3 ký tự) → candidate list giảm phạm vi |
| lengthBuckets | Buckets theo độ dài để prune |
| version/meta | Theo dõi & rebuild khi thiếu field mới |

### 4.2 Custom Phrase Index
| Map | Mô tả |
|-----|------|
| byFirstChar / foldByFirstChar | Lấy nhanh theo ký tự đầu (có & không dấu) |
| byFirstTok2 / foldByFirstTok2 | Hẹp hơn theo 1–2 ký tự đầu token đầu |

### 4.3 Kho Dự Đoán
- phrase dict (PMI): cụm tail nhiều người gõ → expansion.
- next-token (bigram/trigram): dự đoán token tiếp theo.
- embedding (tuỳ chọn): bổ sung ngữ nghĩa.

## 5. Chiến Lược Hiệu Năng
| Kỹ thuật | Mục đích |
|----------|---------|
| Prefix narrowing | Giảm candidate ngay đầu |
| Early exit custom | Trả sớm khi đủ kết quả quan trọng |
| LRU next-token | Tránh truy vấn lặp context |
| Map reuse (version-guard) | Giảm alloc & GC |
| Time budget expansion | Cắt nhánh sâu nếu tốn thời gian |
| Precompute scoring | Giảm phép toán lặp |

Tất cả đều phải zero-diff output (so với baseline) khi bật ở chế độ an toàn.

## 6. Scoring Rút Gọn
```
rawScore = baseFreq/heuristics
norm = normalizer(rawScore) // vd: log1p
final = norm * weight(type) + priorityBoost(optional)
```
Custom ưu tiên; match chính xác đầu câu + length hợp lý được bonus.

## 7. Cache Tầng
| Lớp | Nội dung |
|-----|---------|
| In-process | Reuse map / index trong vòng đời process |
| LRU (RAM) | Kết quả next-token context |
| Redis ngắn | Kết quả suggest (20s), danh sách custom (60s), index (~5m) |
| Redis dài | Spell snapshot (30d) |

Invalidation: bump version key là đủ (không cần xoá từng entry).

## 8. Integrity Dấu & Overrides
- `originalForm` đảm bảo tái tạo chính xác dấu chuẩn.
- Overrides loại `protect` chặn sửa; `replace` cưỡng bức thay.
- Không sửa token dài ≤2 hoặc đã known.

## 9. Headers (Khi bật timing)
| Header | Ý nghĩa |
|--------|--------|
| x-suggest-latency-ms | Tổng thời gian xử lý |
| x-prof-custom-scan-ms | Thời gian quét custom |
| x-prof-custom-candidates | Số candidate thô |
| x-early-exit-stage | Giai đoạn dừng sớm (nếu có) |

## 10. Roadmap Ngắn
| Mục | Trạng thái |
|-----|-----------|
| Map reuse | Done |
| LRU next-token | Done |
| Scoring precompute | Done |
| Top-K heap | Pending |
| Prefix incremental reuse | Pending |
| Instrumentation+ | Pending |
| Async embedding | Pending |

## 11. Playbook Nhanh
| Tình huống | Hành động |
|-----------|----------|
| Thêm custom phrase | Insert DB → bump version → warm query |
| Snapshot thiếu field | Rebuild (endpoint hoặc xoá key) |
| Latency tăng | Bật timing → khoanh phase → giảm limit/budget |
| Over-correction | Giảm SPELL_MAX_DISTANCE hoặc tăng CORRECTION_TRIGGER_MIN_RESULTS |

## 12. Nguyên Tắc Thiết Kế
1. Zero-diff trước – tối ưu sau.
2. Narrow trước, expand sau.
3. Dấu là chuẩn vàng (accent integrity) – không hi sinh.
4. Stateless logic, state version qua Redis.
5. Flag-driven rollout & rollback.

---
Tài liệu chi tiết (thuật toán đầy đủ, bảng config, flows mở rộng) xem `README.md`.
