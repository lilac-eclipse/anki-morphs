# Feature Proposal: Support for Note Types with Dedicated Target Word Fields (Yomitan/Lapis Mining)

## Problem

**Current limitation**: AnkiMorphs is designed for bulk sentence mining (sub2srs-style decks) and cannot read dedicated target word fields. It only infers target morphs by analyzing sentences.

**Impact**: This makes AnkiMorphs incompatible with some yomitan-based manual mining workflows, the approach recommended by Donkuri, TheMoeWay, and other major Japanese learning guides. Learners using popular note types like [Lapis](https://github.com/donkuri/lapis) (which separate "Target Word" and "Example Sentence" fields) cannot use AnkiMorphs effectively.

**What's needed:**
- Read target morph from a dedicated field (e.g., "Target Word")
- Track knowledge based on user-specified target morphs
- Evaluate card difficulty based on example sentence for i+1 ordering

## Use Case

**Card structure:**
- **Target Word field**: Target morph user wants to learn (e.g., `聞く`)
- **Example Sentence field**: Example sentence containing that morph (e.g., `全然ウワサ聞かないね`)

**Desired behavior:**
- User specifies `聞く` as the target morph (via `Target Word` field)
- System tracks whether user knows `聞く`
- System evaluates difficulty based on `Example Sentence` field (i+1 ordering)
- Card success/failure tied exclusively to target morph `聞く`

## Value Proposition

### 1. Enables Yomitan Mining Workflows

Mainstream Japanese learning guides (Donkuri, TheMoeWay) recommend manual mining with yomitan into word+sentence card formats (following SuperMemo's [minimum information principle](https://www.supermemo.com/en/blog/twenty-rules-of-formulating-knowledge)). Popular note types like [Lapis](https://github.com/donkuri/lapis) use separate "Target Word" and "Example Sentence" fields.

**AnkiMorphs currently cannot track the target word** because it only analyzes sentences and has no way to read the target word field.

This feature adds compatibility by:
- Reading target morphs from the "Target Word" field (tracks what you're actually mining)
- Using the "Example Sentence" for i+1 difficulty evaluation (maintains context benefits)

### 2. Enhanced Vocab Deck + Mining Deck Integration

With manual target specification, combining a vocab deck (Kaishi 1.5k, Core 2.3k) with mining becomes more effective. AnkiMorphs can automatically prioritize **i+1 mined cards → vocab cards → i+n mined cards**.

**Setup:**
- Position vocab deck cards after i+1 cards (due position of 1500000+)
- Set vocab note type to read-only in AnkiMorphs
- Result: i+1 mined cards appear first, vocab cards fill gaps, harder mined cards wait

**Why this matters:** You're mining specific words from content you care about. When AnkiMorphs tracks those exact target words (instead of inferring from sentences), prioritization becomes reliable. You learn what you mined for first, but fall back to vocab when mined cards are too difficult.


## Technical Implementation

### Proposed Mechanism: Separate Read/Modify Note Config Fields

**Configuration:**
- **Read field**: Field containing user-specified target morph (e.g., `Target Word`)
- **Modify field**: Field used for difficulty evaluation (e.g., `Example Sentence`)

**Behavior:**
- READ phase: Extract morphs from read_field only → populate knowledge database
- MODIFY phase: Extract morphs from modify_field → score card difficulty, but look up intervals from database (populated by morphs from read_field)

**Effect**: `Target Word` field determines what you're learning (knowledge tracking). `Example Sentence` field determines when you see it (i+1 ordering).

### Core Challenge

Separating read and modify fields creates inflection mismatches:

**Example:**
- Target Word: `聞く` → morphemizer extracts `('聞く', '聞く')`
- Sentence: `全然聞かなかった` → morphemizer extracts `('聞く', '聞かなかった')`

**Problem**: Current code looks up morphs by (lemma, inflection). When scoring the sentence, it searches for `('聞く', '聞かなかった')` but the knowledge database only contains `('聞く', '聞く')` from the target word. Lookup fails.

**Solution**: Look up by lemma only (using MAX interval across inflections).


### Proposed Implementation

**Database Schema Addition:**

Add one new table to cache sentence morphs separately from knowledge morphs:

```sql
CREATE TABLE Card_Scoring_Morph_Map (
    card_id INTEGER,
    morph_lemma TEXT,
    morph_inflection TEXT,
    PRIMARY KEY (card_id, morph_lemma, morph_inflection)
)
```

Existing tables remain unchanged:
- `Morphs` - stores learning intervals (updated only by knowledge morphs)
- `Card_Morph_Map` - links cards to knowledge morphs (from read_field)

**READ Phase Workflow:**

1. **Cache knowledge morphs** from `read_field` (Target Word):
   - Extract morphs from isolated target words
   - Store in `Card_Morph_Map` (tracks which cards teach which morphs)
   - Update intervals in `Morphs` table based on card intervals

2. **Cache scoring morphs** from `modify_field` (Example Sentence) if different:
   - Extract morphs from full sentences
   - Store in `Card_Scoring_Morph_Map`
   - **Do not update intervals** (sentence morphs are for difficulty calculation only)

**MODIFY Phase Workflow:**

When `read_field != modify_field`:
- Query `Card_Scoring_Morph_Map` to get sentence morphs (e.g., `全然`, `聞く`, `ね` from sentence)
- JOIN with `Morphs` table to look up intervals using `evaluate_morph_inflection` setting:
  - If `True`: Match on (lemma, inflection) → finds exact inflection only
  - If `False`: Match on lemma only with MAX(interval) → finds any inflection of that lemma
- Result: Sentence morphs with intervals attached
  - Target morph `聞く` → interval 21 (found in `Morphs` from knowledge tracking)
  - Context morphs `全然`, `ね` → interval 0 (not in `Morphs`, unknown)
- Score card based on sentence morphs: 1 known + 2 unknown = moderate difficulty

**Example data flow for card with Target Word: `聞く`, Sentence: `全然聞かないね`:**

```
Card_Scoring_Morph_Map          Morphs                    Result
┌──────┬────────┐              ┌──────┬──────────┐       ┌──────┬──────────┐
│ card │ morph  │   JOIN       │ lemma│ interval │   →   │ morph│ interval │
├──────┼────────┤   ───→       ├──────┼──────────┤       ├──────┼──────────┤
│ 1001 │ 全然   │              │ 聞く  │ 21       │        │ 全然 │ 0        │
│ 1001 │ 聞く   │              └───────┴──────────┘       │  聞く │ 21       │ ← Found!
│ 1001 │ ね     │                                         │  ね  │ 0        │
└──────┴────────┘                                        └──────┴──────────┘
   Sentence morphs          Knowledge morphs only        Sentence morphs with
   (for scoring)            (from Target Word)           intervals for scoring
```


## Open Questions

### Morphemizer Lemma Consistency Across Contexts

Morphemizers may produce different lemmas when processing isolated words versus sentence contexts:

**Example scenario:**
- Target Word: `聞く` → morphemizer produces `('聞く', '聞く')`
- Example Sentence: `全然聞かなかった` → morphemizer might produce `('聞か', '聞かなかった')` (stem form)

**Potential issue**: If the knowledge database contains lemma `聞く` but the sentence morphemizer produces lemma `聞か`, even lemma-only lookups would fail.

**Status**: Needs empirical testing with representative examples across different morphemizers (spaCy, MeCab, etc.) to determine if this occurs in practice.

## Request

Seeking feedback on whether this is:
1. A reasonable/valuable feature for morph-based learning
2. Architecturally compatible with AnkiMorphs without major refactoring
3. Feasible to implement as proposed
