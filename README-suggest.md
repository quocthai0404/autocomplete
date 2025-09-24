# Suggest Keywords System (Autocomplete)

This document describes the design and implementation of the suggest keywords feature exposed via `GET /api/search/suggest_keywords`, along with related maintenance endpoints and data pipelines.

## Overview

The endpoint provides smart autocomplete suggestions by combining multiple sources:

- Prefix matches on the current token
- Semantic neighbors from co-occurrence (PPMI) vectors
- Next-token predictions (1 and 2-step contexts) and on-the-fly multi-token phrase expansion
- Admin-curated phrases (custom phrases)
- Embedding-based neighbors (optional)

Performance:
- Parallel fetching for key branches (prefix vs neighbor, phrase_dict vs next)
- 20s Redis cache keyed by normalized query + trailing-space flag
- Response includes `x-suggest-latency-ms` header

Online learning:
- Query and suggestion clicks can be used to learn online (optional service hooks included).

## API: GET /api/search/suggest_keywords

- Auth: protected by auth middleware
- Query params
  - `query`: string (required)
- Response shape
  ```jsonc
  {
    "success": true,
    "message": "Thành công",
    "data": [
      {
        "text": "tai nghe bluetooth",     // Display form (keeps diacritics)
        "token": "tai nghe bluetooth",    // Normalized token(s)
        "type": "phrase",                 // custom | phrase_dict | phrase | next | prefix | neighbor | embedding
        "score": 0.42,                     // raw score before normalization/weight
        "full": "tai nghe bluetooth"      // Full query after applying suggestion
      }
    ]
  }
  ```
- Headers
  - `x-suggest-latency-ms`: processing time in ms
  - `x-cache`: "hit" when served from Redis cache

## Algorithm

1) Validate query (zod schema). If invalid → 400.

2) Normalize & tokenize
- `normalizePreserveDiacritics` keeps display diacritics but normalizes spacing/case.
- `SegmentService.tokens` produces normalized tokens for logic.
- `endsWithSpace` determines whether the last token is complete.
- Derive `lastToken`, `prevToken`, `prevPrevToken` accordingly.

3) Cache check
- Cache key: `suggest:<normalized_query>:space=<0|1>`.
- If query effectively empty → return empty list and cache for 20s.
- If cached result exists → return it (with `x-cache: hit`).

4) Utilities
- `displayOf(token)` via `keywordVectorRepo.displayMapForTokens` → display form with diacritics.
- `makeFullForPrefix` and `makeFullForNeighbor/Next/Phrase` compose the final full query string.

5) Candidates from multiple sources (parallelized when possible)

- Prefix (ENABLE_PREFIX, `lastToken` exists)
  - `keywordVectorRepo.suggestByPrefix(lastToken, LIMITS.PREFIX)`
  - → `{ text=display||token, token, type='prefix', score=freq, full=makeFullForPrefix(text) }`

- Neighbor (ENABLE_NEIGHBOR, `prevToken` exists, and not disabled on space)
  - `keywordVectorRepo.getByToken(prevToken)` → top `LIMITS.NEIGHBOR` neighbors (PPMI)
  - → `{ text=displayOf(n.token), token=n.token, type='neighbor', score=n.score, full=append }`

- Custom phrases (admin-curated)
  - `customPhraseRepo.findMatches(typedTokens, partial, 10)`
  - Take the tail (suffix beyond the already typed tokens) if any
  - → `{ type='custom', text=display||full, token=tailTokens.join(" "), score=priority||1, full=(append/replace) }`

- When `endsWithSpace` = true (a token just completed)
  - Phrase dictionary (ENABLE_PHRASE_DICT)
    - If `prevPrevToken` exists: `keywordPhraseRepo.getByPrefix2(prevPrev, prev, LIMITS.PHRASE_DICT)`
    - Else: `getByPrefix1(prev, LIMITS.PHRASE_DICT)`
    - Tail beyond the prefix → `{ type='phrase_dict', text=tailDisp, token=tailTokens.join(" "), score=pmi, full=rawQuery + tailDisp }`
  - Next token (ENABLE_NEXT)
    - Context priority: `prevPrev|||prev`, fallback `prev`
    - → `{ type='next', text=displayOf(token), token, score=count|score, full=append }`
    - Phrase expansion (ENABLE_PHRASE):
      - Seed from top `PHRASE_FIRST_LIMIT` next tokens; expand up to `MAX_PHRASE_TOKENS` with breadth `PHRASE_SECOND_LIMIT` using 2-step contexts where possible
      - Score phrase = product of (score|count) along the path
      - → `{ type='phrase', text=join displayOf(tokens), token=join tokens, score, full=append }`

- When not `endsWithSpace` and prefix docs exist
  - For the top 1–2 prefix completions, find 1–2 next tokens to form a 2-token phrase immediately
  - → items type `phrase` using `makeFullForPrefix`

- When not `endsWithSpace` and no prefix docs
  - Use `lastToken` to find next tokens and form 2-token phrases

- Embedding neighbors (optional, `EMBEDDINGS.ENABLED`)
  - `embeddingNeighborService.neighborsForQuery(query, K)`
  - → `{ type='embedding', text=displayOf(token), token, score, full=append }`

6) Fast path (optional)
- If `FASTPATH_IF_CUSTOM` and there is at least one `custom` candidate → return only `[custom + phrase_dict]` (up to 10), cache for 20s.

7) Merge, dedupe, and rank
- Merge in the following order: `custom`, `phrase_dict`, `phrase`, `next`, `prefix`, `neighbor`, `embedding`.
- Dedupe by `full` (preferred) or `token|type`.
- Score normalization and weighting:
  - Normalizer: if `SCORE_NORMALIZER = 'log1p'` → `log1p(max(0, rawScore))`, else passthrough
  - Weight by type from `WEIGHTS` (defaults: custom 2.0, phrase_dict 1.35, phrase 1.1, next 1.0, prefix 0.9, neighbor 0.75, embedding 1.05)
  - `finalScore = normalized * weight`
- Pin up to 2 `custom` items, then fill with top-scored others to a max of 10.

8) Response and caching
- Set `x-suggest-latency-ms`, cache 20s under cache key, return payload.

## Configuration (suggest.config.ts)

- Toggles: `ENABLE_PREFIX`, `ENABLE_NEIGHBOR`, `ENABLE_NEXT`, `ENABLE_PHRASE`, `ENABLE_PHRASE_DICT`, `EMBEDDINGS.ENABLED`
- Limits: `LIMITS.PREFIX`, `LIMITS.NEIGHBOR`, `LIMITS.PHRASE_DICT`, `LIMITS.NEXT`
- Phrase expansion: `PHRASE_FIRST_LIMIT`, `PHRASE_SECOND_LIMIT`, `MAX_PHRASE_TOKENS`
- Behavior: `DISABLE_NEIGHBOR_ON_SPACE` (default true), `FASTPATH_IF_CUSTOM`
- Scoring: `SCORE_NORMALIZER` ('log1p' | 'none'), `WEIGHTS` per type
- Data sources for build: `SOURCE_FIELDS` (default ["title", "name"]) plus `STOPWORDS`, `SYNONYMS`

## Maintenance and Related Endpoints

### POST /api/search/rebuild_keyword_vectors

Builds co-occurrence vectors, next-token transitions, and mined phrases from product data.

1) Source & tokenization
- Read products with `published_on != null`, projecting `SOURCE_FIELDS` (default: title, name).
- Concatenate fields, segment tokens via `SegmentService.tokens`.
- Keep a display sample (with diacritics) for each normalized token.

2) Statistics
- `tokenFreq`: documents per token (use Set(tokens) per doc)
- `pairFreq`: document-level co-occurrence for token pairs
- `nextCounts`: sequential counts `prev -> next`
- `next2Counts`: two-step context counts `p1|||p2 -> next`
- `totalDocs`: total products scanned

3) Neighbors (PPMI)
- PMI(a,b) = `log( (cooc * totalDocs) / (fa * fb) )`
- PPMI = `max(0, PMI)`
- For each token with `freq >= minFreq` (default 3), keep top `maxNeighbors` (default 10), include `display`, `freq`, `neighbors`.
- Upsert to `keywordVectorRepo`.

4) Next-token (1-step, 2-step)
- For each `prev` (or `ctx = p1|||p2`), compute `score = count / total` and keep top ~12.
- Upsert to `keywordNextRepo`.

5) Phrase mining
- Seed bigrams from top co-occurring nexts per `prev`.
- `pushPhrase(tokens, count)` filters by `MIN_PHRASE_COUNT` (default 3) and PMI ≥ `MIN_PHRASE_PMI` (default 2.0) using the first pair in tokens.
- Expand to up to `PHRASE_MAX_NGRAM` (default 3) using nextCounts; branch limits keep the search bounded.
- Upsert to `keywordPhraseRepo` as `{ phrase, tokens[], count, pmi }`.

6) Result
- Returns `{ tokens: <#docs> }`. Parameters: `{ minFreq = 3, maxNeighbors = 10 }`.

### POST /api/search/rebuild_dictionary

Builds a spell dictionary to support typo suggestions.

1) Extract tokens via Mongo aggregation
- Project `title`, `alias`, `brand.text`, `vendor`, `tags` from products
- Reduce to a single text string; split by spaces; unwind tokens; drop empty; group and count

2) Normalize and merge frequencies
- `normalizeBasic` per token → term
- Merge counts by normalized term, keep a single `original` sample with diacritics

3) Bulk upsert into `spell_dictionary`
- Entry: `{ term, original, freq }`
- Utility methods:
  - `isKnownToken(term)`, `getByTerm(term)`, `topCandidatesForToken(token, limit)`

4) Usage in typo suggest
- Levenshtein-based ranking (distance threshold depends on token length)
- Prefer lower distance and higher frequency
- Beautify final suggestion by mapping normalized terms back to `original` with diacritics

## Analytics and Online Learning

- POST `/api/search/suggest_click`: record user selection `{ q, chosen, token, type }`
- GET `/api/search/suggest_stats`: view counts per suggestion type
- Online learning hooks
  - After successful search: `onlineLearnerService.learnFromQuery(query)`
  - After suggest click: `onlineLearnerService.learnFromClick(q, token|chosen)`

## Edge Cases and Notes

- Empty query → returns empty data (cached)
- `DISABLE_NEIGHBOR_ON_SPACE = true` → neighbor suggestions are suppressed immediately after a space
- Self-loop avoidance: `next` suggestions filter out `token === prev`
- Display vs token: display keeps diacritics; token is normalized

## Example Requests (optional)

- GET `/api/search/suggest_keywords?query=tai%20ng`
- GET `/api/search/typo_suggest?query=tai%20ngeh`
- POST `/api/search/rebuild_keyword_vectors`
- POST `/api/search/rebuild_dictionary`

---

For implementation references, see:
- `src/controllers/search.controller.ts` (main logic)
- `src/services/keyword_vector_builder.service.ts` (build pipeline)
- `src/services/spell_suggest.service.ts` and `src/repositories/search/spell_dictionary.repository.ts` (typo)
- `src/repositories/search/*` (keyword_vector, keyword_next, keyword_phrase, custom_phrase)
- `src/config/suggest.config.ts` (tuning)
