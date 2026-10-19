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
