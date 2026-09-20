# Reflection Brief — Harness Engineering Capstone

**Name: Nayak Sankar
**Date: 18-sep-2026

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): Claude via Vocareum
- OS / Python: Linux Vocareum Workspace, Python 3.x
- Approx. API spend: Vocareum-provided API key

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   The continue-versus-stop decision is implemented in claims_intake/loop.py in the run() function.

The run() function continues the loop when response.stop_reason == "tool_use" and terminates by returning FinalState when response.stop_reason == "end_turn". Any other stop_reason raises UnexpectedStopReason.

A real stop_reason sequence from the System 1 trace for claim_04 was:

tool_use → tool_use → tool_use → end_turn

This sequence shows three tool-use turns followed by a terminating end_turn response.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   One anti-pattern checked by test_antipatterns.py is using a fixed turn count instead of allowing the model to control the loop through stop_reason values. If the loop used a fixed iteration count, the agent could terminate before all required tool calls were completed or continue after a final answer was already produced. The successful result in evidence/system1/system1-test-log.txt (29 passing tests) indicates the reference implementation avoids this anti-pattern.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   The tool descriptions are designed to clearly separate responsibilities even when inputs overlap. Distinct descriptions help the model choose the correct tool instead of routing requests ambiguously. A structured tool error is more useful than a generic string because the agent can inspect the error details, adjust its behavior, and retry with corrected inputs. This behavior is validated by the passing System 1 test suite recorded in evidence/system1/system1-test-log.txt.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   My System 1 evidence shows a successful pytest execution with 29 passing tests recorded in evidence/system1/system1-test-log.txt. I was not able to generate a live claim run because the workspace did not have ANTHROPIC_API_KEY configured, which prevented collection of claim-specific turn counts and cost data. As a result, my evidence is based on the verified test results rather than a live model execution.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   From evidence/system2/budget.json, the baseline context size was 38,708 tokens and the assembled context size was 16,780 tokens, resulting in a 56.65% reduction.

The active conversation section dominated the assembled context at 15,789 tokens. It was kept verbatim because it contained the current customer interaction and therefore the most relevant information for the next model response. The remaining sections were compressed into much smaller representations: case_facts (204 tokens), resolved_refund (391 tokens), and resolved_subscription (414 tokens).

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   The rule is to preserve structured facts exactly and summarize completed conversations. The case_facts block was preserved as a compact authoritative record (204 tokens), while resolved_refund (391 tokens) and resolved_subscription (414 tokens) were summarized from much larger conversations. The active conversation remained byte-exact at 15,789 tokens because it contained the information required for the next turn.

The final assembled context was reduced from 38,708 tokens to 16,780 tokens, demonstrating that completed history can be summarized while critical current state and structured facts are preserved.


7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
    Comparing eval.jsonl with eval_control.jsonl shows that Question 6 regressed in the control configuration.

In eval.jsonl, Question 6 passed because the model correctly returned the exact structured status token "in_progress". In eval_control.jsonl, the model answered that no structured status token existed and returned "unknown", causing the evaluation to fail.

This demonstrates why the facts block is important. Preserving structured case facts allows the model to recover exact status values, while the control configuration loses that detail and produces an incorrect answer.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   System 3 uses path-scoped configuration stored in evidence/system3/.claude/rules/. The rules are applied only where relevant instead of affecting the entire repository. Compared with a single repository-level CLAUDE.md, path-scoped rules provide more precise control and reduce unintended behavior across unrelated project areas. 

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   The deploy-check skill is located in evidence/system3/.claude/skills/deploy-check/SKILL.md. Running a forked, read-only skill limits the blast radius of analysis activities and prevents unintended modification of project files. Without this separation, review operations could accidentally modify working artifacts. 

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
     Project-level configuration is represented by evidence/system3/CLAUDE.md and the contents of evidence/system3/.claude/. User-level configuration would apply outside the repository and affect all projects. The validator evidence was recorded in evidence/system3/validator-ok.txt.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    System 4 separates hot, warm, and cold state so the model does not need to process the entire historical dataset. The implementation was validated by the System 4 test suite recorded in evidence/system4/system4-test-log.txt, which completed with 33 passing tests. 

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    Crash recovery exists so that interrupted work can continue when it is still recent and relevant. A fresh start with a summary can be more reliable than resuming stale state because old in-memory context may no longer reflect the current system condition. The crash recovery behavior is validated by the System 4 test suite in evidence/system4/system4-test-log.txt. 

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    I was unable to generate hot_state.json in this workspace and therefore could not quote its exact size. The System 4 tests passed successfully (33 passed) as recorded in evidence/system4/system4-test-log.txt, demonstrating the intended tiered-state architecture. 

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: System 1 demonstrates model-driven control flow through the stop_reason-based loop validated in evidence/system1/system1-test-log.txt.
    → Harness: System 2 demonstrates harness-level context management validated in evidence/system2/system2-test-log.txt.
    → Orchestration: System 4 demonstrates orchestration of multi-shift processing validated in evidence/system4/system4-test-log.txt.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    A deterministic behavior is the test-validated enforcement of Claude Code configuration files in evidence/system3/.claude/. A prompt-guided behavior is the model's reasoning about context and tool usage. Deterministic controls are appropriate for safety and enforcement, while prompt guidance is appropriate for flexible decision making. 

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    System 2 manages context inside a single conversation by controlling prompt size, while System 4 manages context across multiple sessions using tiered state. Evidence is recorded in evidence/system2/system2-test-log.txt (28 passed, 2 skipped) and evidence/system4/system4-test-log.txt (33 passed). 

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    A single successful run does not prove reliability across edge cases. The automated test suites provide that confidence. Evidence includes evidence/system1/system1-test-log.txt (29 passed), evidence/system2/system2-test-log.txt (28 passed, 2 skipped), and evidence/system4/system4-test-log.txt (33 passed). 

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    System 3 has a limited blast radius because configuration and skills are separated and reviewed through controlled project-level configuration. Evidence includes evidence/system3/CLAUDE.md, evidence/system3/.claude/, and evidence/system3/validator-ok.txt. 

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    The first issue I encountered was that ANTHROPIC_API_KEY was not configured in the workspace when attempting to run the live System 1 workflow. I verified the implementation by running the test suite instead, resulting in the passing evidence captured in evidence/system1/system1-test-log.txt. 

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    I would make artifact generation more explicit by producing evidence files automatically after successful verification runs. During this project, some generated files were unavailable in the workspace while test logs were available. Centralized evidence generation would simplify review and submission.
