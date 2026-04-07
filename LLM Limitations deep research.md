# LLM limitations and workarounds for agentic CLI workflows

**Every frontier LLM degrades as context fills — and the practical ceiling is far lower than advertised.** Chroma's 2025 research tested 18 frontier models and found universal performance decay beginning well before context limits. Gemini CLI users report quality collapse at just 15–20% of the 1M-token window. This report compiles 80+ production-tested findings on what breaks, why, and exactly how practitioners work around it — with direct implications for building reliable Gemini CLI skill files for Tableau optimization and BigQuery workflows.

---

## TOPIC 1: Core LLM limitations in agentic coding workflows

### Context fills up and everything gets worse

**Finding**: All 18 frontier models tested (GPT-4.1, Claude Opus 4, Gemini 2.5 Pro, Qwen3-235B) degrade continuously as input length increases, even on simple tasks. A model with a 200K token window shows significant degradation at 50K tokens. The decline is continuous, not a cliff — capacity is the wrong metric; **signal-to-noise ratio** determines output quality.
**Example**: A typical coding agent session — reading an issue (500 tokens), grepping files (8,000 tokens), following a wrong lead (6,000 tokens), backtracking (5,000 tokens) — accumulates 20,000 total tokens of which only ~500 are relevant. Signal-to-noise ratio: **2.5%**. The model attends to the relevant code with 1/40th the weight it would have in clean context.
**Why It Matters**: A Tableau optimizer skill that reads workbook XML, analyzes data sources, reviews calculated fields, and generates recommendations will rapidly accumulate tokens. Each step adds context noise. Without aggressive context management, recommendation quality degrades by the third or fourth analysis step.
**Reference**: https://research.trychroma.com/context-rot

**Finding**: The "lost in the middle" problem causes a **30%+ accuracy drop** for information placed in middle positions of the context. LLM performance follows a U-shaped curve: ~75% accuracy for position 1 (beginning), 45–55% for middle positions, ~72% for the final position. This holds even for models designed for long contexts.
**Example**: In a GEMINI.md file with 200 lines of instructions, rules in lines 80–150 will be followed significantly less reliably than rules in the first 20 or last 20 lines.
**Why It Matters**: Critical Tableau optimization rules (e.g., "always check for extract vs. live connection before recommending performance changes") must go at the top or bottom of the skill file, never buried in the middle.
**Reference**: https://aclanthology.org/2024.tacl-1.9/

**Finding**: Every AI coding agent's success rate decreases after **35 minutes** of human-equivalent task time. Doubling task duration quadruples the failure rate. By 35 minutes, agents have typically accumulated 80K–150K tokens across 15–30 files read.
**Example**: A Tableau workbook audit that involves reading the TWB XML, analyzing 5 data sources, reviewing 50 calculated fields, and generating optimization recommendations will likely exceed this threshold if done in a single session.
**Why It Matters**: Break Tableau optimization workflows into discrete phases (extract analysis → data source review → calculated field audit → recommendations) with explicit context resets between phases.
**Reference**: https://www.morphllm.com/context-rot

**Finding**: Well-organized codebases with consistent naming conventions actually **increase distractor density** for the model. Models performed better on randomly shuffled haystacks than logically structured documents across all 18 models tested. Semantically similar content is harder for LLMs to distinguish than random noise.
**Example**: Tableau workbooks with many similarly-named calculated fields (e.g., "Sales YTD," "Sales MTD," "Sales QTD," "Sales LY") create high distractor density. The model may confuse field definitions when multiple similar formulas are in context.
**Why It Matters**: When feeding Tableau calculated field definitions to the LLM, group them by function (time intelligence, LOD expressions, basic calculations) with clear headers, rather than alphabetically, to reduce semantic confusion.
**Reference**: https://research.trychroma.com/context-rot

**Finding**: Context length hurts reasoning even with perfect retrieval. Llama-3.1-8B showed significant performance drops when 25,000 white spaces were inserted — the model could still extract all conditions correctly but nevertheless reached wrong answers. Retrieval and reasoning are independent capabilities, both vulnerable to context length.
**Example**: An LLM might correctly identify that a Tableau data source uses a CROSS JOIN (retrieval) but still recommend the wrong fix (reasoning failure) because the accumulated context from earlier analysis steps has degraded its reasoning capacity.
**Why It Matters**: Don't assume that because the model can "see" the relevant information, it will reason about it correctly. Verify reasoning outputs independently of retrieval accuracy.
**Reference**: https://arxiv.org/html/2510.05381v1

### Instructions get ignored as prompts grow longer

**Finding**: Despite architectural window claims of 128K–1M tokens, **effective context** (where models reliably follow instructions) is often less than 1% of the nominal window on multi-step reasoning tasks. LoCoBench shows that as codebase context scales from 10K to 1M tokens, success rates halve and cross-file reasoning degrades most severely.
**Example**: Gemini CLI's 1M-token window has an effective reliable window closer to 100K–150K tokens for complex coding tasks. Users report: "Performance significantly degrades after I've used around 15–20% of the context."
**Why It Matters**: A Tableau optimizer skill should explicitly instruct the model to use `/compact` or start fresh sessions well before hitting even 20% context usage.
**Reference**: https://github.com/google-gemini/gemini-cli/discussions/5269

**Finding**: Complex instruction sets degrade faster than simple ones. LIFBench (2024) evaluated 20 LLMs and found that single-step instructions are less degraded than range/periodic constraints. The more complex the instruction set, the faster adherence degrades with context length.
**Example**: "Generate BigQuery SQL" (simple) holds up better at high context than "Generate BigQuery SQL using EXTRACT for dates, backtick notation for table references, partition filters for cost optimization, and ARRAY_AGG for repeated fields" (complex compound instruction).
**Why It Matters**: Break complex Tableau optimization rules into separate, simple instructions rather than compound paragraphs. Each rule should express one constraint.
**Reference**: https://arxiv.org/abs/2411.07037

**Finding**: Adding just a 10-word instruction to the end of a prompt reduced total misses from 165 to 74 in needle-in-a-haystack testing of Claude 2.1. This highlights extreme sensitivity of instruction following to prompt phrasing and position — **recency bias** means the last instruction has outsized influence.
**Example**: Append a short reminder at the end of complex prompts: "Remember: always verify SQL syntax is BigQuery-compatible before outputting."
**Why It Matters**: In the Tableau optimizer skill, place a brief "critical reminders" section at the very end of the SKILL.md to exploit recency bias.
**Reference**: https://arize.com/blog-course/the-needle-in-a-haystack-test-evaluating-the-performance-of-llm-rag-systems/

### Hallucination patterns in coding and SQL workflows

**Finding**: API hallucinations constitute the most common factual hallucination type at **20.41%** of all code hallucinations. These create a "snowball effect" — a hallucinated API call leads to hallucinated handling of its response, compounding downstream. Low-frequency APIs (rarely seen in training) are particularly vulnerable.
**Example**: An LLM generating Python code for the Tableau Server REST API might hallucinate endpoint paths or parameter names for less common operations like workbook permissions or extract refresh scheduling.
**Why It Matters**: Include the exact Tableau REST API endpoints and Hyper API function signatures in the skill's reference files. Don't rely on the model's training data for API details — provide authoritative documentation.
**Reference**: https://arxiv.org/html/2407.09726v1

**Finding**: Text-to-SQL hallucinations occur in two distinct phases: **schema-linking hallucinations** (mapping natural language to wrong table/column pairs) and **logical-synthesis hallucinations** (producing syntactically valid but logically incorrect SQL). Many SQL hallucinations remain syntactically valid and evade simple syntax checks.
**Example**: The LLM generates `SELECT SUM(revenue) FROM orders` when the actual column is `order_total` in table `transactions`. The SQL parses fine but returns an error only at execution time — or worse, if a `revenue` column exists in another table, it returns wrong results silently.
**Why It Matters**: For Tableau Custom SQL optimization, syntax validation alone is insufficient. The skill should instruct the model to verify column names against the provided schema and explain its join logic step by step.
**Reference**: https://arxiv.org/html/2512.22250v1

**Finding**: The four most common SQL generation failures are: (1) **faulty joins** — missing intermediate tables or using subqueries instead of proper JOINs, (2) **aggregation mistakes** — wrong GROUP BY levels, (3) **missing filters** — omitting business-logic WHERE clauses, (4) **schema hallucination** — inventing column/table names. These patterns dominate real production failures across 50,000+ evaluated queries.
**Example**: LLM joins `customers` directly to `products`, missing the intermediate `orders` and `order_items` tables. Or omits `WHERE is_active = TRUE` that is an implicit business rule.
**Why It Matters**: The Tableau optimizer skill should include explicit join-path verification steps and business rule checklists extracted from the workbook's existing calculated fields and filters.
**Reference**: https://www.usedatabrain.com/blog/llm-sql-evaluation

**Finding**: Code inconsistency hallucinations include undefined variables (16.91%), useless/inert statements (6.60%), and fragmented logic (1.73%). Reasoning-enhanced models like DeepSeek-R1 show lower rates, suggesting enhanced reasoning mitigates some hallucinations.
**Example**: Generated Python code references a variable `tableau_connection` that was never defined in the current scope, or includes a `try/except` block that catches an exception but does nothing with it.
**Why It Matters**: When the skill generates multi-step optimization scripts, instruct it to verify all variables are defined and all code paths are reachable before presenting output.
**Reference**: https://arxiv.org/html/2404.00971v3

### Tool calling breaks in predictable ways

**Finding**: Common tool-calling failure modes include: **eager invocation** (tools called unnecessarily), **wrong tool selection** (searching when should be reading), and **invalid arguments** (missing or malformed parameters). The NESTful benchmark found existing LLMs score **below 10%** on nested/sequential API calls — a critical real-world requirement.
**Example**: When asked to optimize a Tableau workbook, the model might immediately try to write modified XML before reading the existing file, or call `read_file` on a path that doesn't exist because it hallucinated the filename.
**Why It Matters**: The skill file should specify an explicit tool-use sequence: "ALWAYS read the TWB file first. THEN parse the XML structure. THEN analyze specific sections. NEVER write changes without reading the current state."
**Reference**: https://www.docker.com/blog/local-llm-tool-calling-a-practical-evaluation/

**Finding**: Models tuned for helpfulness substitute missing data rather than reporting absence — the "helpfulness bias." When asked about data for a company not in the database, models substitute a similar company's data or interpret "missing company" as "all companies," producing computationally valid but factually wrong results.
**Example**: If the Tableau workbook references a data source that the model can't read, it might fabricate plausible-sounding column names and schema details rather than reporting "I cannot access this data source."
**Why It Matters**: Include explicit instructions: "If you cannot read or access a file, report this clearly. NEVER guess or fabricate file contents, schema details, or data values. Say 'I could not access [file] — please provide it' instead."
**Reference**: https://arxiv.org/pdf/2512.07497

### Planning failures compound in multi-step tasks

**Finding**: GPT-4 achieved only **0.6% success rate** on complex travel planning benchmarks when applied end-to-end. Kambhampati et al. argued in a widely cited position paper that "LLMs Can't Plan, But Can Help Planning in LLM-Modulo Frameworks" — LLMs should serve as idea generators within verification loops rather than autonomous planners.
**Example**: Asking an LLM to "optimize this entire Tableau workbook end-to-end" will fail. Asking it to "identify the three highest-impact performance issues in this data source configuration" succeeds because the scope is bounded.
**Why It Matters**: The Tableau optimizer skill must decompose optimization into discrete, verifiable steps rather than presenting it as a single planning task. Each step should have clear success criteria.
**Reference**: https://openreview.net/pdf?id=kFrqoVtMIy

**Finding**: Multi-hop reasoning degrades sharply due to three root causes: (1) **missing memory** — failure to represent intermediate facts needed for subsequent steps, (2) **hypothetical inconsistency** — the model's answer isn't invariant when queried about its own reasoning, (3) **compositional inconsistency** — substituting a model's sub-step output into a subsequent prompt changes the final outcome. Performance drops to **chance level (25%)** when essential intermediate steps are missing.
**Example**: The model correctly identifies that a Tableau extract is 2GB (step 1), correctly knows that extracts over 1GB benefit from aggregation (step 2), but fails to connect these facts and recommend aggregate extracts (step 3).
**Why It Matters**: The skill should force explicit intermediate reasoning: "First, list all findings from the data source analysis. Then, for each finding, state the applicable optimization rule. Finally, generate the specific recommendation."
**Reference**: https://www.emergentmind.com/topics/reasoning-failures-in-llms

---

## TOPIC 2: Practical workarounds and prompt engineering strategies

### GEMINI.md and CLAUDE.md as persistent context

**Finding**: GEMINI.md uses a **hierarchical three-tier loading system**: (1) Global `~/.gemini/GEMINI.md` for user-wide defaults, (2) Project root `GEMINI.md` for repo-specific rules, (3) Dynamic subdirectory GEMINI.md files auto-discovered when tools access files. Use `/memory show` to inspect concatenated context and `/memory reload` after edits.
**Example**:
```
~/.gemini/GEMINI.md → "Always use strict typing, prefer explicit over implicit"
./GEMINI.md → "Python 3.11, use uv, BigQuery Standard SQL"
./skills/tableau-optimizer/GEMINI.md → "Tableau-specific analysis rules"
```
**Why It Matters**: The Tableau optimizer can use subdirectory-level GEMINI.md files to scope optimization rules to specific contexts (BigQuery SQL rules vs. Tableau XML analysis rules).
**Reference**: https://geminicli.com/docs/cli/gemini-md/

**Finding**: GEMINI.md supports **modular imports** with `@file.md` syntax and in-session `/memory add` for persistent facts. Use `/compress` when sessions grow long — this compresses chat context but preserves GEMINI.md content, which is always re-injected.
**Example**:
```markdown
# Main GEMINI.md
@./components/bigquery-rules.md
@./components/tableau-patterns.md
@./docs/optimization-checklist.md
```
**Why It Matters**: Break the Tableau optimizer's rules into modular components (BigQuery SQL rules, Tableau XML patterns, LOD expression guidelines) imported via `@` syntax rather than one massive file.
**Reference**: https://geminicli.com/docs/cli/gemini-md/

**Finding**: Claude Code wraps CLAUDE.md with a system reminder: "this context **may or may not be relevant** to your tasks." This means Claude actively ignores CLAUDE.md content it deems irrelevant. The more non-universal content in the file, the more likely ALL instructions get ignored.
**Example**: ❌ "When creating database schemas, use UUIDs for primary keys" (only sometimes relevant — gets ignored). ✅ "Run `npm run typecheck` after code changes" (universally applicable — gets followed).
**Why It Matters**: In both GEMINI.md and skill files, every instruction must be clearly relevant to the active task. Move specialized Tableau optimization rules into a dedicated skill that activates on-demand, not the root GEMINI.md.
**Reference**: https://www.humanlayer.dev/blog/writing-a-good-claude-md

**Finding**: Frontier LLMs can reliably follow approximately **150–200 instructions**. Performance degrades uniformly as instruction count increases. Claude Code's built-in system prompt already contains ~50 instructions, consuming roughly a third of the budget before any user rules are added.
**Example**: HumanLayer's own root CLAUDE.md is under 60 lines. Community consensus is under 200 lines maximum, with shorter being better. For each line, ask: "Would removing this cause the model to make mistakes? If not, cut it."
**Why It Matters**: The Tableau optimizer skill file should be ruthlessly concise. Every instruction must earn its place. Aim for under 150 lines in the main SKILL.md with details in reference files.
**Reference**: https://www.humanlayer.dev/blog/writing-a-good-claude-md

### System prompt engineering that actually works

**Finding**: Prefer **general instructions over prescriptive step-by-step plans** in agentic prompts. Anthropic's guidance: "A prompt like 'think thoroughly' often produces better reasoning than a hand-written step-by-step plan. Claude's reasoning frequently exceeds what a human would prescribe."
**Example**: Instead of a 10-step Tableau analysis plan, use: "This workbook analysis is complex. Plan your approach carefully, then work systematically through each data source and worksheet. Continue until you have completed the full optimization review."
**Why It Matters**: Over-specifying analysis steps constrains the model. Let it determine the optimal investigation order while specifying what outputs you expect.
**Reference**: https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices

**Finding**: XML tags are Anthropic's officially recommended method for structuring complex prompts. They provide clarity, accuracy, flexibility, and parseability. No canonical "best" tag names — use semantic names that describe the content.
**Example**:
```xml
<tableau_context>
  <data_sources>{{extracted_datasource_xml}}</data_sources>
  <calculated_fields>{{extracted_calc_fields}}</calculated_fields>
</tableau_context>
<optimization_rules>
  Check for: extract vs live connection, unused fields, LOD efficiency
</optimization_rules>
<output_format>
  Return findings as JSON with severity, description, and recommendation.
</output_format>
```
**Why It Matters**: Structuring the Tableau optimizer skill with XML tags creates clear boundaries between workbook context, optimization rules, and output format — reducing ambiguity.
**Reference**: https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/use-xml-tags

**Finding**: Combining XML tags with few-shot examples and chain-of-thought creates high-performance prompts. Use `<thinking>` tags inside few-shot examples to show reasoning patterns, and the model will generalize that reasoning style.
**Example**:
```xml
<example>
  <input>Analyze this calculated field: IF [Region] = "West" THEN [Sales] END</input>
  <thinking>This field uses IF/THEN without ELSE, returning NULL for non-West regions. 
  This creates sparse data. Could be replaced with IIF() for better performance.</thinking>
  <recommendation>Replace IF/THEN/END with IIF([Region]="West", [Sales], NULL) 
  for ~15% calculation speed improvement.</recommendation>
</example>
```
**Why It Matters**: Showing the model how to reason about Tableau calculated field optimization through examples is more effective than writing abstract rules about it.
**Reference**: https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/use-xml-tags

### Context management for long sessions

**Finding**: **Observation masking** outperforms LLM summarization for coding agent context management. JetBrains Research found that replacing old tool outputs with placeholders like `[Output Omitted]` while keeping the agent's reasoning/commands intact consistently matched or outperformed LLM summarization across experiments. Both approaches cut costs by >50%. LLM summarization had a hidden cost: summary-generation API calls comprised >7% of total spend.
**Example**: Replace verbose tool output: `[TWB XML content from read_file omitted — 47 data sources extracted, see analysis in previous turn]` while preserving: "I analyzed the data sources and found 3 using live connections to BigQuery without partition filters."
**Why It Matters**: When the Tableau optimizer reads large TWB files, the raw XML should be masked from context after analysis, preserving only the extracted findings.
**Reference**: https://blog.jetbrains.com/research/2025/12/efficient-context-management/

**Finding**: LLM summarization can cause **"trajectory elongation"** — smoothing over failures leads agents to repeat mistakes. When an LLM summarized a failed analysis, it softened the severity. The agent, reading the summary, didn't realize how stuck it was and repeated failed actions.
**Example**: If the model fails to parse a Tableau XML section and the context is summarized as "partial analysis of data sources completed," the model won't recognize it needs to retry with a different approach.
**Why It Matters**: Use observation masking by default. Trigger LLM summarization only as a last resort when context is critically full.
**Reference**: https://blog.jetbrains.com/research/2025/12/efficient-context-management/

**Finding**: Anthropic recommends **hierarchical summarization at multiple time scales** for sessions exceeding 50 messages: detailed summaries every 5 messages, moderate summaries every 25 messages, high-level overviews every 100 messages. Build context as: most recent high-level summary + last 2 moderate summaries + last 10 raw messages.
**Example**: `[High-level: "Analyzing customer_analytics.twb — 5 data sources, 3 dashboards, 47 calc fields identified"] + [Last 2 section summaries] + [Last 10 verbatim messages with current analysis]`
**Why It Matters**: For multi-phase Tableau workbook analysis, maintain a running project summary that gets progressively compressed, with only the current analysis phase kept at full fidelity.
**Reference**: https://medium.com/@najeebkan/context-management-for-ai-agents-d68716a37965

**Finding**: Start new sessions rather than compacting for task switches. Write plans and state to files that persist across sessions. Google's "Conductor" extension formalizes this: `spec.md` and `plan.md` live alongside code, work can be paused/resumed/moved between machines.
**Example**: Write to `tableau-audit-state.md`: "## Phase 1 Complete ✅\n- 5 data sources analyzed\n- 3 live connections identified\n- Issues: [list]\n\n## Phase 2: Calculated Field Review\n- [ ] Review LOD expressions\n- [ ] Check string calculations\n- [ ] Verify date functions"
**Why It Matters**: The Tableau optimizer should write analysis state to a file after each phase, enabling `/clear` and fresh session start for the next phase without losing work.
**Reference**: https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/use-xml-tags

### Output format control

**Finding**: **Structured Outputs** (API-native JSON schema enforcement) achieves 100% schema compliance vs. ~35% with prompting alone. For self-hosted models, constrained decoding offers similar guarantees.
**Example**: Use API-level `response_format: { type: "json_schema", json_schema: {...} }` rather than prompt-level instructions for machine-consumed outputs.
**Why It Matters**: When the Tableau optimizer generates structured findings (JSON with severity, category, recommendation), use structured output mode to guarantee parseable results.
**Reference**: https://agenta.ai/blog/the-guide-to-structured-outputs-and-function-calling-with-llms

**Finding**: Format restrictions **degrade LLM reasoning capabilities**. Stricter constraints cause larger performance drops, especially on mathematical problem-solving. The LLM is "unaware" of the constraints during probability computation.
**Example**: For reasoning-heavy tasks (analyzing whether a Tableau LOD expression is correct), let the model reason in free text first, then extract structured output in a second pass.
**Why It Matters**: Don't force JSON output during the analysis phase. Let the model think freely about Tableau optimization, then format the results.
**Reference**: https://www.dataiku.com/stories/blog/your-guide-to-structured-text-generation

**Finding**: Field naming in structured output schemas dramatically impacts accuracy. Changing a single field name improved accuracy from **4.5% to 95%**. Adding a `reasoning` field as the **FIRST field** in a JSON schema improved accuracy by 60%.
**Example**:
```json
{
  "analysis_reasoning": "The LOD expression uses FIXED which creates a separate query...",
  "finding": "Inefficient LOD nesting",
  "severity": "high",
  "recommendation": "Combine nested FIXED LODs into a single expression"
}
```
Place `analysis_reasoning` BEFORE `finding` in the schema definition.
**Why It Matters**: The Tableau optimizer's output schema must put a reasoning/analysis field first so the model thinks before concluding. This single design choice can transform output quality.
**Reference**: https://python.useinstructor.com/blog/2024/09/26/bad-schemas-could-break-your-llm-structured-outputs/

### Verification loops — what works and what doesn't

**Finding**: LLMs **largely cannot self-correct** reasoning through intrinsic prompting alone. "Check your work" without external feedback is unreliable and sometimes degrades performance. A critical survey (TACL 2024) found no prior work demonstrates successful self-correction with feedback from prompted LLMs alone.
**Example**: Asking "Is this SQL correct?" without running it produces unreliable verification. The model tends to confirm its own output.
**Why It Matters**: The Tableau optimizer must not rely on "verify your recommendations" as a quality gate. Instead, use external validation: execute generated SQL, parse generated XML, run linters.
**Reference**: https://aclanthology.org/2024.tacl-1.78/

**Finding**: Self-correction works when **external tools provide verifiable feedback** — code execution, unit tests, and compilers are ideal verifiers. The CRITIC framework demonstrated LLMs can effectively verify and correct outputs by interacting with external tools.
**Example**: Agentic loop: `Generate BigQuery SQL → Execute in dry-run mode → If error, feed error message back → Regenerate → Repeat up to 2 times`
**Why It Matters**: The Tableau optimizer should validate generated SQL by executing BigQuery dry runs and validate generated XML by parsing it, feeding errors back for self-correction.
**Reference**: https://openreview.net/forum?id=Sx038qxjek

**Finding**: The "verify-then-correct" approach (ProCo framework) achieves **+6.8 to +14.1 accuracy improvement** by masking key conditions and constructing verification questions, rather than simply asking "is this correct?"
**Example**: Instead of: "Is this LOD correct?" → Verification: "Given this result, what dimension should the FIXED LOD be computing over?" If the model's answer doesn't match the original LOD's dimension, the expression is likely wrong.
**Why It Matters**: Build targeted verification questions into the skill: "After generating a recommendation, verify: What specific Tableau feature does this optimize? What is the expected performance improvement? If you can't answer both specifically, revise."
**Reference**: https://aclanthology.org/2024.emnlp-main.714/

### Few-shot examples — how many and where

**Finding**: Diminishing returns after **2–3 few-shot examples**. Testing across 12 LLMs showed 9/12 improved with few-shot, but some models degraded. Benefits were more pronounced in generative tasks than classification.
**Example**: Include 2–3 diverse Tableau optimization examples in the skill (one data source issue, one calculated field issue, one dashboard layout issue) rather than 10+ similar examples.
**Why It Matters**: The Tableau optimizer skill should include exactly 2–3 high-quality examples covering the most common optimization categories.
**Reference**: https://www.prompthub.us/blog/the-few-shot-prompting-guide

**Finding**: Input context placement matters. Most models perform best with **long data at the beginning** and instructions/queries at the end. Claude-3.5-Sonnet achieved highest accuracy with beginning placement. Anthropic recommends: longform data at the top, queries at the bottom, improving accuracy by up to 30%.
**Example**: Structure as: `[TWB XML / schema data] → [Optimization rules and instructions] → [Few-shot examples] → [Specific query: "Analyze this workbook"]`
**Why It Matters**: When feeding Tableau workbook data to the model, place the XML/schema data first, then rules, then the analysis request.
**Reference**: https://medium.com/tr-labs-ml-engineering-blog/optimizing-prompts-across-llms-a-comprehensive-overview-part-1-3ae4b0a2ff51

---

## TOPIC 3: Gemini CLI specific patterns and limitations

### GEMINI.md — what works and what doesn't

**Finding**: Monolithic GEMINI.md files cause "context bloat" and degrade performance. A Google Cloud community author found that a comprehensive, all-in-one GEMINI.md with detailed instructions for every scenario made the assistant "confused, slow, and often incorrect."
**Example**: The author's initial approach (a full PRAR workflow + tech-stack decision guide in a single file) performed worse than breaking it into gated operational modes with distinct `<PROTOCOL>` blocks.
**Why It Matters**: The Tableau optimizer's instructions should NOT all live in GEMINI.md. Use GEMINI.md for global defaults only; move optimization logic into a dedicated Agent Skill.
**Reference**: https://medium.com/google-cloud/practical-gemini-cli-structured-approach-to-bloated-gemini-md-360d8a5c7487

**Finding**: **"Gated execution through delayed instructions"** is the recommended GEMINI.md pattern. Define distinct operational modes with gated transitions — Default State (listen), Explain Mode (read-only investigation), Plan Mode (strategy, no implementation), Implement Mode (only after explicit plan approval). Each mode has its own tightly-scoped `<PROTOCOL>` block.
**Example**:
```markdown
## Modes
### EXPLAIN MODE
<PROTOCOL>Read files and explain findings. Do NOT modify any files.</PROTOCOL>

### PLAN MODE  
<PROTOCOL>Create optimization plan in plan.md. Do NOT implement changes.</PROTOCOL>

### IMPLEMENT MODE
<PROTOCOL>Only enter after user approves plan. Apply changes from plan.md.</PROTOCOL>
```
**Why It Matters**: The Tableau optimizer should use gated modes: Analyze (read TWB, identify issues), Plan (generate optimization recommendations), Implement (apply changes only after user review).
**Reference**: https://medium.com/google-cloud/practical-gemini-cli-structured-approach-to-bloated-gemini-md-360d8a5c7487

### Agent Skills solve context bloat

**Finding**: Agent Skills are the newer, more powerful alternative to bloated GEMINI.md files. Skills use **progressive disclosure**: only name + description are loaded at startup. Detailed instructions load on-demand when activated, saving context tokens. Skills follow the open Agent Skills spec, making them portable across Claude Code, GitHub Copilot, and Cursor.
**Example**:
```yaml
---
name: tableau-optimizer
description: Analyze and optimize Tableau workbooks (.twb/.twbx). Use when the user 
  asks to "optimize", "audit", "review", or "improve" a Tableau workbook, dashboard, 
  or data source.
---
# Tableau Optimizer Instructions
When this skill is active, you MUST:
1. Read the TWB file and extract data source configurations
2. Analyze calculated fields for performance issues
3. Generate prioritized optimization recommendations
```
**Why It Matters**: This is the exact mechanism for building the Tableau optimizer. The skill's metadata loads cheaply at startup; full instructions load only when the user triggers an optimization task.
**Reference**: https://geminicli.com/docs/cli/skills/

**Finding**: Custom slash commands are defined in `.toml` files in `~/.gemini/commands/` (global) or `.gemini/commands/` (project-scoped). They support `{{args}}` for argument injection, `!{...}` for shell command execution, and `@{...}` for file content injection.
**Example**:
```toml
# .gemini/commands/tableau/audit.toml
description = "Audit a Tableau workbook for performance issues"
prompt = """
Analyze the following Tableau workbook file for performance optimization opportunities.
Focus on: data source connections, calculated field efficiency, extract vs live connections,
LOD expression complexity, and unused fields.

File contents:
@{{{args}}}
"""
```
Triggered with: `/tableau:audit path/to/workbook.twb`
**Why It Matters**: Create a `/tableau:audit` slash command as a quick-trigger entry point for the optimization skill.
**Reference**: https://geminicli.com/docs/cli/custom-commands/

### Context window degradation is a critical issue

**Finding**: Gemini CLI users report significant degradation after using just **15–20% of the 1M-token context window**. Multiple GitHub issues document the model confusing past information with current state, stopping to honor conditions in context files, entering loops, and providing worse solutions.
**Example**: Users report: "It doesn't do any of that in a fresh chat. It's quite good when I start a fresh chat and up to the point I've used 10% of the context." The issue was flagged as **P0 (critical)** in August 2025.
**Why It Matters**: The Tableau optimizer MUST include explicit context management instructions: compact after each analysis phase, start fresh sessions for new workbooks, never let context exceed ~15% before compacting.
**Reference**: https://github.com/google-gemini/gemini-cli/issues/5160

**Finding**: Essential context management commands for long sessions: `/clear` (hard reset), `/compact` (replace chat with summary, preserves GEMINI.md), `/chat save <tag>` and `/chat resume <tag>` (branch conversation state), `/restore` (undo file changes via checkpointing).
**Example**: Workflow: Analyze data sources → `/compact` → Analyze calculated fields → `/compact` → Generate recommendations → `/chat save pre-implement` → Apply changes → if wrong, `/restore` and `/chat resume pre-implement`
**Why It Matters**: Build these commands into the skill's workflow instructions so the model manages its own context proactively.
**Reference**: https://medium.com/google-cloud/gemini-cli-tutorial-series-part-9-understanding-context-memory-and-conversational-branching-095feb3e5a43

### Gemini CLI vs. Claude Code

**Finding**: Claude Code consistently produces cleaner, more maintainable code; Gemini CLI is faster and cheaper for prototyping. In a Composio benchmark, Claude finished a full CLI tool build in 1h17m vs Gemini's 2h2m. Claude was **~80% autonomous**; Gemini needed manual nudging and retries. Claude used 260K input tokens vs. Gemini's 432K for the same task.
**Example**: A creative HackerNews strategy: "Claude makes the plan, and let Gemini implement." This leverages Claude's superior planning with Gemini's free tier for execution.
**Why It Matters**: For the Tableau optimizer, consider using Claude Code for complex analysis planning and Gemini CLI for execution of well-defined optimization steps.
**Reference**: https://composio.dev/content/gemini-cli-vs-claude-code-the-better-coding-agent

**Finding**: Gemini CLI integrates tightly with **Google Cloud services**: BigQuery, Cloud Functions, Vertex AI, Cloud Run, Firebase. Google provides first-party BigQuery Data Analytics extensions with natural-language-to-SQL, forecasting, and contribution analysis capabilities. Install via `gemini extensions install bigquery-data-analytics`.
**Example**: Use the BigQuery extension to: find required tables via natural language, ask analytical questions and generate SQL automatically, generate forecasts — all from the terminal without switching to GCP console.
**Why It Matters**: The Tableau optimizer can leverage the BigQuery extension to validate generated SQL, inspect actual table schemas, and test query performance directly from Gemini CLI.
**Reference**: https://docs.cloud.google.com/bigquery/docs/develop-with-gemini-cli

### Tool use patterns and safety

**Finding**: Gemini CLI has 11+ built-in tools: `read_file`, `read_many_files`, `write_file`, `replace`, `list_directory`, `glob`, `search_file_content`, `run_shell_command`, `web_fetch`, `google_web_search`, `save_memory`, `write_todos`, and `codebase_investigator`. The system prompt defines a strategic sequence: Understand (grep, glob, read) → Plan & Implement (edit/replace) → Verify (shell for tests).
**Example**: Direct tool triggers: `@filepath` triggers `read_many_files`, `!command` triggers `run_shell_command`. Restrict tools via settings.json: `"tools": {"core": ["read_file", "search_file_content", "run_shell_command(python3)"]}` to limit what the optimizer can do.
**Why It Matters**: For the Tableau optimizer, restrict write tools initially — allow read/search/shell but require explicit confirmation before file modifications. Use `--sandbox` for safe execution.
**Reference**: https://google-gemini.github.io/gemini-cli/docs/tools/

---

## TOPIC 4: Skill file and instruction file design patterns

### Structure that LLMs actually follow

**Finding**: Structure instruction files around **WHAT / WHY / HOW**: WHAT (tech stack, project structure, codebase map), WHY (project purpose, what each component does), HOW (commands, verification steps — how the model should actually work).
**Example**:
```markdown
# Tableau Optimizer Skill (WHAT)
Analyzes .twb/.twbx workbook XML for performance optimization.

# Purpose (WHY)
Identifies data source, calculated field, and dashboard inefficiencies 
that cause slow load times and excessive BigQuery costs.

# Workflow (HOW)
1. Extract and parse TWB XML
2. Analyze each data source connection
3. Review calculated fields for anti-patterns
4. Generate prioritized recommendations with severity ratings
```
**Why It Matters**: This three-pillar structure gives the model clear context on what it's doing, why it matters, and exactly how to execute — reducing ambiguity.
**Reference**: https://www.humanlayer.dev/blog/writing-a-good-claude-md

**Finding**: Use **progressive disclosure** as the core design principle. Keep the main SKILL.md concise with pointers to detailed docs. Three loading stages: metadata (~100 tokens, always in context), SKILL.md body (<5k tokens, loaded when triggered), bundled resources (loaded on demand).
**Example**:
```
tableau-optimizer/
├── SKILL.md              # Main instructions (<150 lines)
├── references/
│   ├── bigquery-sql-rules.md    # BigQuery dialect patterns
│   ├── tableau-calc-patterns.md # Calculated field anti-patterns
│   ├── lod-optimization.md      # LOD expression best practices
│   └── twb-xml-structure.md     # TWB XML parsing guide
├── scripts/
│   ├── extract_calcs.py         # Extract calc fields from TWB
│   └── validate_sql.py          # BigQuery SQL dry-run validator
└── assets/
    └── example-audit.json       # Example output format
```
**Why It Matters**: This structure keeps the initial context cost minimal while providing deep reference material the model can access when needed for specific optimization tasks.
**Reference**: https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills

**Finding**: Keep SKILL.md **under 500 lines**, GEMINI.md/CLAUDE.md **under 200 lines**. Don't auto-generate these files — they're the highest-leverage point of the entire harness. A bad line affects every phase of every workflow.
**Example**: For each line, ask: "Would removing this cause the model to make mistakes on Tableau optimization tasks? If not, cut it."
**Why It Matters**: A 500-line skill file that's never read is worse than a 100-line file that's always followed. Ruthless concision is essential.
**Reference**: https://code.claude.com/docs/en/best-practices

### Making triggers reliable

**Finding**: Skill description is the primary triggering mechanism. The description should include both what the skill does AND specific trigger phrases. Skills tend to **undertrigger** rather than overtrigger, so descriptions should be slightly "pushy." Simple, one-step queries may not trigger a skill even if the description matches.
**Example**:
```yaml
# Too vague (undertriggers):
description: Tableau workbook analysis

# Better (specific triggers):
description: Analyze and optimize Tableau workbooks (.twb/.twbx files). 
  Use this skill whenever the user mentions Tableau optimization, workbook 
  performance, dashboard speed, calculated field review, data source audit, 
  extract optimization, LOD analysis, or BigQuery query optimization for 
  Tableau, even if they don't explicitly say "optimize."
```
**Why It Matters**: The Tableau optimizer skill needs an aggressive description that catches all the ways a user might phrase an optimization request.
**Reference**: https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md

**Finding**: Skills trigger via natural language — no special commands needed. The description field tells the model when to activate. However, for maximum reliability, pair the skill with a **custom slash command** as a guaranteed trigger path.
**Example**: Natural trigger: "Can you audit this Tableau workbook for performance?" → Skill activates. Guaranteed trigger: `/tableau:audit workbook.twb` → Always activates.
**Why It Matters**: Provide both paths: natural language triggers for organic use and a slash command for when the user wants guaranteed activation.
**Reference**: https://medium.com/google-cloud/your-gemini-cli-extensions-just-got-smarter-introducing-agent-skills-a8fbfa077e7f

### Negative instructions — handle with care

**Finding**: Negative instructions ("NEVER do X") are unreliable — the **"Pink Elephant Problem."** Research shows InstructGPT models actually perform worse with negative prompts as they scale. One user reported Claude Code creating `file-fixed.py` and `file-correct.py` despite "NEVER create duplicate files" in CLAUDE.md.
**Example**:
| Negative (Less Effective) | Positive (More Effective) |
|---|---|
| "Don't modify the original TWB" | "Create all changes in a new file: workbook-optimized.twb" |
| "Never use RAWSQL functions" | "Use native Tableau calculations exclusively" |
| "Don't generate SQL without schema" | "Always read the schema before generating SQL" |
**Why It Matters**: Frame all Tableau optimization constraints as positive directives. Instead of listing what not to do, specify exactly what to do.
**Reference**: https://eval.16x.engineer/blog/the-pink-elephant-negative-instructions-llms-effectiveness-analysis

**Finding**: Use emphasis markers ("IMPORTANT", "YOU MUST") to improve adherence on critical rules. Anthropic's own docs confirm this works. Reserve negative instructions for absolute hard boundaries only, and always pair them with positive alternatives.
**Example**: "IMPORTANT: YOU MUST read the complete TWB XML before generating any recommendations. Never guess at workbook structure — always verify by reading the file."
**Why It Matters**: The most critical Tableau optimizer rules (read before writing, verify before recommending) should use emphasis markers.
**Reference**: https://code.claude.com/docs/en/best-practices

### Persona and role definitions

**Finding**: Persona prompting has **mixed results** — it improves alignment-dependent tasks (architecture preferences, style) but hurts accuracy-dependent tasks (math, coding correctness). Telling a model it's an "expert programmer" does not improve code quality. However, detailed, domain-specific persona definitions outperform vague ones when alignment matters.
**Example**:
```markdown
# Bad
You are an expert Tableau developer.

# Better
You are a Tableau performance optimization specialist. You analyze TWB XML 
to identify data source inefficiencies, calculated field anti-patterns, 
and dashboard design issues. You prioritize recommendations by performance 
impact. You know BigQuery Standard SQL and Tableau calculation syntax deeply.
```
**Why It Matters**: Use a specific persona that defines expertise scope and working style, but don't expect it to improve factual accuracy — that comes from providing correct reference material.
**Reference**: https://www.prompthub.us/blog/role-prompting-does-adding-personas-to-your-prompts-really-make-a-difference

**Finding**: Anthropic uses **descriptive third-person framing** over imperative roles in their own system prompts. "Claude does X" rather than "You must do X." This approach may perform better for some models.
**Example**: "The Tableau optimizer always reads the complete TWB file before making recommendations. It verifies all column references against the data source schema. It generates BigQuery Standard SQL exclusively."
**Why It Matters**: Test both framing styles (imperative "You must" vs. descriptive "The optimizer always") and use whichever produces more consistent behavior.
**Reference**: https://eval.16x.engineer/blog/the-pink-elephant-negative-instructions-llms-effectiveness-analysis

**Finding**: Rules require **periodic reinforcement** in long conversations. After several messages, rules get ignored due to recency bias. A useful trick: require the agent to mention which rules it's applying in its response.
**Example**: Add to skill: "At the start of each analysis section, state which optimization category you are evaluating (data source, calculated fields, LOD expressions, dashboard layout)."
**Why It Matters**: For multi-phase Tableau analysis, having the model explicitly state its current focus area acts as a self-reinforcement mechanism for following the skill's rules.
**Reference**: https://medium.com/elementor-engineers/cursor-rules-best-practices-for-developers-16a438a4935c

### Document mistakes, not everything

**Finding**: The most effective instruction files are **reactive** — they encode corrections for observed mistakes, not comprehensive manuals. When the model makes a mistake, add the correction to the instruction file. The file becomes a living feedback loop.
**Example**: After observing that the model repeatedly recommends `RAWSQL` functions (which bypass Tableau's query optimizer), add: "IMPORTANT: Never recommend RAWSQL functions. Use native Tableau calculations. RAWSQL bypasses query optimization and creates vendor lock-in."
**Why It Matters**: Start the Tableau optimizer skill lean, then add rules iteratively as you observe failure patterns. This produces a more effective file than trying to anticipate every scenario upfront.
**Reference**: https://www.turbodocx.com/blog/how-to-write-claude-md-best-practices

---

## TOPIC 5: Data engineering and analytics LLM workflow patterns

### SQL generation — what makes it reliable

**Finding**: On Spider benchmark, top systems achieve 85–86% execution accuracy. But **Spider 2.0** (testing real enterprise workflows) sees accuracy plummet to **6–16%**. BIRD benchmark with enterprise databases shows GPT-4 at only 52% vs. 93% for human experts. The gap is driven by complex schemas, multi-step reasoning, ambiguous questions, and dirty data.
**Example**: A query that works on a clean academic benchmark fails in production because the enterprise schema has 200+ tables, inconsistent naming, and implicit business rules.
**Why It Matters**: Assume ~50% accuracy for unassisted SQL generation against real Tableau/BigQuery schemas. Every generated query needs human review or automated validation.
**Reference**: https://promethium.ai/guides/text-to-sql-evaluation-benchmarks-metrics/

**Finding**: **DDL (CREATE TABLE) format** is the most effective format for schema prompts. Adding 3–5 sample rows per table boosts accuracy by **4–10 percentage points**. The combination of DDL + sample rows + foreign key declarations is the gold standard. One practitioner went from 60.9% to 70.8% accuracy by adding sample rows and few-shot examples.
**Example**:
```sql
CREATE TABLE orders (
  order_id INT64 NOT NULL,
  customer_id INT64 REFERENCES customers(customer_id),
  order_date DATE,
  total DECIMAL(10,2),
  region STRING
) PARTITION BY order_date;
-- Sample: | 1 | 42 | 2024-01-15 | 299.99 | "West" |
-- Sample: | 2 | 17 | 2024-01-16 | 149.50 | "East" |
```
**Why It Matters**: When the Tableau optimizer needs to generate or review BigQuery SQL, always extract and include the DDL with sample data. This should be a standard step in the skill workflow.
**Reference**: https://www.databricks.com/blog/improving-text2sql-performance-ease-databricks

**Finding**: Filter schema to **relevant tables only** — don't dump the whole catalog. For databases with 100+ tables, use domain-based filtering or vector search over table/column descriptions to retrieve the ~5–10 most relevant tables.
**Example**: User asks about revenue → Schema router selects `orders`, `order_items`, `products`, `customers` only, filtering out 195 other tables.
**Why It Matters**: Tableau workbooks already define which tables they use. Extract the table list from the TWB XML data source definitions and provide only those schemas, not the entire BigQuery dataset.
**Reference**: https://dev.to/rakesh_tanwar_8a7d83bc8f0/best-practices-for-connecting-llms-to-sql-databases-47pn

**Finding**: **Semantic layers and business context** are critical. Without business context, LLMs miss implicit rules. A semantic layer that defines metrics boosts accuracy from ~55% to **90%+**. Pre-defined metrics eliminate most aggregation and filter errors.
**Example**: Include in prompt: `"active_customers" = SELECT COUNT(*) FROM customers WHERE status = 'active' AND account_type != 'trial'`. When the user asks about "customers," the semantic layer automatically applies business logic.
**Why It Matters**: Tableau's calculated fields ARE a semantic layer. Extracting existing calculated field definitions from TWB XML and including them in prompts gives the model the exact business logic it needs. This is perhaps the highest-impact pattern for the Tableau optimizer.
**Reference**: https://cloud.google.com/blog/products/databases/techniques-for-improving-text-to-sql

### BigQuery dialect-specific patterns

**Finding**: SQL dialect differences cause subtle, hard-to-detect errors. Google Cloud fine-tunes models specifically for dialect correctness. SQL-GEN (2024) creates dialect-specific synthetic training data, boosting BigQuery performance by **4–27%**.
**Example**: BigQuery-specific prompt section: "Generate BigQuery Standard SQL. Use `EXTRACT()` for date parts, `ARRAY_AGG` for arrays, `UNNEST` for repeated fields, backtick notation for `project.dataset.table` references. Use `PARTITION BY` in window functions. Do NOT use `LIMIT` with `OFFSET` for pagination — use `ROW_NUMBER() OVER()` instead."
**Why It Matters**: The Tableau optimizer skill must include explicit BigQuery dialect rules since Tableau frequently connects to BigQuery and any generated SQL must be dialect-correct.
**Reference**: https://cloud.google.com/blog/products/databases/techniques-for-improving-text-to-sql

**Finding**: A practitioner-tested BigQuery prompt template achieves **94% accuracy**: clear goal, return format, dialect warnings (cost, partitioning), full table schema. Includes cost optimization hints (partition filtering, column selection) and references a SQL style guide.
**Example**: "You are a BigQuery SQL expert. Generate cost-optimized BigQuery Standard SQL. Always filter on partition columns. Use `project.dataset.table` notation. Include cost estimation comments. Prefer `APPROX_COUNT_DISTINCT` over `COUNT(DISTINCT)` for large tables."
**Why It Matters**: Include BigQuery cost-optimization rules in the Tableau optimizer skill. Tableau extracts run BigQuery queries that cost real money — partition filtering alone can reduce costs 10–100x.
**Reference**: https://datatovalue.blog/quick-tip-bigquery-llm-prompt-43b72657455b

### Decomposition dramatically improves SQL quality

**Finding**: Breaking SQL generation into sub-steps (schema linking → plan → SQL → self-correction) significantly outperforms single-shot generation. DIN-SQL achieves **85.3%** on Spider through decomposition. Research shows 1–2 self-correction rounds are optimal — more rounds yield diminishing returns.
**Example**: Step 1: "Identify relevant tables and columns." Step 2: "Plan the join path and aggregation levels." Step 3: "Write the SQL using the plan." Step 4: "Execute and verify results are non-empty and reasonable."
**Why It Matters**: The Tableau optimizer skill should decompose SQL analysis into explicit phases: identify tables → verify joins → check aggregations → validate filters → optimize for cost.
**Reference**: https://aclanthology.org/2024.findings-acl.641.pdf

**Finding**: Set **temperature to 0** and use post-processing for deterministic SQL generation. The non-deterministic nature of LLMs means identical prompts can produce different SQL. Multi-path generation with candidate selection improves reliability.
**Example**: `temperature=0` in API calls; regex extraction of SQL between code fences; validation that output parses as valid SQL before execution.
**Why It Matters**: Any SQL generated for Tableau Custom SQL data sources or BigQuery scheduled queries must be deterministic and validated before deployment.
**Reference**: https://www.vldb.org/pvldb/vol17/p1132-gao.pdf

### Working with large Tableau workbook files

**Finding**: For files exceeding context limits, **hierarchical summarization** at semantic boundaries works best. For XML specifically, structure-aware chunking at tag boundaries preserves meaning. Overlapping chunks (10–20%) maintain continuity.
**Example**: For a 500KB TWB file: Parse XML → Extract sections (datasources, worksheets, dashboards, calculated fields) → Summarize each section independently → Combine summaries for holistic analysis.
**Why It Matters**: TWB/TWBX files can exceed context limits. The skill should instruct: "Parse the TWB XML. Extract `<datasource>`, `<worksheet>`, and `<dashboard>` elements separately. Analyze each section independently, then synthesize findings."
**Reference**: https://www.typedef.ai/resources/tackle-chunking-context-windows-llm-data-pipelines

**Finding**: TWB files are XML containing complete workbook metadata: data source connections, table schemas, join structures, calculated field formulas, filters, parameters, and dashboard layouts. Calculated fields live in `<column>` elements with `<calculation>` children. Custom SQL lives in `<relation type='text'>` elements.
**Example**: Extract all calculated fields: Parse TWB XML → Find all `<column>` elements with `<calculation>` children → Extract `caption` attribute (display name) and `formula` attribute (calculation logic). Custom SQL lives inside `<relation type='text'>` elements within datasources.
**Why It Matters**: This is the core data extraction step for the Tableau optimizer. A Python script in the skill's `scripts/` directory should handle XML parsing and feed structured data to the LLM.
**Reference**: https://tableauandbehold.com/2016/06/29/how-tds-twb-files-work-xml/

### Iterative refinement and error handling

**Finding**: The Self-Refine framework (Generate → Feedback → Refine) improves performance **5–40%** across tasks. Key insight: feedback must be actionable — localizing the problem AND providing improvement instructions. Only **1–2 self-correction rounds** are optimal; more rounds risk circular improvements.
**Example**:
```
Round 1: "Review this SQL for correct JOIN paths only."
Round 2: "Now review the aggregation logic."
Round 3: "Optimize for BigQuery cost (partition pruning, column selection)."
```
**Why It Matters**: The Tableau optimizer should refine one component at a time: fix joins first, then aggregations, then filters, then cost optimization — not all at once.
**Reference**: https://selfrefine.info/

**Finding**: Production LLM pipelines need **retries with prompt variation** (don't retry identical prompts — 40% will repeat the same failure), circuit breakers, and graceful degradation. If the LLM generates invalid SQL, feed the error message back; if it generates empty results, retry with relaxed filters; if all attempts fail, present the best attempt for human review.
**Example**: Retry strategy: Attempt 1 (standard prompt) → Attempt 2 (add "Think step by step" + examples) → Attempt 3 (simplify the question, break into sub-queries) → Fallback (return partial result with warning).
**Why It Matters**: The Tableau optimizer should never silently fail. Build validation at each step with clear fallback paths and escalation to human review.
**Reference**: https://www.gocodeo.com/post/error-recovery-and-fallback-strategies-in-ai-agent-development

**Finding**: **Semantic checkpointing** — storing context summaries and task state at intervals — enables recovery without reprocessing entire histories.
**Example**: Write to file after each phase: "Phase 1 complete: Found 5 data sources, 3 using Custom SQL. Phase 2 complete: Identified 47 calculated fields, 12 LOD expressions. Phase 3: Analyzing performance..."
**Why It Matters**: A Tableau workbook audit that crashes mid-analysis can resume from the last checkpoint rather than reprocessing the entire TWB file. Build checkpointing into the skill workflow.
**Reference**: https://www.researchgate.net/publication/399564250_Fault_Tolerance_and_Recovery_Strategies_for_LLM_Agents_in_Distributed_Systems

**Finding**: The **LLM agent pattern** (ReAct-style) with tool access achieves higher SQL accuracy than single-shot prompting. Give the LLM tools: List Tables, Get Schema, Execute SQL, Receive Results. The LLM plans, executes, handles errors, and self-corrects. Google Cloud uses multi-stage retrieval: vector search → schema loading → few-shot retrieval → SQL generation → validation.
**Example**: Agent workflow: "Find tables about sales" → [list_tables] → "Get schema for orders" → [get_schema] → "Write SQL" → [execute_sql] → "Error: column not found" → [corrects and retries] → success.
**Why It Matters**: The Tableau optimizer can use this pattern: read TWB XML → extract table references → query BigQuery for actual schemas → validate field references → generate optimized SQL.
**Reference**: https://medium.com/@vi.ha.engr/bridging-natural-language-and-databases-best-practices-for-llm-generated-sql-fcba0449d4e5

---

## Conclusion: design principles for the Tableau optimizer skill

This research converges on several non-obvious principles that should directly shape the Gemini CLI Tableau optimizer skill.

**Context is the constraint, not intelligence.** The shift from "prompt engineering" to "context engineering" (Andrej Karpathy's framing) reflects the core finding: filling the context window with the right information matters more than clever phrasing. For the Tableau optimizer, this means extracting only relevant TWB XML sections, providing only used table schemas, and aggressively managing context across analysis phases.

**Progressive disclosure is the architecture.** Agent Skills' three-tier loading (metadata → SKILL.md → reference files) directly solves the context bloat problem. The Tableau optimizer should have a lean SKILL.md under 150 lines with deep reference material in separate files that load on demand.

**External verification is non-negotiable.** Self-correction without external feedback (BigQuery dry runs, XML parsing validation, SQL syntax checking) is unreliable. Every generated artifact should be validated through tools, not introspection. The skill should include validation scripts in its `scripts/` directory.

**Gated execution prevents runaway errors.** The analyze → plan → implement pattern with explicit user gates between phases prevents the compounding error problem. Never let the model modify a TWB file without presenting and confirming its analysis first.

**Schema + semantic layer = accuracy.** Providing DDL with sample rows and existing calculated field definitions (Tableau's semantic layer) is the single highest-impact accuracy improvement, potentially moving SQL generation accuracy from ~50% to 90%+. Extract these from the TWB XML as the first step in every optimization workflow.

**Concision wins.** Across every dimension — GEMINI.md, SKILL.md, system prompts, few-shot examples — shorter and more focused consistently outperforms comprehensive and verbose. Two or three well-chosen examples beat ten. One hundred focused lines beat three hundred thorough ones. Every instruction must earn its place by preventing a specific observed failure.