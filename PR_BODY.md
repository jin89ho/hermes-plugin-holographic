# fix(auto_extract): drop `want`/`need` from the English preference pattern

Target repo: `itpartypattaya/hermes-plugin-holographic` (the catalog-maintained
copy; the catalog pins 0.17.4 at sha 827c639, which is this branch's base).

## The bug

`auto_extract` harvested a model's own reasoning sentences into the store as
`user_pref` facts. The English preference pattern is:

```python
re.compile(r'\bI\s+(?:prefer|like|love|use|want|need)\s+\S', re.IGNORECASE)
```

`want`/`need` state a **one-off intent**, not a durable preference. The real
failure mode (observed in a live store, 2026-10): a user pastes a model's
English draft to have it translated. That draft arrives as a `role="user"`
message, so extraction runs on text the **model** wrote:

- `Plus, I want to ask you about details like the intended date and format …`
- `So I need to find the 2023 production.`
- `Wait, but I need the 2023 production for B.`

All three matched `I want/need …` and were stored as `user_pref`. They state no
preference, carry no keywords (so they never help recall), and pollute the
store. The question filter (`_RE_QUESTION`) does not catch them — they are
declarative.

## The fix

Drop `want` and `need` from the pattern; keep `prefer|like|love|use`. The
durable-preference signal survives ("I prefer X over Y", "I use Z daily");
`want`/`need` almost always state transient intent, in the user's own words as
much as in a pasted model draft.

## Tests

`tests/test_holographic_extract_want_need.py` reproduces the exact three
sentences from the live store:

- **Red on base** (verified by stashing the fix): `So I need to find the 2023
  production.` is stored as a fact.
- **Green with the fix**: all three dropped; `I prefer short answers with
  sources.` and `I use Dark mode in every editor on my machines.` are still
  extracted, so durable preferences are unaffected.

## Note on the surrounding pattern

The Russian pattern keeps `хочу` ("want"). I left it untouched: no live-store
evidence for the same failure in Russian, and I did not want to over-reach. The
English change is the one with a concrete repro. Happy to mirror it if a Russian
case surfaces.
