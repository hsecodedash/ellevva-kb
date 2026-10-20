# 07a — Optional next step: Atlas Search + Vector Search for profiles

What ships today (no Atlas dependency): `Profile.skillsNormalized` keys + the alias dictionary in `utils/skillAliases.js` + dictionary typo fallback (edit distance) + ranking by number of matched skills. See plans/07 §C.

This doc covers the upgrade path once the DB (Atlas cluster `elevva`) is confirmed to be on a tier that supports Atlas Search (M10+ is recommended for production load; M0/M2/M5 allow a limited number of search indexes).

## 1. Atlas Search index (fuzzy + synonyms)
Create a synonyms source collection `skill_synonyms`, generated from `SKILL_GROUPS`:
```json
{ "mappingType": "equivalent", "synonyms": ["sharepoint", "spo", "sharepoint online", "share point"] }
```
Index `profiles_search` on collection `profiles`:
```json
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "organizationId": { "type": "objectId" },
      "isActive": { "type": "boolean" },
      "techStack": { "type": "string", "analyzer": "lucene.standard" },
      "skills": { "type": "document", "fields": { "name": { "type": "string" } } },
      "currentRole": { "type": "string" },
      "experience": { "type": "document", "fields": { "title": { "type": "string" } } },
      "skillsNormalized": { "type": "token" }
    }
  },
  "synonyms": [{ "name": "skills", "analyzer": "lucene.standard", "source": { "collection": "skill_synonyms" } }]
}
```
Query shape (it must be the first stage, so tenant scoping goes in `filter` — the tenantScope plugin's `$match` unshift would break `$search`; call it via `Profile.collection.aggregate` or add a plugin bypass):
```js
{ $search: {
  index: "profiles_search",
  compound: {
    filter: [{ equals: { path: "organizationId", value: orgId } }, { equals: { path: "isActive", value: true } }],
    should: terms.map((t) => ({ text: { query: t, path: ["skills.name", "techStack"], fuzzy: { maxEdits: 2, prefixLength: 2 }, synonyms: "skills" } })),
    minimumShouldMatch: 1,
  },
} }
```
Note: `fuzzy` and `synonyms` can't be combined in one `text` operator. Use two `should` clauses per term (one fuzzy, one synonyms).

## 2. Vector (semantic) search
- Per profile, embed `currentRole + skills + techStack + experience titles + summary` (e.g. OpenAI `text-embedding-3-small`, 1536 dims) from the AI server on create or update. Store it in `profileEmbedding`.
- Per requirement or pasted JD, embed it the same way.
- `$vectorSearch` index on `profileEmbedding` with `organizationId` as a filter field.
- Hybrid ranking: reciprocal rank fusion of the vector results and the keyword results, then send the top 50 to the AI server's `/jd-profile-match` for detailed scoring. This cuts OpenAI cost compared with scoring the full pool.

## Rollout
1. Run `scripts/backfillSkillsNormalized.js` (already needed for today's fix).
2. Create the synonyms collection and the search index, and put the `$search` path behind `SEARCH_ENGINE=atlas`. Fall back to the current path when the index is missing.
3. Add embeddings (backfill about 30k profiles: roughly $1–2 with small embeddings), then the vector index, then hybrid ranking.
