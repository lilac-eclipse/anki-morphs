## What is the current behavior?

Closes #370

Currently, AnkiMorphs automatically selects the target morph for each card based on which morphs are unknown. This prevents users from explicitly specifying which morph they want to learn, making AnkiMorphs incompatible with yomitan-based manual mining workflows that use dedicated target word fields.

## What is the new behavior?

Users can now specify separate fields for knowledge tracking and difficulty evaluation in the note filter configuration:

- **Read Field**: Extracts morphs for knowledge tracking (e.g., "Target Word" field with explicit morph)
- **Modify Field**: Extracts morphs for difficulty evaluation (e.g., "Sentence" field with full context)

This enables users to:
1. Use yomitan-based manual mining workflows with note types like Lapis (separate "Target Word" and "Example Sentence" fields)
2. Explicitly declare which morph they're learning via a dedicated field
3. Have difficulty evaluation based on full sentence context (maintaining i+1 principles)
4. Follow the Minimum Information Principle by focusing on a single learning target per card

**Example workflow:**
- Target Word: `聞かない` → system tracks knowledge of lemma `聞く`
- Sentence: `全然ウワサ聞かないね` → system evaluates card difficulty based on all sentence morphs
- Card is marked as appropriate difficulty if sentence has 1 unknown (the target word)

## What kind of changes does this PR introduce?

- [x] New feature (non-breaking change which adds functionality)
- [x] This change requires a documentation update

## Checklist:

- [x] My code successfully passes pre-commit
- [x] I have commented my code, particularly in hard-to-understand areas
- [x] I am willing to help create documentation/guides for the changes made in this PR
- [x] This PR can be rebased onto main without conflicts (create a backup branch before attempting a rebase)
- [x] I have squashed my commits into sensible portions
