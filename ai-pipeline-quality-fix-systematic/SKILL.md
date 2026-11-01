---
name: ai-pipeline-quality-fix-systematic
description: Systematic approach to fix AI pipeline quality issues — trace the full data flow, fix at both AI prompt level and code-level enforcement, update all parallel paths. Based on L05 三要素 fix (角色名/场景锚定/道具一致性).
version: 1.0.0
---

# AI Pipeline Quality Fix — Systematic Approach

## When To Use

When an AI pipeline step produces inconsistent or incorrect output that scoring/verification catches but never fully fixes. The key symptom: **scoring keeps finding the same class of bug across iterations**.

## The Pattern (5 Phases)

### Phase 0: Full Audit — Trace the Data Flow

Don't stop at "where the bug manifests." Trace where the data enters the pipeline:

1. **Ingress**: What data does the generating AI actually receive?
   - L05 bottleneck: AI received Chinese narration text but no character name mapping → hallucinated "Li Wei"
2. **Transit**: Each intermediate step that modifies the data
   - L05: 6 deepen steps each re-call AI with full prompt_en → step-by-step drift
3. **Merge/Combine**: Any step that combines multiple versions
   - L05: merge picked from 6 truncated ([:500]) versions → lost content
4. **Verify**: What the scoring system checks
   - L05: scoring had correct name map, generation didn't → cat-and-mouse game

### Phase 1: Fix the AI Prompt Layer (Prevention)

Inject the missing context into ALL AI prompts in the chain:

- **Generation prompt**: Add exact data the AI needs (name maps, anchor word rules)
- **Deepen/refine prompts**: Add "do not modify [specific element]" instructions
- **Merge/reconstruct prompts**: Add "preserve [specific element]" rules
- **Fix/repair prompts**: Add the same context so fixes are correct

**File checklist**: Search `system_prompt =` across all pipeline files. Every single one needs the additional context.

### Phase 2: Code-Level Enforcement (Safety Net)

AI prompts are "soft constraints" — the AI may not follow them. Add code-level "hard constraints":

1. **Write a consistency function** that:
   - Groups output by scene/group identifiers
   - Extracts the reference anchor value from the first item in each group
   - Forces all other items in the group to match
   - Checks known entity mappings (e.g., character name: Chinese→English)
   - Replaces incorrect values with correct ones
2. **Call it after every pipeline phase** that could introduce drift:
   - After initial generation
   - After merge/reconstruction
   - After auto-fix cycles

### Phase 3: Fix Truncation/Data Loss

Check for content truncation in intermediate processing:
- `[:500]` truncation loses ~60%+ of 150-250 word prompts → expand to `[:1500]`
- Also consider: JSON serialization loss, escaped character corruption

### Phase 4: Unify Scoring Across Paths

If the same data is scored by multiple paths with different prompts:
- Count dimensions (6-dim vs 7-dim, 16-dim vs 17-dim)
- Check weight values
- Unify to the more complete version
- The auto-fix loop and the pipeline check must use the same criteria

### Extra: Fix API Response Parsing

When switching models (e.g., DeepSeek V4 Pro), check if the API response format changed:
- **DeepSeek V4 Pro** returns `reasoning_content` field alongside `content` — if the code only reads `content`, it gets empty/None
- Fix: In `_call_deepseek`, try `response.choices[0].message.content` first, then fall back to `reasoning_content`
- This bug manifests as: API returns 200, no errors, but output is empty — looks like "generation failed"

### Phase 5: Backward Compatibility

- Backup all modified files to `releases/` before starting
- Run syntax check after each file modification
- Restart service and verify import chain works
- Unit test the consistency function

### Phase 6: Verification — Inspect Every Shot, Not Just the Score

**The biggest trap: treating the score as truth.** The scoring system catches different issues each round — same data can score 85 one run and 69 the next because different dimensions are weighted differently. Score alone is misleading.

**Stop after each round.** Never let a test script auto-run multiple rounds. Run one round → present full output → discuss → decide whether to continue.

**Per-shot 三要素 checklist:**

For every shot in the output, verify:
1. **人 (Character)** — Is there at least one character? If not, is it intentional (empty atmosphere shot)?
   - Check both main characters AND side characters (反派、路人等)
   - An ECU of "her hand" counts — it's implied character presence
   - An empty city night shot may be correct as a transition
2. **景 (Scene/Setting)** — Is the scene clearly anchored?
   - Check for key location keywords in the prompt
   - Scene changes (律师事务所→公寓→车库) should match the narrative
3. **物 (Props)** — Are there at least 2-3 identifiable objects?
   - Count distinct prop keywords per shot
   - Check for consistency across shots

**Sample verification script:**
```python
for s in shots:
    prompt = s['prompt_en']
    has_char = any(name in prompt for name in ['Qin Lie', 'Han Yu'])
    has_side = bool(re.search(r'\b(man|woman|figure|someone|burly)\b', prompt, re.I))
    scene_kws = ['office', 'bedroom', 'garage', 'apartment', 'street', 'city']
    has_scene = any(kw in prompt.lower() for kw in scene_kws)
    prop_kws = ['desk', 'lamp', 'cup', 'pen', 'document', 'car', 'phone']
    prop_count = sum(1 for kw in prop_kws if kw in prompt.lower())
    if not (has_char or has_side) and 'atmosphere' not in desc.lower():
        print(f'⚠️ Shot {n}: no character')
```

**Document the verification in OB** — save the per-shot analysis so you can compare across rounds.

### Phase 6: Verification — Inspect Every Shot, Not Just the Score

**The biggest trap: treating the score as truth.** The scoring system catches different issues each round — same data can score 85 one run and 69 the next because different dimensions are weighted differently. Score alone is misleading.

**Stop after each round.** Never let a test script auto-run multiple rounds. Run one round → present full output → discuss → decide to continue.

**Per-shot 三要素 checklist:**

For every shot in the output, verify:
1. **人 (Character)** — Is there at least one character? If not, is it intentional (empty atmosphere shot)?
   - Check both main characters AND side characters (反派、路人等)
   - An ECU of "her hand" counts — it's implied character presence
   - An empty city night shot may be correct as a transition
2. **景 (Scene/Setting)** — Is the scene clearly anchored?
   - Check for key location keywords in the prompt
   - Scene changes (律师事务所→公寓→车库) should match the narrative
3. **物 (Props)** — Are there at least 2-3 identifiable objects?
   - Count distinct prop keywords per shot
   - Check for consistency across shots (same → same)

**Sample verification script (Python):**
```python
# Check every shot for 人/景/物
for s in shots:
    prompt = s['prompt_en']
    # 人: detect character names
    has_main_char = any(name in prompt for name in ['Qin Lie', 'Han Yu'])
    has_side_char = bool(re.search(r'\b(man|woman|figure|someone|burly)\b', prompt, re.I))
    # 景: detect scene keywords
    scene_kws = ['office', 'bedroom', 'garage', 'apartment', 'street', 'city']
    has_scene = any(kw in prompt.lower() for kw in scene_kws)
    # 物: count props
    prop_kws = ['desk', 'lamp', 'cup', 'pen', 'document', 'car', 'phone']
    prop_count = sum(1 for kw in prop_kws if kw in prompt.lower())
    
    if not (has_main_char or has_side_char) and 'atmosphere' not in desc.lower():
        print(f'⚠️ Shot {n}: no character')
    if not has_scene:
        print(f'⚠️ Shot {n}: no scene anchor')
    if prop_count < 2:
        print(f'⚠️ Shot {n}: only {prop_count} props')
```

**Document the verification in OB** — save the per-shot analysis so you can compare across rounds and detect regression.

## Common Pitfalls

### Pitfall 1: Reverse Mapping Direction
If the source mapping is `{English: Chinese}`, creating a reverse `{Chinese: English}` needs priority handling for case variants (e.g., "Qin Lie" and "Qin lie" both map to "秦烈"). Always prefer the uppercase version.

### Pitfall 2: Duplicate Entries in Map
A map may have `"Qin Lie": "秦烈"` and `"Qin lie": "秦烈"` — dict iteration processes both, and the second overwrites the first. Use a priority condition: prefer uppercase first-letter.

### Pitfall 3: Distribution for Multiple Changes
Changes spread across 4+ files. Use `delegate_task` with parallel workers for independent files, but do sequential patches within the same function to avoid drift.

## Files Modified in Reference Implementation

- `app/routes/l05_storyboard.py` — generation prompt + post-generation guard
- `app/routes/l05_deepen.py` — 6 deepen step prompts + merge prompt + post-merge guards  
- `app/routes/l05_fix.py` — auto-fix prompt + post-fix guard
- `app/_core.py` — consistency helper function + scoring prompt unification
