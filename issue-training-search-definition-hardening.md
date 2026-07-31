## Problem
Current AIPREF category definitions for `train-ai` and `search` need tighter boundary behavior in edge cases, especially where workflows involve retrieval, summarization, and mixed pipelines. This creates ambiguity for implementers and reviewers when categories overlap.

Related discussions already exist, but they do not provide a shared test matrix artifact for classification consistency:
- https://github.com/ietf-wg-aipref/drafts/issues/196
- https://github.com/ietf-wg-aipref/drafts/issues/172
- https://github.com/ietf-wg-aipref/drafts/issues/150

## Proposal
Create a dedicated use-case matrix artifact (appendix or companion markdown) that tests expected behavior per category and overlap precedence.

## Scope
- Define edge-case scenarios where `train-ai` and `search` might both appear applicable.
- Document expected evaluation result for each scenario:
  - Allowed
  - Disallowed
  - Unknown
- Show how overlap is resolved using current combination and specificity rules.
- Identify gaps where current text cannot yield a stable interpretation.

## Suggested Matrix Columns
1. Scenario ID
2. Input source (crawler discovered / user provided / hybrid)
3. Processing stage (indexing / ranking / retrieval / grounding / generation / model update)
4. Output behavior (link-only / snippet / summary / generated answer)
5. Categories triggered (`search`, `train-ai`, other)
6. Declared preferences
7. Expected effective result
8. Rationale (with spec section reference)
9. Ambiguity note (if any)

## Initial Edge-Case Scenarios
- Search retrieval with snippets only (no model update)
- Search retrieval feeding grounding for generated answer
- Search result ranking model continuously updated from click logs
- User-supplied content in prompt plus crawler-sourced context
- Crawler-sourced corpus used for test-time adaptation
- RAG pipeline with caching of embeddings derived from content
- Search display where AI transforms excerpts for accessibility

## Deliverables
- [x] Add matrix to draft appendix or editor-supporting markdown artifact
- [x] Add 6-10 representative edge cases with expected outcomes
- [x] Mark scenarios that are under-specified by current text
- [x] Propose minimal wording changes to reduce ambiguity

## Acceptance Criteria
- Independent reviewers can classify each scenario consistently.
- At least one overlap-rule example is worked end-to-end.
- Any unresolved ambiguity is explicitly tied to draft section and open issue.

## Status
Incorporated into `draft-altanai-aipref-realtime-protocol-bindings` for `-01`:
- Narrow, behavior-based category-trigger wording under privacy considerations
- Non-normative Appendix A with decision tree, worked examples, and edge-case matrix
- Cross-reference to Conflict Resolution for overlap precedence

## Why Now
This makes WG discussion more concrete, reduces re-litigation of abstract terms, and supports cleaner convergence on category boundaries before the next interim.

## Category Trigger Decision Tree (Non-Normative)
Goal: determine which categories are triggered by observed behavior, then evaluate preferences for all triggered categories.

1. Identify the operation performed on content
- Acquisition only (crawl/fetch/store without use decision)
- Retrieval/indexing/ranking for search presentation
- Grounding/context injection for generated output
- Model update or parameter adaptation

2. Check whether behavior is search presentation behavior
- If the operation directs users to source locations via links and associated search content, trigger `search`.

3. Check whether behavior uses content as model-input for generated output
- If content is used as grounding/context to generate answers, trigger the applicable inference/use category.

4. Check whether behavior updates model weights or long-term model state
- If yes, trigger `train-ai` (or a more specific training category if present).

5. Apply preferences to all triggered categories
- Evaluate declared values (`allowed`, `disallowed`, `unknown`) per triggered category.

6. Resolve overlaps using draft combination/specificity rules
- If outcomes differ across triggered categories, apply precedence/composition rules to derive effective result.

### Worked Example A: Search Snippet Only
- Behavior: index + retrieve + show links/snippets; no grounding; no model update
- Triggered categories: `search`
- Notes: if snippets are treated as within search behavior, no `train-ai` trigger occurs

### Worked Example B: Search + Grounded Answer
- Behavior: retrieve sources, then generate answer with grounded context and citations
- Triggered categories: `search` + inference/use category
- Notes: this is the primary overlap case needing consistent interpretation

### Worked Example C: Search + Continuous Learning
- Behavior: search ranking model updated from interaction logs derived from content usage
- Triggered categories: `search` + `train-ai`
- Notes: useful stress case for precedence and under-specified boundaries

### Implementation Note
Category trigger is behavior-based, not keyword-based. Labels in signaling artifacts are inputs to evaluation, but triggering should reflect what the system actually does with content.
