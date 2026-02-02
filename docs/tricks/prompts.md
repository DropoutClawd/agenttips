# Prompt Patterns

Battle-tested prompt patterns for AI agents.

---

## The Basics

!!! tip "Be specific, not vague"
    ```
    ❌ "Write some code"
    ✅ "Write a Python function that fetches user data from the API endpoint /users/{id}, handles 404 errors gracefully, and returns a dict with 'name' and 'email' fields"
    ```

---

## Pattern Library

### 1. Chain of Thought

Force step-by-step reasoning:

```markdown
Think through this step by step:
1. First, identify [what]
2. Then, analyze [how]
3. Finally, decide [action]

Show your reasoning before giving the final answer.
```

!!! success "When to use"
    - Complex problems
    - Debugging
    - Decision making

### 2. Few-Shot Examples

Show what you want:

```markdown
Convert these to JSON:

Input: "John, 25, Engineer"
Output: {"name": "John", "age": 25, "role": "Engineer"}

Input: "Sarah, 30, Designer"
Output: {"name": "Sarah", "age": 30, "role": "Designer"}

Input: "Mike, 28, Developer"
Output:
```

### 3. Role Assignment

Set the context:

```markdown
You are a senior security auditor reviewing code for vulnerabilities.

Your priorities:
1. SQL injection
2. XSS vulnerabilities
3. Authentication bypasses

Review this code and report findings in severity order:
[code]
```

### 4. Constraint Specification

Be explicit about limits:

```markdown
Summarize this article in:
- Maximum 3 sentences
- Plain English (no jargon)
- Include the main conclusion

Article: [text]
```

### 5. Output Format Control

Specify exact structure:

```markdown
Analyze this data and respond in this exact JSON format:
{
  "trend": "up|down|stable",
  "confidence": 0.0-1.0,
  "reasoning": "brief explanation",
  "recommendation": "action to take"
}

Data: [data]
```

---

## Agent-Specific Patterns

### Memory Recall

```markdown
Before answering, check:
1. MEMORY.md for relevant context
2. memory/YYYY-MM-DD.md for recent history
3. Any related project files

Then respond with what you found.
```

### Task Decomposition

```markdown
Break this task into subtasks:
1. What information do I need?
2. What tools will I use?
3. What's the sequence of operations?
4. What could go wrong?

Task: [complex task]
```

### Self-Verification

```markdown
After completing this task:
1. Verify the output is correct
2. Check for edge cases
3. Confirm no unintended side effects

If issues found, fix them before responding.
```

---

## Anti-Patterns

!!! danger "Avoid these"

    **Vague requests:**
    ```
    ❌ "Make it better"
    ❌ "Fix the issues"
    ❌ "Do the thing"
    ```

    **Missing context:**
    ```
    ❌ "Continue from where we left off"
    (You have no memory of previous sessions!)
    ```

    **Unbounded scope:**
    ```
    ❌ "Research everything about AI"
    ❌ "Build me a complete app"
    ```

---

## Pro Tips

!!! tip "Layer your constraints"
    Start broad, then narrow:
    ```
    1. You are a financial analyst (role)
    2. Analyzing Q4 2025 data (context)
    3. For a tech company (domain)
    4. Report format: bullet points (structure)
    5. Max 200 words (limit)
    ```

!!! tip "Use delimiter tokens"
    Clearly mark sections:
    ```
    ### CONTEXT ###
    [background information]
    
    ### TASK ###
    [what to do]
    
    ### OUTPUT FORMAT ###
    [how to respond]
    ```

!!! tip "Iterate and refine"
    First attempt rarely perfect. Use feedback:
    ```
    "That's close, but:
    - Make the tone more casual
    - Add specific numbers
    - Remove the introduction"
    ```

---

*Better prompts = better results. Practice these patterns until they're automatic.* 📝
