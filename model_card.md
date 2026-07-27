# BugHound Mini Model Card (Reflection)

Fill this out after you run BugHound in **both** modes (Heuristic and Gemini).

---

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Bughound was created as a way to spot and find bugs. Use built in debuggers can be overwhelming, Bughhound integrates AI to help you debug your code and offers fixes. It also rates the fix using a risk score and then states whether it can auto implement or if it should be reviewed by a human. 

**Intended users:** Students learning about agentic workflows and learning about how to safely and reliably utlizing the fixes suggested by AI.

---

## 2) How does it work?

Describe the workflow in your own words (plan → analyze → act → test → reflect).  
Include what is done by heuristics vs what is done by Gemini (if enabled).

**Plan**
    - Runs a quick scan, not much done here

**Analyze**
    - Uses the analyze function which checks if we can use llm to parse
    - Based on the mode selection follows either path:
        - Heuristic mode: regex/string checks for specific preset topics

        - Gemini mode: loads analyzer_system.txt/analyzer_user.txt prompts, sends the code to the LLM, parses the JSON array response, and falls back to heuristics if the call errors out or the output isn't parseable JSON.

**Act**
    - Calls the propose_fix function and branches based on mode selection:
        - Heuristic mode: Uses the regex substitutions for errors

        - Gemini mode: Loads fixer prompts, sends code and issues, strips code fences from the response, and (with our recent change) validates the result with ast.parse and falls back to the heuristic fixer on empty output, API errors, or invalid syntax.

**Test**
    - Uses assess_risk function and scores the suggested fix based on issue severities, structural changes (shorter code, removed returns, modified bare excepts), and (with our change) an ast.parse syntax check that immediately zeroes the score if the fix isn't valid Python.

**Reflect**
    - Based on risk["should_autofix"] (true only when risk level is "low"), logs whether the fix is safe to auto apply or needs human review



---

## 3) Inputs and outputs

**Inputs:**

- I used all four sample snippets provided in `sample_code/`: `cleanish.py`, `flaky_try_except.py`, `mixed_issues.py`, and `print_spam.py`.
- Each was a short, one function Python code snippet (5-10 lines).
- The issues varied: `cleanish.py` is a "control" case with no issues (already uses `logging`, no bare excepts or TODOs); `flaky_try_except.py` is a file I/O function with a bare `except:` that swallows errors; `mixed_issues.py` combines a TODO comment, a `print()` call, and a bare `except:` around a division (also a latent `ZeroDivisionError`/type risk); and `print_spam.py` is a function with multiple `print()` calls and no error handling at all.
- Together they cover the three heuristic triggers (`print(`, bare `except:`, `TODO`) both in isolation and combined in one snippet, plus a clean baseline to check for false positives.

**Outputs:**

- What types of issues were detected?
    -  It detected if we were outputting the wrong variable (print_spam.py). It detected a bare except and flagged it and labeled it accordingly as high risk. The proposed fixed it gave for this was correct and it gave the reasoning for it: 
        - Reasoning: "
            1. reliability | High

            Bare except clause catches all exceptions, including system exits and keyboard interrupts, which can mask unexpected errors.

            2. maintainability | Medium

            File opened with open() is not explicitly closed, relying on garbage collection rather than a context manager (with statement)."
        - Code:
            ```
            def load_data(path):
            try:
                with open(path) as f:
                    data = f.read()
            except Exception:
                return None
            return data
            ```


---

## 4) Reliability and safety rules

List at least **two** reliability rules currently used in `assess_risk`. For each:
**Rule 1: Removed return statements**
    - What does the rule check?
        - It checks whether the string "return" appeared in the original code but disappeared entirely from the fix.
    - Why might that check matter for safety or correctness?
    - What is a false positive this rule could cause?
    - What is a false negative this rule could miss?

**Rule 2**
    - What does the rule check?
        - The rule checks 
    - Why might that check matter for safety or correctness?
    - What is a false positive this rule could cause?
    - What is a false negative this rule could miss?

---

## 5) Observed failure modes

Provide at least **two** examples:

1. A time BugHound missed an issue it should have caught  
2. A time BugHound suggested a fix that felt risky, wrong, or unnecessary  

For each, include the snippet (or describe it) and what went wrong.

---

## 6) Heuristic vs Gemini comparison

Compare behavior across the two modes:

- What did Gemini detect that heuristics did not?
- What did heuristics catch consistently?
- How did the proposed fixes differ?
- Did the risk scorer agree with your intuition?

---

## 7) Human-in-the-loop decision

Describe one scenario where BugHound should **refuse** to auto-fix and require human review.

- What trigger would you add?
- Where would you implement it (risk_assessor vs agent workflow vs UI)?
- What message should the tool show the user?

---

## 8) Improvement idea

Propose one improvement that would make BugHound more reliable *without* making it dramatically more complex.

Examples:

- A better output format and parsing strategy
- A new guardrail rule + test
- A more careful “minimal diff” policy
- Better detection of changes that alter behavior

Write your idea clearly and briefly.

**My idea:** Add a syntax-validity guardrail on the proposed fix. Right now the fixer only checks whether the LLM's output is non-empty (`bughound_agent.py`) before accepting it — a fix that isn't valid Python at all (truncated output, stray prose left in after stripping code fences, etc.) could still get passed through and even auto-applied if the risk score happened to come out low. I added an `ast.parse()` check in two places: in `propose_fix()`, so a syntax error falls back to the heuristic fixer the same way an empty response does, and again in `assess_risk()`, so that even if invalid code reached the risk assessor some other way, it immediately scores as high risk with `should_autofix: False`. This is a small, targeted addition (a few lines, no new dependencies) that closes a real gap — "the fix parses as Python" is a minimum bar any correct fix must clear, and checking for it costs almost nothing while preventing a class of clearly broken auto-fixes.
