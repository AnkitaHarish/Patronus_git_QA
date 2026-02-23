# Task QA Guide - Spreadsheet Instructions

**Purpose**: Guide for QA reviewers validating generated tasks using the QA spreadsheet

**Prerequisites**: Read [TASKS-INTRO.md](TASKS-INTRO.md) first to understand task structure, bootstrap data, and grader types

---

## Table of Contents

1. [QA Process Overview](#qa-process-overview)
2. [Column Definitions](#column-definitions)
3. [Grader Validation Guide](#grader-validation-guide)
4. [Common Scenarios](#common-scenarios)
5. [Examples](#examples)
6. [Best Practices](#best-practices)

---

## QA Process Overview

### Your Role

You are validating automatically generated tasks to ensure they are:
1. **Executable**: Can an agent actually complete this task?
2. **Aligned**: Does the task match website features?
3. **Gradable**: Can we automatically verify success?
4. **Realistic**: Does this resemble real user behavior?

### QA Flow

```
1. Read task instruction
   ↓
2. Check if it makes sense
   ↓
3. Verify it's possible to execute
   ↓
4. Validate grader configuration
   ↓
5. Propose Missing graders (if needed)
```

## Column Definitions

### 1. Task ID

**What it is**: Unique identifier for the task

**Your action**: Copy from source data (read-only)

---

### 2. Source

**What it is**: Where this task came from

**Values**:
- `base` - Base tasks list
- `complex` - Tasks with complex problem that can't be graded (except LLM Grader)
- `subjective` - Tasks with simple problem that can't be graded (except LLM Grader)

**Your action**: Make sure the source alignes with the task

---

### 3. Instruction

**What it is**: The natural language task given to the agent

**Example**: _"Find Italian restaurants in San Francisco under $$ and show me the top 3 results"_

**Your action**: Copy full instruction from task JSON (read-only)

---

### 4. Is task valid

**What it is**: Your overall judgment after reviewing all aspects

**Values**:
- ✅ **TRUE** - Task is good as-is
- ❌ **FALSE** - Task should be discarded

**Your action**: Don't change this manually. It has defined formula.

---

## QA: Validate Task Instructions

### Column: Is it aligned with website features?

**What it is**: Does this task use features that actually exist on the website?

**How to check**:
1. Look at the task instruction
2. Identify which features are required (search, filter, reservations, reviews, etc.)
3. Cross-reference with website documentation
4. Verify all required features exist

**Values**:
- ✅ **Yes** - All features mentioned exist
- ❌ **No** - Task uses non-existent features

**Examples**:

✅ **Aligned**:
- _"Find Italian restaurants in San Francisco"_ (search feature exists)
- _"Join the waitlist for Hungry Like The Wolf for 3 people"_ (waitlist feature exists)

❌ **Not aligned**:
- _"Order delivery from Freemans"_ (if website doesn't have delivery ordering)
- _"Join a waitlist for 12pm"_ (feature exists but don't have time selection)

**Your action**: Mark ✅/❌️ and note which features are problematic

---

### Column: Does it make sense?

**What it is**: Is the instruction coherent, unambiguous, and realistic?

**How to check**:
1. Read the instruction as if you were the agent
2. Ask: "Do I understand what I need to do?"
3. Check for contradictions or impossible combinations
4. Verify it resembles real user behavior

**Values**:
- ✅ **Yes** - Clear and logical
- ❌ **No** - Confusing, contradictory, ambiguous or unrealistic

**Examples**:

✅ **Makes sense**:
- _"Find vegan-friendly spots in San Francisco for an upcoming solo birthday celebration"_
- _"Write a 4-star review for Beit Rima on Church Street mentioning their amazing falafel"_

❌ **Doesn't make sense**:
- _"Find the cheapest expensive restaurant"_ (contradictory)
- _"Show me restaurants that don't serve food"_ (nonsensical)
- _"Book a reservation for yesterday"_ (impossible)
- _"Write a 4-star review for Beit Rima mentioning their amazing falafel"_ (ambiguious - there is more than one 'Beit Rima')


**Your action**: Mark ✅/❌️ and note specific issues

---

## Validate Task Execution

### Column: Task possible to execute

**What it is**: Can an agent actually complete this task given the website capabilities?

**How to check**:
1. Consider each step required
2. Check if necessary data exists (restaurants, users, etc.)
3. Verify no external dependencies (phone calls, physical actions, recieving a text)
4. Confirm all required information is provided (via instruction or bootstrap data)

**Values**:
- ✅ **Yes** - Fully executable
- ❌ **No** - Cannot be completed

**Examples**:

✅ **Executable**:
- _"Find Hungry Like The Wolf in Houston"_ + bootstrap data with restaurant ID
- _"Log in and view your profile"_ + bootstrap data with user credentials

❌ **Not executable**:
- _"Find the restaurant where I proposed to my wife"_ (requires personal context)
- _"Call ahead to confirm the reservation"_ (requires phone call - outside browser)
- _"Find restaurants near my current location"_ (if no location provided and geolocation disabled)

**Your action**: Mark ✅/❌ and explain blocking issues

---

### Column: Steps

**What it is**: Number of actions the agent must take to finish the task

**Your action**: Set steps number

---

### Column: Gold answer possible

**What it is**: Can we determine the objectively correct answer/outcome for this task?

**Definition**:
- **Gold answer possible (Yes)**: Task has a deterministic, verifiable outcome
  - Example: _"Find Beit Rima on Church Street in San Francisco"_ → Specific restaurant (ID known)
  - Example: _"Join waitlist for 3 people"_ → Waitlist entry with partySize=3

- **No gold answer (No)**: Task is subjective (requires human judgement) or has multiple valid outcomes
  - Example: _"Find a good Italian restaurant"_ → "good" is subjective
  - Example: _"Recommend a place for a romantic dinner"_ → Multiple valid answers, "romantic" requires human judgement

**Values**:
- ✅ **Yes** - Objective outcome can be verified
- ❌ **No** - Subjective or multiple valid answers

**Why it matters**:
- **Yes** → Should have state/answer/URL graders (precise validation)
- **No** → Should rely more on LLM graders (semantic validation)

**Examples**:

✅ **Gold answer possible**:
- _"Join waitlist at Freemans for 4 people"_ → partySize must = 4, single restaurant mentioned
- _"Write a 5-star review for Beit Rima on Church Street"_ → rating must = 5, restaurant name provided
- _"Find the highest rated restaurant under $$"_ → priceRange must include $ or $$ and ther restaurant order is rank

❌ **No gold answer**:
- _"Find a romantic restaurant"_ → "romantic" is subjective
- _"Suggest a restaurant for a business lunch"_ → Multiple valid options

**Your action**: Mark ✅/❌/⚠️ and explain reasoning

**Impact on graders**:
```
✅ Gold answer possible → SHOULD have precise graders
   - State graders to verify data
   - Answer graders to check specific content
   - URL graders to verify filters

❌ No gold answer → SHOULD rely on LLM graders
   - LLM grader to evaluate semantic completion
   - Minimal/no state graders
```

---

## Validate Graders

**Important concepts**:

1. **Missing grader** = Task NEEDS this grader type but doesn't have it
   - Example: Task creates a reservation but has no state grader to verify it
   - **Action required**: Note in "Grading proposition" what's Missing

2. **No grader needed** = Task doesn't require this grader type
   - Example: Subjective task with LLM grader only
   - **Action required**: Mark "N/A - not needed"

3. **Grader present** = Task has this grader and it's configured
   - **Action required**: Validate it's correct

---

### Column: State graders

**What it is**: Graders that validate backend state changes (database, API state)

**When needed**:
- ✅ Task creates/modifies data (reservations, reviews, waitlist entries, collections)
- ✅ Task has gold answer possible (specific restaurant, specific filters)
- ✅ Task requires authentication (validates login state)

**When NOT needed**:
- ❌ Task is read-only (just browsing, searching without specific criteria)
- ❌ Task is subjective with no verifiable outcome
- ❌ Task only requires text output validation

**Your validation**:

```
IF task creates/modifies data:
   IF state_grader_configs exists:
      ✅ Mark "Valid"
      → Verify assertions match task requirements
   ELSE:
      ❌ Mark "Missing"
      → Add to "Grading proposition"
ELSE:
   ✅ Mark "No need"
```

**Values**:
- ✅ **Valid** - Has state graders and they're appropriate
- ❌ **Missing** - Needs state graders but doesn't have them
- ✅ **No need** - Task doesn't require state validation

**Examples**:

✅ **Present - Correct**:
Task: _"Join waitlist for 3 people"_
```json
{
  "state_grader_configs": [
    {
      "name": "Waitlist entry created",
      "path_to_actual": "$.waitlists[0]",
      "assertions": [
        {"operator": "NUMERIC_MATCH", "expected": ["==3"], "path_to_actual": "$.partySize"}
      ]
    }
  ]
}
```
✅ Correct - validates waitlist entry with party size 3

❌ **Missing**:
Task: _"Write a 5-star review for Beit Rima"_
```json
{
  "grader_config": {
    "llm_grader_config": {
      "instruction": "Verify review was submitted"
    }
  }
}
```
❌ Has LLM grader but NO state grader to verify review in database

✅ **N/A - not needed**:
Task: _"Find me some random restaurant and show the menu"_
```json
{
  "llm_grader_config": {
    "instruction": "Find me some random restaurant and show the menu"
  }
}
```
✅ Subjective task with no specific outcome - LLM grader is sufficient

**Your action**:
- Mark status (Valid/Missing/No need)
- If present, verify assertions match task requirements
- If Missing, note in "Grading proposition"

---

### Column: Answer grader

**What it is**: Graders that validate the agent's final text response contains expected content

**When needed**:
- ✅ Task requires specific information in the answer (restaurant names, prices, details)
- ✅ Task has gold answer possible with known entities
- ✅ Task asks agent to "show me" or "tell me" specific things

**When NOT needed**:
- ❌ Task is action-only (join waitlist, submit review) without requiring text output
- ❌ Task outcome is fully verified by state graders
- ❌ Task is subjective with no specific required content

**Your validation**:

```
IF task requires specific content in answer:
   IF answer_grader_config exists:
      ✅ Mark "Valid"
      → Verify expected values match task
   ELSE:
      ❌ Mark "Missing"
      → Add to "Grading proposition"
ELSE:
   ✅ Mark "N/A - not needed"
```

**Values**:
- ✅ **Present - Correct** - Has answer grader with correct assertions
- ❌ **Missing** - Needs answer grader but doesn't have it
- ✅ **N/A - not needed** - Task doesn't require answer validation

**Examples**:

✅ **Present - Correct**:
Task: _"Find Hungry Like The Wolf in Houston and show me the details"_
```json
{
  "answer_grader_config": {
    "assertions": [
      {
        "operator": "STRING_CONTAINS",
        "expected": ["Hungry Like The Wolf"]
      }
    ]
  }
}
```
✅ Correct - verifies restaurant name in answer

❌ **Missing**:
Task: _"Find Beit Rima in San Francisco and show me their hours"_
```json
{
  "grader_config": {
    "llm_grader_config": {
      "instruction": "Verify restaurant found"
    }
  }
}
```
❌ Should have answer grader to check "Beit Rima" is in response

✅ **N/A - not needed**:
Task: _"Join the waitlist for 3 people"_ (action-only, no answer validation needed)
```json
{
  "state_grader_configs": [...]
}
```
✅ State grader is sufficient - task doesn't require text output

**Your action**:
- Mark status (Present/Missing/N/A)
- If present, verify expected values are correct
- If Missing, note what should be checked in "Grading proposition"

---

### Column: URL grader

**What it is**: Graders that validate the final URL contains expected parameters or paths

**When needed**:
- ✅ Task requires specific filters (price, location, cuisine, rating)
- ✅ Task requires navigation to specific page (restaurant detail, profile, search results)
- ✅ Task has verifiable URL structure (query parameters, paths)

**When NOT needed**:
- ❌ Task doesn't care about final URL (e.g., "write a review" - could end on confirmation page or restaurant page)
- ❌ Task is about reading/viewing without specific navigation
- ❌ Website doesn't use URL parameters for state (SPA with client-side routing)

**Your validation**:

```
IF task requires specific URL state:
   IF url_grader_config exists:
      ✅ Mark "Valid"
      → Verify parameters match task filters
   ELSE:
      ❌ Mark "Missing"
      → Add to "Grading proposition"
ELSE:
   ✅ Mark "N/A - not needed"
```

**Values**:
- ✅ **Present - Correct** - Has URL grader with correct parameters
- ⚠️ **Present - Needs fixes** - Has URL grader but parameters are wrong
- ❌ **Missing** - Needs URL grader but doesn't have it
- ✅ **N/A - not needed** - Task doesn't require URL validation

**Examples**:

✅ **Present - Correct**:
Task: _"Find Italian restaurants in San Francisco under $$"_
```json
{
  "url_grader_config": {
    "assertions": [
      {"operator": "STRING_CONTAINS", "expected": ["cuisine=Italian"], "path_to_actual": "$.url"},
      {"operator": "STRING_CONTAINS", "expected": ["priceRange=$$"], "path_to_actual": "$.url"},
      {"operator": "STRING_CONTAINS", "expected": ["San+Francisco"], "path_to_actual": "$.url"}
    ]
  }
}
```
✅ Correct - validates all filters in URL

❌ **Missing**:
Task: _"Find vegan restaurants in Portland"_
```json
{
  "grader_config": {
    "llm_grader_config": {
      "instruction": "Verify vegan restaurants shown"
    }
  }
}
```
❌ Should verify URL contains "Portland" and vegan-related parameters

✅ **N/A - not needed**:
Task: _"Write a review for Beit Rima"_ (final URL could be confirmation page, restaurant page, or profile - doesn't matter)
```json
{
  "state_grader_configs": [...]
}
```
✅ State grader is sufficient - URL doesn't affect validation

**Your action**:
- Mark status (Present/Missing/N/A)
- If present, verify parameters match filters in task
- If Missing, note required parameters in "Grading proposition"

---

### Column: LLM grader

**What it is**: LLM-based semantic validation of task completion

**When needed** (at least one should be true):
- ✅ Task has subjective elements (quality, appropriateness, semantic understanding)
- ✅ Task requires understanding trajectory/steps taken
- ✅ Task has elements that are hard to validate with precise assertions
- ✅ Task mentions specific content that should appear (e.g., "mention the falafel")
- ✅ **Recommended**: Almost all tasks should have an LLM grader as a safety net

**When NOT needed** (rare):
- ❌ Task is 100% verifiable with state/answer/URL graders AND very simple
- ❌ Example: _"Log in"_ with state grader checking session - LLM grader adds little value

**Your validation**:

```
IF task has subjective elements OR mentions specific content:
   IF llm_grader_config exists:
      ✅ Mark "Valid"
      → Verify instruction covers all requirements
   ELSE:
      ❌ Mark "Missing"
      → Add to "Grading proposition"
ELSE:
   ✅ Mark "N/A - not needed" (rare)
```

**Values**:
- ✅ **Present - Correct** - Has LLM grader with comprehensive instruction
- ❌ **Missing** - Needs LLM grader but doesn't have it
- ✅ **N/A - not needed** - Task is simple and fully covered by other graders (rare)

**Examples**:

✅ **Present - Correct**:
Task: _"Write a 4-star review for Beit Rima mentioning their amazing falafel"_
```json
{
  "llm_grader_config": {
    "instruction": "Verify that a review with 4 stars is submitted mentioning the falafel.",
    "include_trajectory": true
  }
}
```
✅ Correct - covers star rating AND specific content (falafel)

❌ **Missing**:
Task: _"Recommend a good Italian restaurant for a business lunch"_
```json
{
  "grader_config": {
    "answer_grader_config": {
      "assertions": [
        {"operator": "STRING_CONTAINS", "expected": ["Italian"]}
      ]
    }
  }
}
```
❌ "Good" and "business lunch" are subjective - needs LLM grader

✅ **N/A - not needed**:
Task: _"Click the login button"_
```json
{
  "state_grader_configs": [
    {"assertions": [{"operator": "STRING_EQUALS", "expected": ["authenticated"], "path_to_actual": "$.status"}]}
  ]
}
```
✅ Simple task, state grader fully validates (but LLM grader still wouldn't hurt)

**Your action**:
- Mark status (Present/Missing/Recommended/N/A)
- If present, verify instruction covers all task requirements
- If Missing and subjective, note in "Grading proposition"

---

## Column: Grading proposition

**What it is**: Your recommendations for fixing or improving the grader configuration

**When to fill**:
- ❌ Any grader marked "Missing"

**Format**:
```
Add [grader type]: [what to check]
Fix [grader type]: [what to change]
```

**Examples**:

**Missing state grader**:
```
Task: "Join waitlist for 3 people at Freemans"
Current: No state grader

Proposition:
Add state grader: Verify waitlist entry created
- Check $.waitlists[0].partySize == 3
- Check $.waitlists[0].restaurantId == "freemans-id"
- Check $.waitlists[0].status == "active"
```

**Missing answer grader**:
```
Task: "Find Italian restaurants in San Francisco and show me the top 3"
Current: Has URL grader, no answer grader

Proposition:
Add answer grader: Verify restaurant names in answer
- Check answer contains at least 3 restaurant names
- Or use paths_to_expected: ["$[0].result.restaurants[0].name", "$[0].result.restaurants[1].name", "$[0].result.restaurants[2].name"]
```

**Incomplete URL grader**:
```
Task: "Find restaurants in San Francisco sorted by highest rated"
Current: URL grader only checks location

Proposition:
Fix url grader: Add sorting parameters
- Add assertion for "sort_by=rating"
- Add assertion for "sort_order=desc"
```

**Missing LLM grader**:
```
Task: "Write a positive review for Freemans mentioning their BBQ ribs"
Current: Has state grader (rating check), no LLM grader

Proposition:
Add LLM grader: Verify review content
- Instruction: "Verify that a positive review is submitted mentioning BBQ ribs"
- Set include_trajectory: true
```

**Multiple issues**:
```
Task: "Find vegan-friendly restaurants in Portland under $$ and join waitlist for 2"
Current: Only has LLM grader

Proposition:
Add state grader: Verify waitlist entry
- Check $.waitlists[0].partySize == 2
- Check $.waitlists[0].status == "active"

Add url grader: Verify search filters
- Check URL contains "Portland" or "near=Portland"
- Check URL contains "priceRange=$" or "priceRange=$$"
- Check URL contains vegan-related filter

Fix LLM grader: Make instruction more specific
- Current: "Verify task completed"
- Better: "Verify that vegan-friendly restaurants in Portland under $$ are found AND waitlist is joined for 2 people"
```

**Your action**:
- Be specific about what to add/fix
- Include expected values or parameters
- Reference JSONPaths for state graders
- Suggest operator types (STRING_CONTAINS, NUMERIC_MATCH, etc.)

---

## Common Scenarios

### Scenario 1: Search Task with Specific Criteria

**Task**: _"Find Italian restaurants in San Francisco under $$"_

**Expected graders**:
- ✅ **State grader** OR **Answer grader**: Verify restaurants match criteria
  - If gold answer possible (specific restaurants): State grader with expected_state_function
  - If more flexible: Answer grader checking for restaurant names
- ✅ **URL grader**: Verify filters in URL
  - cuisine=Italian
  - location=San+Francisco
  - priceRange=$$
- ✅ **LLM grader**: Verify overall completion (recommended but not critical)

**QA checklist**:
- [ ] URL grader checks all three filters?
- [ ] Answer/State grader verifies restaurants shown?
- [ ] LLM grader covers all criteria?

---

### Scenario 2: Data Creation Task

**Task**: _"Write a 5-star review for Beit Rima mentioning their amazing falafel"_

**Expected graders**:
- ✅ **State grader**: REQUIRED - verify review in database
  - Check $.reviews[0].rating == 5
  - Check $.reviews[0].restaurantId == "beit-rima-id"
  - Check $.reviews[0].userId matches bootstrap user
- ❌ **Answer grader**: Not needed (action-only task)
- ❌ **URL grader**: Not needed (final URL doesn't matter)
- ✅ **LLM grader**: REQUIRED - verify "falafel" mentioned
  - Instruction: "Verify that a 5-star review is submitted mentioning the falafel"

**QA checklist**:
- [ ] State grader validates review created?
- [ ] State grader checks rating = 5?
- [ ] State grader checks correct restaurant ID?
- [ ] LLM grader checks "falafel" mentioned?

---

### Scenario 3: Subjective Task

**Task**: _"Find romantic restaurants in San Francisco suitable for a date night"_

**Expected graders**:
- ❌ **State grader**: Not needed (no specific restaurants)
- ⚠️ **Answer grader**: Optional - could check "San Francisco" mentioned
- ⚠️ **URL grader**: Optional - could check location filter
- ✅ **LLM grader**: REQUIRED - only way to validate "romantic" and "suitable for date night"
  - Instruction: "Verify that romantic restaurants in San Francisco suitable for a date night are shown"

**QA checklist**:
- [ ] LLM grader covers all subjective criteria?
- [ ] No unnecessary precise graders?
- [ ] Gold answer marked as "No"?

---

### Scenario 4: Multi-Step Task

**Task**: _"Find Hungry Like The Wolf in Houston, join waitlist for 3, and show me the confirmation"_

**Expected graders**:
- ✅ **State grader**: REQUIRED - verify waitlist entry
  - Check $.waitlists[0].partySize == 3
  - Check $.waitlists[0].restaurantId == "hungry-wolf-id"
- ⚠️ **Answer grader**: Optional - could verify "Hungry Like The Wolf" and "3" in confirmation
- ❌ **URL grader**: Not critical (could end on confirmation page or waitlist page)
- ✅ **LLM grader**: REQUIRED - verify full flow completed
  - Instruction: "Verify that the restaurant 'Hungry Like The Wolf' is found in Houston, waitlist is joined for 3 people, and confirmation is displayed"
  - Set include_trajectory: true (to validate all steps)

**QA checklist**:
- [ ] State grader validates waitlist entry?
- [ ] LLM grader validates all three parts (find, join, show)?
- [ ] LLM grader has include_trajectory: true?

---

## Best Practices

### For QA Reviewers

1. **Read TASKS-INTRO.md first** - Understand grader types before starting QA

2. **Be specific in propositions** - Don't just say "add state grader", specify what to check

3. **Consider the user perspective** - Would a real user ask for this?

4. **Check bootstrap data alignment** - If task mentions "your account", is user data in bootstrap_data?

5. **Validate assertion operators** - Use NUMERIC_MATCH for numbers, STRING_CONTAINS for text

6. **Don't over-grade** - Simple tasks don't need all four grader types

7. **Don't under-grade** - Data creation tasks MUST have state graders

### Red Flags

🚩 **Task mentions specific entity but no state grader**
- Example: "Join waitlist for 3" with no state grader checking partySize=3

🚩 **Task says "show me" but no answer grader or LLM grader**
- Example: "Show me the details" with no way to validate output

🚩 **Task is subjective but only has precise graders**
- Example: "Find romantic restaurants" with only URL grader

🚩 **Task creates data but no state grader**
- Example: "Write a review" with no state grader checking review in database

🚩 **Task has zero graders total**
- Every task needs at least one grader (usually LLM grader minimum)

### Decision Tree: Grader Selection

```
Does task create/modify data? (review, reservation, waitlist, collection)
├─ YES → State grader REQUIRED
└─ NO → State grader optional or N/A

Does task require specific content in answer?
├─ YES → Answer grader REQUIRED or LLM grader REQUIRED
└─ NO → Answer grader N/A

Does task use filters or navigate to specific URL?
├─ YES → URL grader REQUIRED
└─ NO → URL grader N/A

Is task subjective or has hard-to-validate aspects?
├─ YES → LLM grader REQUIRED
└─ NO → LLM grader RECOMMENDED (as safety net)

Is task complex (3+ steps)?
├─ YES → LLM grader REQUIRED (with include_trajectory: true)
└─ NO → LLM grader OPTIONAL
```

---

## Quick Reference

### Grader Checklist by Task Type

**Search & Filter Tasks**:
- [ ] URL grader for filter parameters
- [ ] Answer grader OR State grader for results
- [ ] LLM grader (recommended)

**Data Creation Tasks**:
- [ ] State grader (REQUIRED) for created entity
- [ ] LLM grader (REQUIRED) for content quality
- [ ] Answer grader (optional)
- [ ] URL grader (usually N/A)

**Subjective Tasks**:
- [ ] LLM grader (REQUIRED)
- [ ] Answer/URL grader (optional)
- [ ] State grader (usually N/A)

**Authentication Tasks**:
- [ ] State grader for login state
- [ ] LLM grader (optional)

**Multi-Step Tasks**:
- [ ] State grader for final state
- [ ] LLM grader with include_trajectory: true (REQUIRED)
- [ ] URL grader (if applicable)
- [ ] Answer grader (if applicable)

---

## Glossary

- **Gold answer possible**: Task has objectively verifiable correct outcome
- **Bootstrap data**: Pre-existing data (credentials, IDs) provided to agent
- **State grader**: Validates backend database/API state
- **Answer grader**: Validates agent's text output
- **URL grader**: Validates final URL parameters
- **LLM grader**: Semantic validation using LLM judgment
- **Assertion**: Single validation check (e.g., "partySize == 3")
- **JSONPath**: Query language for navigating JSON (e.g., "$.waitlists[0].status")
- **Operator**: Comparison type (STRING_CONTAINS, NUMERIC_MATCH, etc.)

---

## Getting Help

If you encounter:
- **Unclear task instruction** → Mark "Does it make sense?" as ⚠️ and explain ambiguity
- **Unsure about graders** → Refer to [TASKS-INTRO.md](TASKS-INTRO.md) grader type section
- **Complex edge case** → Escalate to engineering team with detailed notes

Remember: Your QA is critical for ensuring high-quality evaluation data. When in doubt, be thorough and specific in your notes!
