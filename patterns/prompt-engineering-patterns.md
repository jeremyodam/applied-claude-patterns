# Pattern: 17 Prompt Engineering Patterns — Production Reference

*Vanderbilt Generative AI Specialization — Mapped to buddyOS Architecture*

These 17 patterns form the prompting foundation of the buddyOS platform. Each is documented with its definition, its specific application inside a production multi-tenant AI system, and a working example prompt. Drop this file into any Claude Code session as instant context for the full pattern stack.

---

## 1. Persona Pattern

**Definition:** Assigns the model a specific role or identity before it begins responding.

**Production Application:** Every Buddy module opens with a persona preamble — FieldBuddy is a 25-year gas ops technician, not a general assistant. The persona anchors tone, vocabulary, and risk calibration for the entire session.

**Example Prompt:**
```
Act as a certified gas distribution technician with 25 years of field experience.
When I describe a field condition, respond as that technician would.
```

---

## 2. Audience Persona Pattern

**Definition:** Tells the model who it is speaking to, so it calibrates vocabulary, depth, and tone accordingly.

**Production Application:** SafetyBuddy adjusts output for a field tech vs. a supervisor vs. a regulator — same question, different answer depth. The audience persona is injected from the tenant config at request time.

**Example Prompt:**
```
My audience is a first-year service technician with no regulatory background.
Explain this NFPA 54 requirement in plain language they can act on immediately.
```

---

## 3. Flipped Interaction Pattern

**Definition:** Instructs the model to ask the user questions rather than immediately providing answers — driving toward a goal through dialogue.

**Production Application:** CounselBuddy uses this to gather facts before generating a statement of expectations or corrective action document. It will not draft until it has asked at least five qualifying questions.

**Example Prompt:**
```
I need to write a corrective action document. Ask me one question at a time
to gather the facts. Do not draft anything until you have asked at least five questions.
```

---

## 4. Game Play Pattern

**Definition:** Creates an interactive game or simulation structure with defined rules, scoring, and progression.

**Production Application:** GuitarBuddy uses this for ear training and chord recognition drills. Future use: SafetyBuddy field scenario simulator scored against CFR 192.

**Example Prompt:**
```
We are going to play a gas leak scenario drill. You will describe a field condition.
I will respond with my actions. You will score each response against 49 CFR 192
and tell me what I missed.
```

---

## 5. Template Pattern

**Definition:** Provides a fill-in-the-blank structure where the model populates placeholders with contextually correct content.

**Production Application:** SupervisorBuddy generates performance review drafts from a template populated with the employee's actual data. The template is stored in tenant config and injected at request time — never hardcoded in the model prompt.

**Example Prompt:**
```
Use this template for every field incident summary:
Date: [DATE] | Location: [ADDRESS] | Hazard: [HAZARD TYPE] |
Risk Level: [CRITICAL/HIGH/LOW] | Regulation: [CFR REF] |
Action Taken: [ACTION] | Follow-up Required: [YES/NO]
```

---

## 6. Meta Language Creation Pattern

**Definition:** Defines a custom shorthand vocabulary inside the prompt so the model interprets field-specific terms correctly every session.

**Production Application:** FieldBuddy's vocabulary preamble — SV, ILI, JB, LOTO, OOS — is this pattern in production. Built by operations, for operations. The vocabulary block is versioned and maintained by the operations team, not the dev team.

**Example Prompt:**
```
In this session: SV = service valve, ILI = inline inspection, JB = junction box,
LOTO = lockout/tagout, OOS = out of service.
Interpret all field conditions using these definitions.
```

---

## 7. Recipe Pattern

**Definition:** Provides a known start state and a known end state, then asks the model to fill in all required intermediate steps in sequence.

**Production Application:** FieldBuddy leak response workflow — from scene arrival to service restoration and documentation — is a Recipe Pattern. The start and end states are fixed by regulation; the model fills the procedural middle.

**Example Prompt:**
```
Starting point: technician arrives on scene at reported gas odor, customer evacuated.
End point: service restored and incident documented.
Provide every step in sequence, referencing applicable regulations.
```

---

## 8. Alternative Approaches Pattern

**Definition:** Instructs the model to generate multiple distinct solutions to a problem rather than a single answer, forcing comparison and trade-off thinking.

**Production Application:** DigBuddy uses this when a locate request has ambiguous site conditions — present three excavation approaches with risk ratings, let the crew decide.

**Example Prompt:**
```
Give me three alternative approaches to completing this meter set replacement
under active pressure. For each approach, list the steps, required equipment,
and risk level.
```

---

## 9. Ask for Input Pattern

**Definition:** Ends the model's response with a prompt asking the user what to do next, keeping the session interactive rather than terminal.

**Production Application:** Every Buddy session ends with a directed next-action question, not a dead stop. This pattern is enforced in the base system prompt so individual Buddy configs don't have to repeat it.

**Example Prompt:**
```
After providing each field recommendation, always end with:
What would you like to do next — document this incident,
escalate to dispatch, or assess an additional hazard?
```

---

## 10. Combining Patterns

**Definition:** Stacks two or more patterns in a single prompt to produce compound behavior — Persona + Recipe + Template all firing together.

**Production Application:** Every production Buddy module is a combination. FieldBuddy = Persona + Meta Language + Recipe + Template + Ask for Input. That combined stack is what lives in the `CLAUDE.md` system prompt.

**Example Prompt:**
```
Act as a senior gas technician (Persona).
Use field vocabulary: SV, LOTO, OOS (Meta Language).
Walk me through leak response start to finish (Recipe).
Output in incident summary format (Template).
Ask me what's next when done (Ask for Input).
```

---

## 11. Outline Expansion Pattern

**Definition:** Starts with a high-level outline and progressively expands each section into full content on demand.

**Production Application:** Used in buddyOS documentation builds and enterprise business case development — outline first, expand section by section. Prevents the model from hallucinating a 10,000-word document that goes off the rails by page 3.

**Example Prompt:**
```
Create a high-level outline for a FieldBuddy implementation guide.
Then ask me which section to expand first.
Expand one section at a time and ask before proceeding to the next.
```

---

## 12. Menu Actions Pattern

**Definition:** Defines trigger words or commands that fire specific model behaviors — a command vocabulary for the session.

**Production Application:** FieldBuddy command triggers: `INSPECT` launches inspection checklist, `ESCALATE` drafts dispatch notification, `DOCUMENT` opens incident summary template. This turns a chat interface into a command interface without any UI work.

**Example Prompt:**
```
Whenever I type INSPECT, generate a step-by-step inspection checklist
for the condition I describe.
Whenever I type DOCUMENT, generate an incident summary.
Whenever I type ESCALATE, draft a dispatch notification.
Ask me for the first command.
```

---

## 13. Fact Check List Pattern

**Definition:** Instructs the model to generate a list of claims it made and flag any it is uncertain about, surfacing hallucination risk explicitly.

**Production Application:** Critical for CorrosionBuddy and SafetyBuddy — any regulatory reference must be fact-check listed before the technician acts on it. This pattern is what makes AI usable in regulated-industry workflows.

**Example Prompt:**
```
After providing any regulatory guidance, generate a fact check list of every
specific CFR citation you used. Flag any citation you are less than 100%
confident about with [VERIFY BEFORE USE].
```

---

## 14. Tail Generation Pattern

**Definition:** Appends a consistent closing element to every response — a question, a prompt, a summary line — ensuring the session never goes dead.

**Production Application:** Pairs with Ask for Input (#9). Every Buddy response closes with a tail: next action prompt, confidence rating, and source reference. Baked into the base system prompt, not optional.

**Example Prompt:**
```
At the end of every response, add this tail:
Confidence: [HIGH/MEDIUM/LOW] | Source: [REGULATION OR BEST PRACTICE] |
Next: [SUGGESTED NEXT ACTION]
```

---

## 15. Semantic Filter Pattern

**Definition:** Instructs the model to rewrite or filter content through a specific lens — removing jargon, adjusting reading level, or applying a tone.

**Production Application:** SalonBuddy translates stylist service notes into client-facing appointment summaries. FieldBuddy translates regulatory language into plain field instructions. Same underlying data, different output register.

**Example Prompt:**
```
Rewrite this 49 CFR 192 section as a plain-language checklist a first-year
technician can follow in the field. Remove all legal and regulatory language.
Keep only actionable steps.
```

---

## 16. Context Manager Pattern

**Definition:** Explicitly tells the model what context to ignore, what to focus on, and what to carry forward — managing the session's memory intentionally.

**Production Application:** Used in long Buddy sessions to prevent context drift. When a technician moves from one work order to another mid-session, the Context Manager pattern resets the working frame without losing the persona or vocabulary preamble.

**Example Prompt:**
```
For this session, ignore all previous examples we discussed.
Focus only on the following work order: [WORK ORDER].
Carry forward only the equipment list and the site address.
```

---

## 17. Few-Shot Pattern

**Definition:** Teaches the model a desired output format by providing two or three examples before making the real request — no explicit instructions needed.

**Production Application:** The foundation of FieldBuddy's incident classification system — show the model three classified examples, then hand it a live incident. Consistently outperforms zero-shot classification on regulatory edge cases.

**Example Prompt:**
```
Example 1: Gas odor inside structure → Critical → Evacuate and dispatch emergency crew.
Example 2: Pilot light out → High → Schedule service within 24 hours.
Example 3: High bill complaint → Low → Route to billing.

Now classify: Technician reports corroded fitting at meter set during routine inspection.
```

---

## Real-World Note

These 17 patterns are not academic exercises — they are the active system prompt architecture powering 12+ production Buddy modules deployed across enterprise operational teams. Every pattern listed above maps to a specific system prompt section or behavioral rule in a live, customer-facing application.

The patterns were formalized during the Vanderbilt Generative AI Software Engineering Specialization (Certificate ID: 8HA10H8ELG80). The production implementations predate the formalization — recognizing that you've already been doing it is most of the learning.

The most powerful insight: production system prompts are almost always a **Combining Pattern** (Pattern #10) — Persona + Meta Language + Recipe + Template + Ask for Input + Tail Generation, all stacked. When a Buddy module doesn't perform, the fix is almost always identifying which layer of the stack is missing.
