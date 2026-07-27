# BugHound Mini Model Card (Reflection)

Fill this out after you run BugHound in **both** modes (Heuristic and Gemini).

---

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Bughound was created as a way to spot and find bugs. Use built in debuggers can be overwhelming, Bughhound integrates AI to help you debug your code and offers fixes. It also rates the fix using a risk score and then states whether it can auto implement or if it should be reviewed by a human. 

**Intended users:** Students learning about agentic workflows and learning about how to safely and reliably utlizing the fixes suggested by AI.

---

## 2) How does it work?

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

**Rule 1: Removed return statements**
    - What does the rule check?
        - It checks whether the string "return" appeared in the original code but disappeared entirely from the fix.
    - Why might that check matter for safety or correctness?
        - This is an important check because it would cause the snippet no matter what to return None and that could cause issues that could have been an easy fix.
    - What is a false positive this rule could cause?
        -  A legitimate fix that renames/restructures code so the word "return" literally doesn't appear as a substring in that exact form (e.g., splits return x across lines, or the fix simplifies logic and only needs one return where before there were two are the same count so no issue).
    - What is a false negative this rule could miss?
        - A fix that keeps the word return somewhere but changes what specific value or branch actually returns it, the check would pass right through since "return" is still present, even though behavior changed

**Rule 2: Syntax validity**
    - What does the rule check?
        - The rule checks if fixed_code parses as valid Python at all.

    - Why might that check matter for safety or correctness?
        - This is the minimum bar for correctness and auto-applying code that doesn't even parse would break the codebase immediately.

    - What is a false positive this rule could cause?
        -  None really in terms of safe code flagged risky, it could reject something that's valid Python just written in an different style the assessor doesn't understand semantically.

    - What is a false negative this rule could miss?
        - The ast.parse function says nothing about semantics, so plenty of bad fixes still pass through this rule.
---

## 5) Observed failure modes


1. A time BugHound missed an issue it should have caught
    - BugHound's  regex only matches a literal `except`, so it doesn't flag `except Exception: pass` even though this has the exact same effect as a bare except: since it ignores the error and returns `None` instead of the actual result.
        - Code snippet:
        ```
        def divide(a, b):
            try:
                return a / b
            except Exception:
                pass
        ```
2. A time BugHound suggested a fix that felt risky, wrong, or unnecessary
    - I wrote a broken function meant to return the last value in an array, and BugHound proposed this "fix":
        ```
        def find_last(arr):
            if not arr:
                return None
            for i in range(len(arr)):
                if i == len(arr) - 1:
                    return i
        ```
    - This fix is riskier than it looks: it still returns `i` (the index), not `arr[i]` (the value) — the exact same bug as the original code, just wrapped in a more complicated loop. It's also far more convoluted than the idiomatic `return arr[-1]`. This is a good example of why fixes shouldn't be auto-applied just because they're syntactically valid and "look like" a fix — this one didn't actually fix anything, and a less careful review could easily approve it.
---

## 6) Heuristic vs Gemini comparison

Compare behavior across the two modes:

- **What did Gemini detect that heuristics did not?** On `flaky_try_except.py`, Gemini also flagged that the file opened with `open()` was never explicitly closed and suggested a `with` statement — a semantic/best-practice issue no regex rule checks for.
- **What did heuristics catch consistently?** The three preset triggers (`print(`, bare `except:`, `TODO`) fired reliably on every snippet that contained them, regardless of surrounding context.
- **How did the proposed fixes differ?** The heuristic fixer does blind text substitution (swap `print(` → `logging.info(`, bare `except:` → `except Exception as e:`), while Gemini rewrote the function more holistically (e.g. converting to a `with` block) and explained its reasoning.
- **Did the risk scorer agree with your intuition?** Largely yes — the bare-except fix on `flaky_try_except.py` scored as high severity/risk as expected, matching how risky I'd judge a swallowed exception to be.

---

## 7) Human-in-the-loop decision

Describe one scenario where BugHound should **refuse** to auto-fix and require human review.

- **What trigger would you add?** If the LLM's proposed fix doesn't parse as valid Python at all, BugHound should never auto-apply it — that's exactly the `ast.parse` syntax-validity trigger added in section 8.
- **Where would you implement it?** In `risk_assessor.py`, forcing `should_autofix: False` and a high risk score whenever the fix fails to parse, so it's caught regardless of how the invalid code got there.
- **What message should the tool show the user?** Something like: "BugHound could not verify this fix is valid Python — it will not be auto-applied. Please review the suggested change manually."

---

## 8) Improvement idea


**My idea:** Add a syntax-validity guardrail on the proposed fix. Right now the fixer only checks whether the LLM's output is non-empty (`bughound_agent.py`) before accepting it — a fix that isn't valid Python at all (truncated output, stray prose left in after stripping code fences, etc.) could still get passed through and even auto-applied if the risk score happened to come out low. I added an `ast.parse()` check in two places: in `propose_fix()`, so a syntax error falls back to the heuristic fixer the same way an empty response does, and again in `assess_risk()`, so that even if invalid code reached the risk assessor some other way, it immediately scores as high risk with `should_autofix: False`. This is a small, targeted addition (a few lines, no new dependencies) that closes a real gap — "the fix parses as Python" is a minimum bar any correct fix must clear, and checking for it costs almost nothing while preventing a class of clearly broken auto-fixes.
