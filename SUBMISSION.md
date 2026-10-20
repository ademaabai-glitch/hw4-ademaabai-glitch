HW4 — submission
AI assistance:Claude
## Part A: Plan on paper

### A1. Steps

I split the task into 5 steps. Each step has one job. No two steps do the same job. Together they cover everything from the question to the answer. The loop is not a step of the task. It only carries messages between the model and the tools.

| # | Step | Input → Output | If it goes wrong | Assumptions | Who |
|---|------|----------------|------------------|-------------|-----|
| 1 | Plan | `question: str` → list of needed facts `[(district: str, topic: str)]` and a formula in words | A name is misspelled → step 2 returns suggestions and the model tries again. The question is unclear → the model says it cannot answer. | Names in the question may have typos. Two names look alike (Ossbridge / Ossford). | Model |
| 2 | Find the fact | `district: str, topic: str` → `dict` with `doc_id: str`, `snippets: list[str]`, `notes: list[str]` | Nothing found → `{"error": ...}` with "did you mean". Unknown topic → error with the list of allowed topics. Two similar names → an exact match wins, otherwise an error with suggestions. | Use only the 2025 profiles. Archived profiles (NR-13, NR-14) are never returned. A correction notice (NR-16) is added to `notes` and it beats the profile. | Tool `lookup` |
| 3 | Read the numbers | `snippets, notes` → `float` values with units | A number cannot be read → the model calls `lookup` again with another topic. Unit unknown → the model says so and does not guess. | Notes override snippets. Convert units before comparing: Lindenmoor gives water in m³, the others in megalitres (1 ML = 1,000 m³). | Model |
| 4 | Do the maths | `expression: str` → `float` | Division by zero, bad characters, or a number that is too big → `{"error": ...}`. The model fixes the expression. | Keep full precision. Do not round in the middle. | Tool `calculate` |
| 5 | Write the answer | numbers → `ANSWER: <name or yes/no> \| <number>` | A fact is missing → say it cannot answer. | Round only here, to the number of decimals the question needs. | Model |

**Loop (glue code):** it reads the model's request, checks that the JSON arguments are valid, runs the right tool, sends the result back with the same `tool_call_id`, and stops after 8 rounds (then the question is marked "unanswered"). I chose 8 because I expect the hardest questions to need about 3 to 4 rounds, so 8 leaves room for retries but still stops endless loops.

**How the steps connect:**

```
question
  → (1) Plan [model]
  → (2) lookup [tool]  → snippets + notes
  → (3) Read numbers [model]
  → (4) calculate [tool] → result
  → (5) Write answer [model] → "ANSWER: name | number"
(steps 2 to 4 can repeat, up to 8 rounds)
```

**The assumption my solution depends on most:** `lookup` always returns the correction notice together with the profile. Q-09 depends on it. Without the notice, Brightwater's area is 912.4 and the difference is about 0.08, not 0.6, so the answer would be wrong.

---

### A2. Tools

**Tool 1: lookup**

```json
{
  "type": "function",
  "function": {
    "name": "lookup",
    "description": "Find one fact about one district in the Nine Rivers documents. Give the district name and a topic. Returns short text from the current 2025 profile, with the document id. Archived profiles are never used. If a correction notice exists, it is returned in 'notes' and it is more correct than the profile. For the water price use district 'Region' and topic 'water_tariff'. Read the unit in the text carefully, units can differ between districts.",
    "parameters": {
      "type": "object",
      "properties": {
        "district": {
          "type": "string",
          "description": "District name, for example 'Brightwater'. Use 'Region' for the water tariff."
        },
        "topic": {
          "type": "string",
          "enum": ["population", "area", "budget", "water_use", "schools", "pupils", "founded", "roads", "water_tariff"],
          "description": "The kind of fact you need."
        }
      },
      "required": ["district", "topic"]
    }
  }
}
```

| Case | Returns |
|------|---------|
| Works | `{"district": "Brightwater", "doc_id": "NR-05", "snippets": ["Census population: 39,902 in 2020 and 44,675 in 2025."], "notes": []}` |
| Works, with a correction | Same shape, with `"notes": ["NR-16: the correct area is 921.4 square kilometres, not 912.4. Use 921.4."]` |
| Misspelled name | `{"error": "no district named Ossbrige; did you mean Ossbridge or Ossford?"}` |
| Unknown topic | `{"error": "unknown topic 'x'; use one of: population, area, ..."}` |
| Only an archived profile exists | Cannot happen with this data (every district has a 2025 profile). Archived profiles are skipped anyway. |
| Worst case with strange or dangerous input | The inputs are only used to match text. They are never run as code and never used as a file path. Strings are cut to 50 characters and the output is cut to a safe length. Worst result: an error message. |

**Tool 2: calculate**

```json
{
  "type": "function",
  "function": {
    "name": "calculate",
    "description": "Do arithmetic. Input is one expression with numbers, + - * / ** and brackets, for example '44675 / 921.4'. Use it for every calculation. Do not do maths in your head. Do not write words or function names, only numbers and operators.",
    "parameters": {
      "type": "object",
      "properties": {
        "expression": {
          "type": "string",
          "description": "The arithmetic expression."
        }
      },
      "required": ["expression"]
    }
  }
}
```

| Case | Returns |
|------|---------|
| Works | `{"result": 48.486}` |
| Division by zero | `{"error": "division by zero"}` |
| Not allowed (names, calls, attributes like `.__class__`) | `{"error": "not allowed: ..."}` |
| Too long, or number too big (like `9**9**9`) | `{"error": "number too large"}` |
| Worst case with strange or dangerous input | Nothing is run. The expression is parsed with `ast` and only numbers and chosen operators pass. The power operator has a limit, so it cannot freeze the program. |

---

### A3. Q-09 by hand

Question: *Which district has the higher population density in 2025, Brightwater or Millbrook, and by how many residents per square kilometre?*

```json
[
  {"role": "user", "content": "Which district has the higher population density in 2025, Brightwater or Millbrook, and by how many residents per square kilometre?"},

  {"role": "assistant", "tool_calls": [
    {"id": "call_1", "type": "function", "function": {"name": "lookup", "arguments": "{\"district\": \"Brightwater\", \"topic\": \"population\"}"}},
    {"id": "call_2", "type": "function", "function": {"name": "lookup", "arguments": "{\"district\": \"Brightwater\", \"topic\": \"area\"}"}},
    {"id": "call_3", "type": "function", "function": {"name": "lookup", "arguments": "{\"district\": \"Millbrook\", \"topic\": \"population\"}"}},
    {"id": "call_4", "type": "function", "function": {"name": "lookup", "arguments": "{\"district\": \"Millbrook\", \"topic\": \"area\"}"}}
  ]},

  {"role": "tool", "tool_call_id": "call_1", "content": "{\"district\": \"Brightwater\", \"doc_id\": \"NR-05\", \"snippets\": [\"Census population: 39,902 in 2020 and 44,675 in 2025.\"], \"notes\": []}"},
  {"role": "tool", "tool_call_id": "call_2", "content": "{\"district\": \"Brightwater\", \"doc_id\": \"NR-05\", \"snippets\": [\"The district, founded in 1889, stretches along the Bright over 912.4 square kilometres and keeps 455.0 km of paved road.\"], \"notes\": [\"NR-16 correction: the correct area is 921.4 square kilometres, not 912.4. Use 921.4.\"]}"},
  {"role": "tool", "tool_call_id": "call_3", "content": "{\"district\": \"Millbrook\", \"doc_id\": \"NR-11\", \"snippets\": [\"Census population: 34,590 in 2020 and 37,203 in 2025.\"], \"notes\": []}"},
  {"role": "tool", "tool_call_id": "call_4", "content": "{\"district\": \"Millbrook\", \"doc_id\": \"NR-11\", \"snippets\": [\"The district, founded in 1897, stretches along the Mill Brook over 758.6 square kilometres and keeps 360.1 km of paved road.\"], \"notes\": []}"},

  {"role": "assistant", "tool_calls": [
    {"id": "call_5", "type": "function", "function": {"name": "calculate", "arguments": "{\"expression\": \"44675 / 921.4\"}"}},
    {"id": "call_6", "type": "function", "function": {"name": "calculate", "arguments": "{\"expression\": \"37203 / 758.6\"}"}}
  ]},
  {"role": "tool", "tool_call_id": "call_5", "content": "{\"result\": 48.486}"},
  {"role": "tool", "tool_call_id": "call_6", "content": "{\"result\": 49.04166}"},

  {"role": "assistant", "tool_calls": [
    {"id": "call_7", "type": "function", "function": {"name": "calculate", "arguments": "{\"expression\": \"49.04166 - 48.486\"}"}}
  ]},
  {"role": "tool", "tool_call_id": "call_7", "content": "{\"result\": 0.55566}"},

  {"role": "assistant", "content": "Brightwater: 44,675 / 921.4 = 48.486 per km2 (corrected area from NR-16). Millbrook: 37,203 / 758.6 = 49.042 per km2. Millbrook is denser by 0.556, about 0.6.\nANSWER: Millbrook | 0.6"}
]
```

**Arithmetic by hand:**

- Brightwater: 44,675 ÷ 921.4 = 48.486 residents per km²
- Millbrook: 37,203 ÷ 758.6 = 49.042 residents per km²
- Difference: 49.042 − 48.486 = 0.556, which rounds to **0.6**. Millbrook is denser.

**Why NR-16 matters:** with the wrong area 912.4, Brightwater would be 44,675 ÷ 912.4 = 48.964, and the difference would be only 0.077. So the correction notice changes the answer.

---

### A4. Another way to split

**Alternative I rejected:** one tool for each calculation (`density`, `growth_rate`, `water_per_person`, `budget_per_pupil`, and so on).

The 20 questions use about a dozen different formulas, so this would need about a dozen tools. Every tool definition is sent to the model in every request, so S3 would cost more tokens on every call. The model would also have to pick the right tool from many, and a wrong pick is a new kind of mistake. With one general `calculate`, the model writes the formula itself and the tool only does the arithmetic. A new type of question needs no new tool. A single calculator is also simple to test and to make safe.

I also rejected one big "answer the question" tool that searches and calculates inside. Mistakes would be hidden inside it, I could not test the parts alone, and I could not see in the log which step failed.

## Part B: Build and run

(If the scoring rule, pytest output and results tables are already in SUBMISSION.md from the notebook cells, do not paste them twice. Only add the parts that are missing.)

### B1. Tools

Code: `tools.py`.
- `lookup(district, topic)` reads `data/corpus.jsonl` once. It uses only the 2025 profiles, so archived profiles (NR-13, NR-14) are never returned. It adds the correction notice (NR-16) in `notes` for Brightwater. For the water price it uses district "Region" and topic "water_tariff" (NR-15). It returns at most 3 short sentences, each with the `doc_id`. Errors come back as `{"error": ...}`, for example a "did you mean" hint for a misspelled name. It never crashes.
- `calculate(expression)` uses `ast`, not `eval` or `exec`. It allows only numbers and `+ - * / **` with brackets. Limits: length 200, exponent 100, result 1e18. Division by zero and everything else return an error message.

### B2. Tests

Code: `tests/test_tools.py`. No API key needed. 11 test functions for `lookup` and 7 for `calculate` (some run with several inputs). They include: a correct 2025 number, a misspelled name, Ossbridge against Ossford, archived profiles ignored, `48317 * 1.0947`, and the three dangerous inputs (`__import__('os').system('echo hi')`, `(1).__class__`, `9**9**9`).

pytest output (from `outputs/pytest_output.txt`):

```
[PASTE the output of outputs/pytest_output.txt here]
```

### B3. The loop

Code: `strategies.py`, function `run_s3`. For every model reply the loop:
1. reads each tool request and runs `json.loads` on the arguments; invalid JSON gives an error result;
2. checks the tool name and the argument names (`run_tool`) before running anything;
3. adds the model message with its `tool_calls` unchanged, then sends one result for every `tool_call_id`;
4. stops after **8 rounds** and then marks the question "unanswered";
5. saves a log per question in `outputs/logs/` (round, id, tool, arguments, result, seconds).

Why 8: the hand plan needed 3 rounds. In my runs the most rounds were 6 (run 1) and 5 (run 2), average 3.75 and 3.5. 8 leaves room for retries and still stops endless loops. The limit was never reached.

**Q-09, real conversation against my plan in A3.**
- Same as planned: 4 lookups in the first round, 2 calculations in the second round, correct answer `Millbrook | 0.6`, and the NR-16 note arrived (area 921.4).
- Different 1: the model did not make the third `calculate` call. It subtracted 49.04 − 48.49 itself and wrote 0.6. So it used 3 rounds (inputs 523, 951, 1044 tokens).
- Different 2: `calculate` returns many decimals (48.4859995659), not 48.486.
- Different 3: the NR-16 note comes back with every Brightwater lookup, also for "population", so it costs extra tokens.
- Plan changes while coding (Part A is not edited): I added `reasoning_effort="none"` because `gpt-5.6-luna` refuses function tools in Chat Completions when reasoning is on. I used it in all three strategies so they stay equal.

### B4. The three strategies

Code: `strategies.py` (S1, S2, S3), `run.py` (runs and tables). Results and logs are in `outputs/` (`run1_*.jsonl`, `run2_*.jsonl`, `report.md`).

**Fixed for all strategies:** model `gpt-5.6-luna`; the same 20 questions; the same RULES text and the same last line `ANSWER: <name or yes/no> | <number>`; the `unit` and `decimals` of each question are in the prompt (format only, never the answer); `reasoning_effort="none"`; `temperature=1.0` [CHECK: if `strategies.SETTINGS["temperature_used"]` is False, write that the model does not accept temperature]; S2 uses k = 5 and takes the most common answer; S3 has no documents and a limit of 8 rounds. Tokens are counted from the `usage` field of every call (all 5 samples in S2, every round in S3).

**Rule for a correct answer** (written and committed before the first full run): the name must match (ignore case), and the number must be within 1 in the last digit of `value` rounded to `decimals`. No readable `ANSWER:` line, or hitting the round limit, counts as unanswered and wrong.

**Why 20 questions and this rule:** S2 sends all documents 5 times per question, so 20 questions is what I could pay for twice. The rule allows 1 in the last digit so that normal rounding of a long number is not punished, but it is a little generous (see C4).

**Results** (copy from `outputs/report.md`):

| Run | Strategy | Correct /20 | Unanswered | Input tokens | Output tokens | Median s/question | Total s | Cost ($) |
|---|---|---|---|---|---|---|---|---|
| 1 | S1 | 5 | 5 | 44746 | 668 | 1.4 | 30 | [FILL] |
| 1 | S2 | 13 | 0 | 224530 | 7465 | 8.2 | 168 | [FILL] |
| 1 | S3 | 19 | 0 | 61582 | 4092 | 5.3 | 106 | [FILL] |
| 2 | S1 | 3 | 3 | 44746 | 643 | 1.5 | 35 | [FILL] |
| 2 | S2 | 15 | 0 | 224530 | 7294 | 8.1 | 163 | [FILL] |
| 2 | S3 | 16 | 0 | 56468 | 3974 | 5.1 | 108 | [FILL] |

Table of all 20 questions (✓/✗ for each run and strategy): copy it from `outputs/report.md`.

**Cost of the full run:** [FILL: total dollars for both runs, from the table and from the OpenAI usage page].


## Part C: Explain and choose

### C1. Does the tool description matter?

I ran S3 again on Q-03, Q-08, Q-09, Q-13 and Q-17. I changed only the first line of the `lookup` description to "Looks things up." Everything else stayed the same, including the parameter descriptions and the list of allowed topics.

- Correct answers: [FILL: x]/5 with the vague description. With the good description I had [FILL: x]/5 in run 1 and [FILL: x]/5 in run 2.
- Number of lookup calls: [FILL: vague] vs [FILL: good run 1] and [FILL: good run 2].
- Inputs: [FILL: did the model still use district "Region" and topic "water_tariff" for Q-08 and Q-17? Did it still ask for population and area for Q-09?]
- Why: [FILL: one or two sentences. Idea: the description tells the model when to use the tool, what comes back, and that notes and units matter. The topic list in the parameters still helps even if the main description is vague.]

Run 1 and run 2 also differ from each other, so a small change here may be only noise.

### C2. Three mistakes

**S1: Q-09, run 1.** S1 answered `Brightwater | 435.8`. The correct answer is `Millbrook | 0.6`. Both densities are about 49 residents per km², so a difference of 435.8 is impossible. The model did the division in its head, with no calculator and no steps, and one wrong guess became the final answer. S1 got only 5/20 and 3/20 correct. In 5 questions (run 1) and 3 questions (run 2) it gave no readable `ANSWER:` line at all. [CHECK: paste the model's reply for Q-09 from outputs/run1_S1.jsonl, and look at section 2 of outputs/analysis_output.txt to see why those answers were missing.]

**S2: Q-13, both runs.** S2 answered `Stonecairn | 3.53` in run 1 and again `Stonecairn | 3.53` in run 2. The correct answer is `Coldharbour | 4.16`. Voting helps when mistakes are random, because the wrong answers are spread out and the right one wins. Here the wrong answer won the vote in both runs, so the model made the same mistake again and again. [CHECK: look at section 3 of outputs/analysis_output.txt, "agreement of the 5 samples" for Q-13, and write how many of the 5 samples said Stonecairn.] Q-05 shows the other case: S2 gave 14373 and 14280, and the correct value is 14235. These are different near-misses from long divisions done in text, so there is no clear majority. Voting cannot fix an error the model makes every time.

**S3: Q-14, both runs.** S3 answered `Ashgrove | 1771.3`. The correct answer is `Ashgrove | 4.9`. The log shows the model called `calculate` with `5207 * 1000000 / 71388` and got `72939.4295960105`. This is litres per resident per year. It never divided by 365, but the final text says "litres per resident per day". The calculator was correct. The model chose the wrong formula. A calculator removes arithmetic errors, because `ast` computes the exact result of the expression. It cannot check whether the expression answers the question.

**Why S3 needed many rounds.** Q-14 used 6 rounds in run 1. Round 1: two lookups for water use. Round 2: two lookups for population. Rounds 3, 4 and 5: three calculations, one after another, because each used the result of the one before. Round 6: the final answer. Every round sends the whole conversation again, so input tokens grow: 530, 696, 885, 933, 981, 1034 per round.

### C3. Which strategy would I use?

I would use **S3 (tools)**.

| | S1 | S2 | S3 |
|---|---|---|---|
| Correct (run 1 / run 2) | 5 / 3 | 13 / 15 | 19 / 16 |
| Input tokens (run 1) | 44,746 | 224,530 | 61,582 |
| Median seconds per question | 1.4 | 8.2 | 5.3 |

S3 was the most accurate. It uses about 1.4 times the input tokens of S1, and S2 uses 5 times. S3 is also faster than S2. S3 is better than S2 because the calculator removes arithmetic errors, which voting cannot do, and S3 does not send the documents 5 times.

- **Why not S1:** it is the fastest and cheapest, but it got only 3 to 5 out of 20. It does the maths in its head and has no way to check.
- **Why not S2:** it costs 5 times the input tokens of S1 and is slower than S3, but it was less accurate. It repeated the same mistake on Q-05, Q-11, Q-13, Q-17 and Q-20 in both runs.

What would change my choice:
1. **Questions with no arithmetic and few documents** (like Q-10, where S1 was right in both runs): I would pick S1. It is cheap and fast.
2. **1,000 times more documents:** S1 and S2 could not put them all in the prompt. I would pick S3 with a better search tool.
3. **Strict time limit or very small budget:** I would pick S1, and accept lower accuracy.

### C4. How sure can I be?

**How different were my two runs?** S1: 5 then 3. S2: 13 then 15. S3: 19 then 16. So one strategy changed by 2 to 3 questions between runs. S1 got 4 questions different, S2 got 2, and S3 got 3.

**Is the gap between S3 and S2 bigger than that?** Not clearly. The gap was 6 in run 1 and only 1 in run 2. Over both runs, S3 was right and S2 wrong on 9 question-runs, and S2 was right and S3 wrong on 2. A sign test gives p = 0.065, which is not below 0.05. This is also too optimistic, because the same 20 questions were used twice. So I cannot say S3 is better than S2 with confidence. S1 is clearly worse than both.

**How many questions would I need?** About 82 questions to separate S3 (87.5%) from S2 (70%) with 80% power. For S1 against S2 about 12 are enough.

**Other limits.**
- My rule allows a difference of 1 in the last digit. Some S2 and S1 answers were counted right only because of this rule: S2 gave 0.5 for Q-09 (correct is 0.6) and 18.49 for Q-07 (correct is 18.50).
- I used `reasoning_effort="none"` for all three strategies, because the model refuses tools otherwise. S1 and S2 might do better with reasoning on.
- I tested one model, with two runs.

---

## Final questions

**1. When the model sends a `tool_calls` message, what has actually run so far?**
Nothing. The message is only text that asks for a function and gives the arguments. My loop reads it, checks it, and runs the tool, and then sends the result back with the same `tool_call_id`.

**2. Why does `arguments` arrive as a string, and what goes wrong if you pass it straight into your function?**
The model writes text, so the arguments come as a JSON string. If I passed the string into the function, it would get a `str` and not the inputs it needs, and it would crash. The JSON can also be broken. I use `json.loads` first, then check the names of the arguments, and return an error message to the model if something is wrong. [FILL: number of tool calls that returned an error in my runs, from outputs/analysis_output.txt]

**3. How many tokens do your two tool definitions add to every S3 request?**
348 tokens. A request with the tools had 357 prompt tokens, and the same request without them had 9. In Q-14 (run 1) there were 6 requests, so the definitions alone added about 2,088 of the 5,059 input tokens.

**4. How much more did S2 cost than S1?**
S2 used about 5.0 times the input tokens (224,530 vs 44,746 in run 1) and about 11 times the output tokens (7,465 vs 668). In dollars S2 cost [FILL: S2 cost / S1 cost from the table] times more. The reason is that S2 sends all the documents 5 times for each question and writes its steps in the answer.

**5. Which question needed the most rounds in S3, and why?**
Q-14 needed 6 rounds in run 1 and 5 in run 2. The model first asked for water use, then for population, and then used three calculations one after another, because each used the result of the one before. No question reached my limit of 8 rounds.

**6. What would you change in `lookup` if there were 100,000 documents?**
I would not scan by name any more. I would build an index (keyword search like BM25, or embeddings) and return only the best few short pieces, each with its document id. I would also keep a date and an "archived" flag for every document, so old profiles are filtered out and correction notices are linked to the document they correct.

**7. What stops your `calculate` from running dangerous code? Could the model still make it misbehave?**
I do not use `eval` or `exec`. I parse the text with `ast` and allow only numbers, `+ - * / **` and brackets. Names, calls and attributes are refused, so `__import__('os').system('echo hi')`, `(1).__class__` and `9**9**9` all return errors. I also limit the length (200), the exponent (100) and the size of the result (1e18). The model can still use a valid but wrong formula, like in Q-14 where it forgot to divide by 365.

**8. For which kind of question did tools help most, and for which did they not help at all?**
They helped most on long calculations with several steps. S3 was right in both runs on Q-05, Q-13 and Q-20, and S2 was wrong on all three in both runs. They did not help when the formula was wrong (Q-14 was wrong in both S3 runs). They were also not needed for easy questions like Q-10, where S1 was right in both runs.
