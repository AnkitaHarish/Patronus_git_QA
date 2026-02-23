# Task and Grader System - Technical Overview

**Purpose**: Onboarding guide for working with tasks and graders in the Browser Use Gym

**Audience**: Engineers creating, modifying, or debugging tasks and grader configurations

---

## Table of Contents

1. [What is a Task?](#what-is-a-task)
2. [Task Structure](#task-structure)
3. [Bootstrap Data](#bootstrap-data)
4. [Grader System Overview](#grader-system-overview)
5. [Grader Types](#grader-types)
6. [How Grading Works](#how-grading-works)
7. [Working with Tasks](#working-with-tasks)
8. [Common Patterns](#common-patterns)

---

## What is a Task?

A **task** is a test case for evaluating an AI agent's ability to navigate a website and complete a specific objective. Each task:

- Defines what the agent needs to accomplish (natural language instruction)
- Provides necessary context (bootstrap data like login credentials)
- Specifies how to validate success (grader configuration)
- Records metadata about features, actions, and complexity

Think of tasks as automated integration tests for AI agents instead of code.

---

## Task Structure

Each task is a JSON object with these key components:

```json
{
  "task_id": "10__user_0006__login_navigate_write_review_submit_review",
  "start_url": "https://cua-restaurant-reviews-cdn.develop.patronus.ai",
  "task": "Please help me write a four-star review for Beit Rima...",
  "simulator_config": {
    "bootstrap_data": {
      "user": {
        "email": "morgan.johnson@email.com",
        "password": "password123"
      }
    }
  },
  "grader_config": {
    "state_grader_configs": [...],
    "answer_grader_config": {...},
    "url_grader_config": {...},
    "llm_grader_config": {...}
  },
  "metadata": {
    "primary_feature": "write_review",
    "expected_actions": ["login", "navigate", "write_review", "submit"],
    "llm_capabilities": [...]
  }
}
```

### Key Fields

- **task_id**: Unique identifier (format: `{feature}__{actions}__{sequence}`)
- **start_url**: Where the agent begins
- **task**: Natural language instruction for the agent
- **simulator_config**: Bootstrap data for task execution (credentials, etc.)
- **grader_config**: How to validate task completion
- **metadata**: Context about features, actions, and capabilities

---

## Bootstrap Data

**Bootstrap data** provides the agent with necessary context to complete authenticated or personalized tasks.

### What is Bootstrap Data?

When a task requires authentication or specific user context (e.g., "join a waitlist", "write a review"), bootstrap data provides the credentials or IDs needed without requiring the agent to create them.

### Example: User Authentication

```json
"simulator_config": {
  "bootstrap_data": {
    "user": {
      "email": "morgan.johnson@email.com",
      "password": "password123"
    }
  }
}
```

**Task instruction might say**: "Write a review for Beit Rima"

**The agent receives**: Pre-existing credentials it can use to log in

### Why Bootstrap Data?

1. **Reduces complexity**: Agent doesn't need to handle account creation
2. **Ensures consistency**: Uses known entities that exist in the test database
3. **Enables validation**: Graders can verify actions using known user IDs
4. **Realistic scenarios**: Mimics real-world use where users already have accounts

### Common Bootstrap Types

- **User credentials**: email, password, user_id
- **Entity references**: restaurant IDs, reservation IDs
- **Pre-existing state**: collections, favorites, preferences

---

## Grader System Overview

The grader system validates whether an agent successfully completed a task using multiple validation layers:

```
┌─────────────────────────────────────────┐
│           Task Execution                │
│  (Agent navigates and performs actions) │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│         Grading Layers                  │
├─────────────────────────────────────────┤
│  1. State Graders                       │ ← Verify backend state changes
│  2. Answer Graders                      │ ← Validate text output
│  3. URL Graders                         │ ← Check final URL
│  4. LLM Graders                         │ ← Semantic validation
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│         Grading Result                  │
│  Pass/Fail + detailed breakdown         │
└─────────────────────────────────────────┘
```

### Grading Philosophy

- **Multi-layered**: Combine precise (state/text) and flexible (LLM) validation
- **Specific assertions**: Each grader checks one specific aspect
- **Expected state**: Compare actual vs expected state from backend
- **Weight-based**: Graders can have different importance weights

---

## Grader Types

### 1. State Graders

**Purpose**: Validate that backend state changed correctly (e.g., reservation created, review submitted)

**How it works**:
1. Extract actual state from backend (via API call or database query)
2. Compare against expected values using JSONPath assertions
3. Validate specific fields match expectations

**Example: Waitlist Validation**

```json
{
  "name": "Waitlist joined validation",
  "weight": 1,
  "path_to_actual": "$.waitlists[0]",
  "assertions": [
    {
      "name": "Correct phone number in waitlist",
      "operator": "STRING_CONTAINS",
      "expected": ["533819300"],
      "path_to_actual": "$.contactInfo.phone"
    },
    {
      "name": "Correct party size",
      "operator": "NUMERIC_MATCH",
      "expected": ["==3"],
      "path_to_actual": "$.partySize"
    }
  ]
}
```

**Reads**: Backend state from application
**Validates**:
- Waitlist entry exists
- Phone number matches task requirement
- Party size is correct

### 2. Answer Graders

**Purpose**: Validate the agent's final text response contains expected information

**How it works**: Check if the agent's output text contains specific strings or patterns

**Example: Restaurant Search**

```json
{
  "name": "Answer grader",
  "assertions": [
    {
      "name": "Answer contains restaurant name",
      "operator": "STRING_CONTAINS",
      "expected": ["Hungry Like The Wolf"]
    }
  ]
}
```

**Validates**: Agent mentioned the correct restaurant in its response

### 3. URL Graders

**Purpose**: Verify the agent navigated to the correct page or applied correct filters

**How it works**: Check final URL contains expected query parameters or paths

**Example: Search Filters**

```json
{
  "name": "URL grader",
  "assertions": [
    {
      "name": "URL contains price range filter",
      "operator": "STRING_CONTAINS",
      "expected": ["$$"],
      "path_to_actual": "$.url"
    },
    {
      "name": "URL contains location",
      "operator": "STRING_CONTAINS",
      "expected": ["San+Francisco"],
      "path_to_actual": "$.url"
    }
  ]
}
```

**Validates**: Agent applied correct search filters visible in URL

### 4. LLM Graders

**Purpose**: Flexible semantic validation for aspects that are hard to test precisely

**How it works**: An LLM evaluates whether task completion criteria are met based on:
- Task instruction
- Agent trajectory (actions taken)
- Final state/answer

**Example: Review Quality**

```json
{
  "name": "LLM judge",
  "instruction": "Verify that a review with 4 stars is submitted mentioning the falafel.",
  "include_trajectory": true
}
```

**Validates**: Semantic aspects like:
- Content quality
- Mention of specific details (e.g., "falafel")
- Overall task completion

**When to use LLM graders**:
- Content quality assessment
- Semantic matching (not exact string match)
- Contextual understanding
- Subjective criteria

---

## How Grading Works

### Step 1: Task Execution

```
Agent receives task → Executes actions → Produces final state/answer
```

### Step 2: State Extraction (if configured)

```json
"extract_states_config": {
  "expected_state_functions": [
    {
      "function": "get_restaurants_state",
      "args": {
        "location": "San Francisco",
        "cuisine": "Italian"
      }
    }
  ]
}
```

**Purpose**: Fetch expected state from backend for comparison
- Calls backend function with specific parameters
- Returns expected data (e.g., restaurants matching criteria)
- Used by state graders for validation

### Step 3: Run Graders

Each grader type runs independently:

```python
# Pseudo-code grading flow
results = []

# 1. State graders (if configured)
for state_grader in task.grader_config.state_grader_configs:
    actual_state = extract_state(state_grader.path_to_actual)
    for assertion in state_grader.assertions:
        result = evaluate_assertion(actual_state, assertion)
        results.append(result)

# 2. Answer grader (if configured)
if task.grader_config.answer_grader_config:
    for assertion in answer_grader.assertions:
        result = evaluate_assertion(agent_answer, assertion)
        results.append(result)

# 3. URL grader (if configured)
if task.grader_config.url_grader_config:
    for assertion in url_grader.assertions:
        result = evaluate_assertion(final_url, assertion)
        results.append(result)

# 4. LLM graders (if configured)
for llm_grader in task.grader_config.llm_grader_configs:
    result = llm_evaluate(
        task=task.task,
        trajectory=agent_trajectory,
        instruction=llm_grader.instruction
    )
    results.append(result)
```

### Step 4: Aggregate Results

```
Overall Pass = All weighted graders pass
Overall Fail = Any critical grader fails

Result includes:
- Pass/Fail status
- Individual grader results
- Detailed error messages
- Confidence scores (for LLM graders)
```

---

## Working with Tasks

### Creating a New Task

**Manual Creation** (for custom tasks):

```json
{
  "task_id": "custom_001",
  "start_url": "https://example.com",
  "task": "Find Italian restaurants in San Francisco",
  "grader_config": {
    "answer_grader_config": {
      "name": "Answer validation",
      "assertions": [
        {
          "operator": "STRING_CONTAINS",
          "expected": ["Italian", "San Francisco"]
        }
      ]
    }
  }
}
```

**Generated Tasks** (via task generation pipeline):

The task generation pipeline automatically creates tasks with:
- Diverse complexity levels
- Feature coverage
- Auto-generated grader configs
- Validated entity references

See `task_generation_pipeline/` for details on automated task generation.

### Modifying an Existing Task

1. **Locate task**: Find by `task_id` in `data/{environment}/tasks.json`
2. **Update fields**: Modify task instruction, grader config, or metadata
3. **Test**: Run the task to validate changes
4. **Document**: Update metadata if changing expected actions or features

### Debugging Failed Tasks

**Step 1: Check which grader failed**

```json
{
  "task_id": "...",
  "passed": false,
  "grader_results": [
    {
      "name": "State grader - Waitlist validation",
      "passed": false,
      "assertions": [
        {
          "name": "Correct phone number",
          "passed": false,
          "expected": ["533819300"],
          "actual": "533819301"
        }
      ]
    }
  ]
}
```

**Step 2: Determine root cause**

- **State grader failure**: Check if agent performed correct action, or if backend state is wrong
- **Answer grader failure**: Check if agent's response format changed
- **URL grader failure**: Check if agent navigated to correct page
- **LLM grader failure**: Review trajectory to understand semantic issue

**Step 3: Fix**

- If agent behavior is wrong: Investigate agent logic
- If grader is too strict: Adjust assertion thresholds
- If grader is incorrect: Update grader config

---

## Common Patterns

### Pattern 1: Authentication Tasks

**Structure**:
- Bootstrap data with user credentials
- State grader validates login state
- Expected actions include "login"

```json
{
  "task": "Log in and view your profile",
  "simulator_config": {
    "bootstrap_data": {
      "user": {"email": "...", "password": "..."}
    }
  },
  "grader_config": {
    "state_grader_configs": [
      {
        "name": "User logged in",
        "path_to_actual": "$.session",
        "assertions": [
          {
            "operator": "STRING_EQUALS",
            "expected": ["user_0001"],
            "path_to_actual": "$.userId"
          }
        ]
      }
    ]
  }
}
```

### Pattern 2: Search and Filter Tasks

**Structure**:
- Extract expected state from backend (restaurants matching criteria)
- Answer grader validates restaurant names in output
- URL grader validates search parameters

```json
{
  "task": "Find Italian restaurants in San Francisco under $$",
  "grader_config": {
    "extract_states_config": {
      "expected_state_functions": [
        {
          "function": "get_restaurants_state",
          "args": {
            "location": "San Francisco",
            "cuisine": "Italian",
            "price_range": ["$$"]
          }
        }
      ]
    },
    "answer_grader_config": {
      "assertions": [
        {
          "operator": "STRING_CONTAINS",
          "paths_to_expected": ["$[0].result.restaurants[0].name"]
        }
      ]
    },
    "url_grader_config": {
      "assertions": [
        {
          "operator": "STRING_CONTAINS",
          "expected": ["cuisine=Italian", "priceRange=$$"],
          "path_to_actual": "$.url"
        }
      ]
    }
  }
}
```

### Pattern 3: Data Creation Tasks

**Structure**:
- Bootstrap data with user credentials
- State grader validates new entity created
- LLM grader validates content quality

```json
{
  "task": "Write a 5-star review for Freemans mentioning their BBQ ribs",
  "simulator_config": {
    "bootstrap_data": {
      "user": {"email": "...", "password": "..."}
    }
  },
  "grader_config": {
    "state_grader_configs": [
      {
        "name": "Review created",
        "path_to_actual": "$.reviews[0]",
        "assertions": [
          {
            "operator": "NUMERIC_MATCH",
            "expected": ["==5"],
            "path_to_actual": "$.rating"
          },
          {
            "operator": "STRING_EQUALS",
            "expected": ["user_0007"],
            "path_to_actual": "$.userId"
          }
        ]
      }
    ],
    "llm_grader_configs": [
      {
        "instruction": "Verify that the review mentions BBQ ribs",
        "include_trajectory": true
      }
    ]
  }
}
```

### Pattern 4: Multi-Step Tasks

**Structure**:
- Multiple expected actions
- State graders for each intermediate step
- LLM grader for overall completion

```json
{
  "task": "Find Hungry Like The Wolf in Houston, join waitlist for 3",
  "metadata": {
    "expected_actions": [
      "login",
      "search",
      "navigate",
      "join_waitlist",
      "view_confirmation"
    ]
  },
  "grader_config": {
    "state_grader_configs": [
      {
        "name": "Waitlist entry created",
        "path_to_actual": "$.waitlists[0]",
        "assertions": [
          {"operator": "STRING_EQUALS", "expected": ["active"], "path_to_actual": "$.status"},
          {"operator": "NUMERIC_MATCH", "expected": ["==3"], "path_to_actual": "$.partySize"}
        ]
      }
    ],
    "llm_grader_configs": [
      {
        "instruction": "Verify agent searched for restaurant, joined waitlist with correct party size, and viewed confirmation",
        "include_trajectory": true
      }
    ]
  }
}
```

---

## Assertion Operators Reference

### String Operators

- **STRING_EQUALS**: Exact match (case-sensitive)
- **STRING_CONTAINS**: Substring match
- **STRING_NOT_CONTAINS**: Substring must not be present
- **STRING_FUZZY_MATCH**: Fuzzy/approximate matching

### Numeric Operators

- **NUMERIC_MATCH**: Numeric comparison with operators
  - `"==5"`: Exactly 5
  - `">=4"`: Greater than or equal to 4
  - `">2"`: Greater than 2
  - `"<=3"`: Less than or equal to 3

### Array Operators

- **ARRAY_LENGTH_MATCH**: Validate array length
  - `"==3"`: Exactly 3 elements
  - `">=2"`: At least 2 elements
- **ARRAY_STRING_CONTAINS**: Array contains specific string element

---

## JSONPath Reference

Graders use JSONPath to navigate JSON structures:

```json
{
  "waitlists": [
    {
      "id": "wl_001",
      "status": "active",
      "contactInfo": {
        "phone": "5551234567"
      }
    }
  ]
}
```

**JSONPath examples**:
- `$.waitlists` → Access waitlists array
- `$.waitlists[0]` → First waitlist entry
- `$.waitlists[0].status` → Status field of first entry
- `$.waitlists[0].contactInfo.phone` → Nested phone field

**In graders**:
- `path_to_actual: "$.waitlists[0]"` → Extract this object as actual state
- Nested assertion `path_to_actual: "$.contactInfo.phone"` → Within that object, check this field

---

## Best Practices

### Task Design

1. **Clear instructions**: Task text should be unambiguous
2. **Realistic scenarios**: Mirror real user behavior
3. **Complete bootstrap data**: Provide all necessary context
4. **Appropriate complexity**: Match difficulty to evaluation goals

### Grader Design

1. **Multiple layers**: Use 2-3 grader types for comprehensive validation
2. **Specific assertions**: Each assertion checks one thing
3. **Meaningful names**: Assertion names should explain what's being validated
4. **Weighted appropriately**: Critical checks should have higher weights
5. **LLM as supplement**: Use LLM graders for semantic validation, not as only grader

### Debugging

1. **Read logs**: Check agent trajectory to understand actions taken
2. **Validate grader logic**: Ensure assertions match task requirements
3. **Test in isolation**: Run individual graders separately to identify issues
4. **Check bootstrap data**: Ensure entities exist and have correct values

---

## Quick Reference

### Task Anatomy
```
task_id + start_url + task (instruction)
    ↓
simulator_config (bootstrap data)
    ↓
grader_config (validation rules)
    ↓
metadata (context and classification)
```

### Grader Flow
```
Execute Task
    ↓
Extract States (if needed)
    ↓
State Graders → Validate backend changes
Answer Graders → Validate text output
URL Graders → Validate navigation
LLM Graders → Validate semantic completion
    ↓
Aggregate Results → Pass/Fail
```

### Key Files

- `data/{environment}/tasks.json` - Task definitions
- `data/{environment}/config.yaml` - Environment configuration
- `task_generation_pipeline/` - Automated task generation
- `src/grader/` - Grader implementation

---

## Additional Resources

- **Task Generation Pipeline**: See `task_generation_pipeline/CLAUDE.md` for automated task creation
- **Grader Implementation**: See `src/grader/` for grader code details
- **Example Tasks**: Browse `data/restaurant-reviews/tasks.json` for real examples

---

## Getting Help

When working with tasks and graders:

1. **Check existing tasks**: Look for similar patterns in `tasks.json`
2. **Review grader logs**: Detailed assertion results explain failures
3. **Test iteratively**: Start with simple graders, add complexity gradually
4. **Validate bootstrap data**: Ensure referenced entities exist in test database

Happy task creating! 🚀
