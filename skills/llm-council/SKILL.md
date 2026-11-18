---
name: llm-council
description: >-
  Run any question, idea, or decision through a council of 5 independent AI advisors who analyze it from distinct thinking angles, peer-review each other anonymously, and synthesize a final verdict. Based on Andrej Karpathy's LLM Council methodology.
  MANDATORY TRIGGERS (EN): 'council this', 'run the council', 'war room this', 'pressure-test this', 'stress-test this', 'debate this'.
  MANDATORY TRIGGERS (PT): 'rodar conselho', 'rodar o conselho', 'passar no conselho', 'conselho nisso', 'sala de guerra', 'estressar isso', 'testar essa ideia', 'testar essa decisão', 'debater isso'.
  STRONG TRIGGERS (use when combined with a real decision or tradeoff): 'should I X or Y', 'which option', 'what would you do', 'is this the right move', 'validate this', 'get multiple perspectives', 'I can't decide', 'I'm torn between', 'devo fazer X ou Y', 'qual opção', 'o que você faria', 'é a escolha certa', 'me ajude a decidir', 'estou em dúvida entre'.
  Do NOT trigger on simple yes/no questions, factual lookups, or casual tradeoffs without stakes (e.g. 'should I use markdown' is not a council question). DO trigger when the user presents a genuine decision with stakes, multiple options, and context that benefits from multi-angle pressure-testing.
---

# LLM Council

You ask one AI a question, you get one answer. That answer might be great. It might be mid. You have no way to tell because you only saw one perspective.

The council fixes this. It runs your question through 5 independent advisors, each thinking from a fundamentally different angle. Then they review each other's work anonymously. Then a chairman synthesizes everything into a final recommendation that tells you where the advisors agree, where they clash, and what you should actually do.

This is adapted from Andrej Karpathy's LLM Council. He dispatches queries to multiple models, has them peer-review each other anonymously, then a chairman produces the final answer. Here we run the exact same methodology natively inside Antigravity using isolated parallel subagents (`define_subagent` + `invoke_subagent`) with distinct thinking lenses.

---

## Global Rules

1. **Language Matching Rule:** Detect the language of the user's question (e.g., Portuguese or English). Instruct every advisor, peer reviewer, and the chairman to write their **entire** output—including analysis, critiques, headings, and verdict—in the **same language as the user's question**.
2. **Parallelism Rule:** Always invoke all 5 advisors in parallel in a single turn (Step 2), and all 5 peer reviewers in parallel in a single turn (Step 3). Never run them sequentially.
3. **Anonymization Rule:** Always shuffle/permute the 5 advisor responses before labeling them `Response A` through `Response E` for Step 3. Never pass advisor names to peer reviewers.
4. **Isolation Rule:** Advisors and peer reviewers must NOT modify files, run commands, or independently search the workspace. Only the coordinator gathers context in Step 1.
5. **Strict Synchronization Barrier Rule:** Antigravity delivers each subagent's reply asynchronously as a separate `<SYSTEM_MESSAGE>` across one or more wakeups. When waiting for a parallel batch of 5 subagents (Step 2 or Step 3):
   - Record the 5 `conversationId` values returned by `invoke_subagent`.
   - End your turn cleanly (call no more tools, and do **NOT** emit repetitive progress text like `"Aguardando 1/5... 2/5..."` or simulate remaining turns).
   - On each wakeup, count how many **distinct** `sender` IDs from the current batch have delivered a `<SYSTEM_MESSAGE>` in the conversation history so far.
   - If fewer than all 5 `sender` IDs have arrived, **end your turn again immediately** without calling any tools.
   - **NEVER** start Step 3 until all 5 advisor `<SYSTEM_MESSAGE>` replies are present verbatim, and **NEVER** start Step 4 until all 5 peer review `<SYSTEM_MESSAGE>` replies are present verbatim. Never invent, extrapolate, or placeholder-fill missing subagent responses.

---

## When to Run the Council

The council is for questions where being wrong is expensive.

Good council questions:
- "Should I launch a $97 workshop or a $497 course?"
- "Devo pedir demissão agora para construir meu micro-SaaS ou validar nos fins de semana?"
- "Which of these 3 positioning angles is strongest?"
- "I'm thinking of pivoting from X to Y. Am I crazy?"
- "Here's my landing page copy. What's weak?"
- "Should I hire a VA or build an automation first?"

Bad council questions:
- "What's the capital of France?" (one right answer, no need for perspectives)
- "Write me a tweet" (creation task, not a decision)
- "Summarize this article" (processing task, not judgment)

The council shines when there's genuine uncertainty and the cost of a bad call is high. If the user already knows the answer and just wants validation, the council will likely tell them things they don't want to hear. That's the point.

---

## The Five Advisors

Each advisor thinks from a different angle. They're not job titles or personas. They're thinking styles that naturally create tension with each other.

### 1. The Contrarian
Actively looks for what's wrong, what's missing, what will fail. Assumes the idea has a fatal flaw and tries to find it. If everything looks solid, digs deeper. The Contrarian is not a pessimist. They're the friend who saves you from a bad deal by asking the questions you're avoiding.

### 2. The First Principles Thinker
Ignores the surface-level question and asks "what are we actually trying to solve here?" Strips away assumptions. Rebuilds the problem from the ground up. Sometimes the most valuable council output is the First Principles Thinker saying "you're asking the wrong question entirely."

### 3. The Expansionist
Looks for upside everyone else is missing. What could be bigger? What adjacent opportunity is hiding? What's being undervalued? The Expansionist doesn't care about risk (that's the Contrarian's job). They care about what happens if this works even better than expected.

### 4. The Outsider
Has zero context about the user, their field, or their history. Responds purely to what's in front of them. This is the most underrated advisor. Experts develop blind spots. The Outsider catches the curse of knowledge: things that are obvious to insiders but confusing to everyone else.

### 5. The Executor
Only cares about one thing: can this actually be done, and what's the fastest path to doing it? Ignores theory, strategy, and big-picture thinking. The Executor looks at every idea through the lens of "OK, but what do you do Monday morning?" If an idea sounds brilliant but has no clear first step, the Executor will say so.

**Why these five:** They create three natural tensions. Contrarian vs. Expansionist (downside vs. upside). First Principles vs. Executor (rethink everything vs. just do it). The Outsider sits in the middle keeping everyone honest by seeing what fresh eyes see.

---

## How a Council Session Works

### Step 1: Frame the Question (with Context Enrichment)

When triggered (via `/llm-council`, `/council`, or a natural trigger phrase), do two things before convening the council:

**A. Scan the workspace for context (when applicable).** If the user is inside a project workspace and the question relates to their work, product, or codebase, quickly scan for and read relevant context files using native search/read tools (e.g., `run_command` with `rg`/`Get-ChildItem` and `read_file`):
- `GEMINI.md`, `AGENTS.md`, or `.agents/rules/*.md` in the workspace (project context, preferences, constraints)
- `CLAUDE.md` / `claude.md` or `memory/` folders if present
- `README.md`, product docs, or any files the user explicitly referenced or attached
- Recent `council-transcript-*.md` files in the workspace (to avoid re-counciling the exact same ground)
- Any other context files relevant to the specific question (e.g., pricing docs, audience notes, specs)

Do not spend more than ~30 seconds on this. You are looking for the 2-3 files that give advisors the context they need for specific, grounded advice instead of generic takes. If the question is standalone/general (e.g., a smoke test or personal dilemma with no relevant workspace files), skip extra file scanning and proceed directly.

**B. Frame the question.** Combine the user's raw question AND any enriched context into a clear, neutral prompt that all five advisors will receive. The framed question should include:
1. The core decision or question
2. Key context from the user's message
3. Key context from workspace files (stage, audience, constraints, relevant numbers), if any
4. What's at stake (why this decision matters)

Don't add your own opinion. Don't steer it. Make sure each advisor has enough context to give a specific, grounded answer—except keep in mind **The Outsider** relies on fresh-eyes clarity, so avoid insider jargon without brief explanation if it isn't part of the user's framing.

If the question is too vague (e.g., `"council this: my business"`), ask **one** clarifying question. Just one. Wait for the user's reply, then proceed.

Save the framed question in memory for the council rounds (and for the transcript if requested later).

---

### Step 2: Convene the Council (5 Subagents in Parallel)

First, register a clean, read-only subagent type once using `define_subagent` (if not already defined in this conversation):
- `Name`: `"council-member"`
- `Description`: `"Isolated analytical member of an LLM Council (Advisor, Peer Reviewer, or Chairman)."`
- `EnableWriteTools`: `false`
- `EnableMCPTools`: `false`
- `EnableSubagentTools`: `false`
- `SystemPrompt`:
  > You are an analytical member of an LLM Council. Follow your assigned role strictly. Analyze only from the prompt provided—do NOT read workspace files, search directories, or inspect external context. Always respond completely in the same language as the user's question.

*(If `define_subagent` is unavailable in the current environment, fall back to `TypeName: "self"` in `invoke_subagent` with the same strict instruction in the prompt.)*

Now spawn **all 5 advisors simultaneously in a single turn** by calling `invoke_subagent` 5 times (`TypeName: "council-member"`):
1. `Role`: `"Advisor: The Contrarian"`
2. `Role`: `"Advisor: First Principles"`
3. `Role`: `"Advisor: The Expansionist"`
4. `Role`: `"Advisor: The Outsider"`
5. `Role`: `"Advisor: The Executor"`

After issuing the 5 `invoke_subagent` calls, record their 5 `conversationId` values, **stop calling tools**, and end your turn cleanly (no progress-counter narration). On each wakeup, check whether all 5 advisor `sender` IDs have delivered a `<SYSTEM_MESSAGE>`; if fewer than 5 have arrived, end your turn again and wait for the remaining reactive wakeups before moving to Step 3.

**Advisor subagent prompt template:**

```text
You are [Advisor Name] on an LLM Council.

Your thinking style: [advisor description from "The Five Advisors" above]

A user has brought this question to the council:

---
[framed question]
---

Instructions:
- Respond from your assigned perspective. Be direct and specific.
- Don't hedge or try to be balanced. Lean fully into your assigned angle. The other advisors will cover the angles you're not covering.
- Do NOT read workspace files or search external context; rely strictly on the framed question above.
- LANGUAGE: Write your entire response in the exact same language as the user's question (e.g., Portuguese if the question is in Portuguese, English if in English).
- Keep your response between 150-300 words. No preamble. Go straight into your analysis.
```

---

### Step 3: Peer Review (5 Subagents in Parallel — Anonymous)

This is the step that makes the council more than just "ask 5 times." It's the core of Karpathy's insight.

1. **Verify barrier first:** Confirm that all 5 advisor `<SYSTEM_MESSAGE>` replies (matching all 5 Step 2 `conversationId`s) have actually arrived in the conversation context. Use the exact verbatim responses received—never reconstruct or invent any advisor response.
2. **Randomize / permute the order** of the 5 advisors (do NOT keep the fixed 1–5 order so there is no positional bias) and label them `Response A` through `Response E`.
3. Keep the mapping (`Response A -> Advisor X`, ..., `Response E -> Advisor Y`) strictly private for Step 4. Never reveal advisor identities to the reviewers.
4. Spawn **5 new `invoke_subagent` calls simultaneously in a single turn** (`TypeName: "council-member"`), with `Role` `"Peer Reviewer 1"` through `"Peer Reviewer 5"`.
5. Record the 5 reviewer `conversationId` values, **stop calling tools**, and end your turn cleanly (no progress-counter narration). On each wakeup, verify whether all 5 reviewer `sender` IDs have delivered a `<SYSTEM_MESSAGE>`; if fewer than 5 have arrived, end your turn again and wait.

**Reviewer subagent prompt template:**

```text
You are reviewing the outputs of an LLM Council. Five advisors independently answered this question:

---
[framed question]
---

Here are their anonymized responses:

**Response A:**
[response]

**Response B:**
[response]

**Response C:**
[response]

**Response D:**
[response]

**Response E:**
[response]

Answer these three questions. Be specific. Reference responses by letter (A-E).

1. Which response is the strongest? Why? (pick one)
2. Which response has the biggest blind spot? What is it missing?
3. What did ALL five responses miss that the council should consider?

Instructions:
- Do NOT read workspace files or search external context.
- LANGUAGE: Write your entire review in the exact same language as the user's question.
- Keep your review under 200 words. Be direct.
```

---

### Step 4: Chairman Synthesis

This is the final analytical step. First, **verify the Step 3 barrier**: confirm that all 5 peer reviewer `<SYSTEM_MESSAGE>` replies (matching all 5 Step 3 `conversationId`s) have actually arrived verbatim in the conversation context.

Then invoke **1 subagent** (`TypeName: "council-member"`, `Role: "Council Chairman"`) that receives everything: the framed question, all 5 advisor responses **de-anonymized** (showing both the advisor name and their anonymous review letter `[Response X]` so the Chairman can accurately map peer critiques), and all 5 peer reviews.

Stop calling tools and end your turn cleanly until the Chairman's `<SYSTEM_MESSAGE>` arrives.

**Chairman subagent prompt template:**

```text
You are the Chairman of an LLM Council. Your job is to synthesize the work of 5 advisors and their peer reviews into a final verdict.

The question brought to the council:
---
[framed question]
---

ADVISOR RESPONSES (de-anonymized with their Step 3 letter):

**The Contrarian [Response X]:**
[response]

**The First Principles Thinker [Response Y]:**
[response]

**The Expansionist [Response Z]:**
[response]

**The Outsider [Response W]:**
[response]

**The Executor [Response V]:**
[response]

PEER REVIEWS:
[all 5 peer reviews]

Produce the council verdict using this exact 5-section structure (translate the section headings to match the user's question language — e.g., in Portuguese use: "Onde o Conselho Concorda", "Onde o Conselho Diverge", "Pontos Cegos Detectados pelo Conselho", "A Recomendação", "A Primeira Coisa a Fazer"):

## Where the Council Agrees
[Points multiple advisors converged on independently. These are high-confidence signals.]

## Where the Council Clashes
[Genuine disagreements. Present both sides. Explain why reasonable advisors disagree.]

## Blind Spots the Council Caught
[Things that only emerged through peer review. Things individual advisors missed that others flagged.]

## The Recommendation
[A clear, direct recommendation. Not "it depends." A real answer with reasoning. You may disagree with the majority if a dissenter's reasoning is stronger.]

## The One Thing to Do First
[A single concrete next step. Not a list. One thing.]

Instructions:
- Be direct. Don't hedge. The whole point of the council is to give the user clarity they couldn't get from a single perspective.
- Do NOT read workspace files or search external context.
- LANGUAGE: Write the entire verdict—including headings—in the exact same language as the user's question.
```

---

### Step 5: Present the Verdict in Chat

After the Chairman synthesis is complete, present the full verdict directly in chat using scannable markdown. Do **NOT** generate an HTML report or unsolicited files. The user reads it directly in the conversation.

Format the output in the user's language:

**If English:**
```markdown
## Council Verdict: {short topic}

### Where the Council Agrees
{content}

### Where the Council Clashes
{content}

### Blind Spots the Council Caught
{content}

### The Recommendation
{content}

### The One Thing to Do First
{content}
```

**If Portuguese:**
```markdown
## Veredito do Conselho: {assunto curto}

### Onde o Conselho Concorda
{conteúdo}

### Onde o Conselho Diverge
{conteúdo}

### Pontos Cegos Detectados pelo Conselho
{conteúdo}

### A Recomendação
{conteúdo}

### A Primeira Coisa a Fazer
{conteúdo}
```

Keep it scannable. Use bullet points and include concrete before/after examples where relevant.

---

### Step 6: Save the Transcript (Optional)

Only save a transcript if the user asks for it (either upfront in their prompt or after seeing the verdict).

When saving:
1. If the user specified a destination path or filename, write the transcript to that exact location.
2. Otherwise, save the file in the **root of the active workspace** as `council-transcript-YYYYMMDD-HHMMSS.md`.
3. Include in the saved file: the framed question, all 5 advisor responses, the Step 3 anonymization map (`A–E`), all 5 peer reviews, and the Chairman's final verdict.
4. After saving, inform the user of the exact file path (with a clickable `file://` link) and **end by asking whether saving at the workspace root is good or if they would like to move it to another location**.

---

## Example: Counciling a Product Decision

**User:** "Council this: I'm thinking of building a $297 course on AI coding for beginners. My audience is mostly non-technical solopreneurs. Is this the right move?"

**The Contrarian:** "The market is flooded with AI courses right now. At $297, you're competing with free YouTube content. Your audience is non-technical, which means high support burden and refund risk. The people who would pay $297 are likely already past beginner level..."

**The First Principles Thinker:** "What are you actually trying to achieve? If it's revenue, a course is one of the slowest paths. If it's authority, a free resource might do more. If it's building a customer base for higher-ticket offers, the price point and audience might be mismatched..."

**The Expansionist:** "Beginner AI workflows for solopreneurs is a massive underserved market. Everyone's teaching advanced stuff. If you nail the beginner angle, you own the entry point to this entire space. The $297 might be low. What if this became a $997 program with community access..."

**The Outsider:** "I don't know what AI coding tools mean in practice. If I saw '$297 course on AI coding for beginners,' I wouldn't know if this is for me. The jargon means nothing to someone outside your world. Your landing page needs to sell the outcome, not the tool..."

**The Executor:** "A full course takes 4-8 weeks to produce properly. Before building anything, run a live workshop at $97 to 50 people. You validate demand, generate testimonials, and create the raw material for the course. If 50 people don't buy the workshop, 500 won't buy the course..."

**Chairman's Verdict:**
- *Where the council agrees:* The beginner solopreneur angle has real demand, but the current tool-specific framing won't resonate with non-technical buyers.
- *Where the council clashes:* Price. The Contrarian says $297 is too high given competition. The Expansionist says it's too low for the value. The resolution depends on how much support and community access is bundled.
- *Blind spots caught:* The Outsider's point that the technical tool name means nothing to the target buyer is the single most important insight. Every advisor except the Outsider assumed the audience already knows what this is.
- *Recommendation:* Don't build the course yet. Validate with a lower-commitment offer first, and reframe entirely: sell the outcome (automate your business, get 10 hours back per week), not the tool.
- *One thing to do first:* Run a $97 live workshop called "How to automate your first business task with AI" to 50 people without mentioning developer jargon in the title.

---

## Important Notes

- **Always spawn all 5 advisors in parallel.** Sequential spawning wastes time and lets earlier responses bleed into later ones.
- **Always anonymize and shuffle for peer review.** If reviewers know which advisor said what (or see them in a fixed predictable order), they'll defer to styles or positions instead of evaluating on merit.
- **The chairman can disagree with the majority.** If 4 out of 5 advisors say "do it" but the reasoning of the 1 dissenter is strongest, the chairman should side with the dissenter and explain why.
- **Don't council trivial questions.** If the user asks something with one right answer, just answer it directly. The council is for genuine uncertainty where multiple perspectives add value.
- **Scannability matters.** Most users will scan the verdict in chat, not read raw transcripts. Keep the Markdown output structured, crisp, and direct.
