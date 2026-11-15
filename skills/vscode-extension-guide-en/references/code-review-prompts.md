# Code review prompts and templates

Use these prompts and templates for code reviews in VS Code chat.

## Review checklist

| Area | What to check |
|------|---------------|
| Bugs and logic | Runtime errors, edge cases, null and undefined values |
| Security | XSS, injection, sensitive data exposure |
| Performance | N+1 queries, unnecessary re-renders, memory leaks |
| Maintainability | Naming, code structure, complexity |
| Tests | Missing test cases |
| Documentation | Comments, JSDoc, README updates |

## Feedback format

Classify findings by severity:

- Critical: Must fix before merge
- Warning: Should fix
- Suggestion: Optional improvement

---

## Prompt templates

### 1. Simple PR review

```markdown
Review the changes in this PR for critical correctness, security, and performance issues.

List must-fix issues with file and line references. If you find none, say so.
```

### 2. Comprehensive code review

```markdown
Review the code for:

- Bugs and edge cases
- Security vulnerabilities
- Performance issues
- Maintainability
- Missing tests
- Documentation errors

Classify findings as critical, warning, or suggestion. Include line references
and suggested fixes.
```

### 3. Git diff review

```markdown
Review the git diff for bugs and regressions. Report findings with line references.
```

### 4. Security-focused review

```markdown
Review this code focusing exclusively on security vulnerabilities:

1. **Input Validation**: Check for missing validation
2. **Authentication/Authorization**: Verify access controls
3. **Data Exposure**: Look for sensitive data leaks
4. **Injection**: Check for SQL/NoSQL/Command injection
5. **XSS**: Verify output encoding
6. **CSRF**: Check token validation
7. **Dependencies**: Note any known vulnerable packages

Rate severity as critical, high, medium, or low.
```

### 5. TypeScript and React review

```markdown
Review this TypeScript/React code for:

1. **Type Safety**: Any `any` types that should be specific?
2. **React Patterns**: Hooks rules, key props, memo usage
3. **State Management**: Unnecessary re-renders, state location
4. **Error Boundaries**: Missing error handling
5. **Accessibility**: Missing ARIA attributes, keyboard navigation

Provide specific fixes with code examples.
```

---

## Instructions file template

### code-review.instructions.md

```markdown
---
name: Code Review
description: Review code for bugs, security issues, and performance problems
applyTo: "**/*.{ts,tsx,js,jsx,py}"
---

# Code Review Instructions

Review the code using this checklist.

## Review Checklist
1. **Bugs & Logic Errors**: Runtime errors, edge cases, null handling
2. **Security**: XSS, injection, sensitive data exposure
3. **Performance**: N+1 queries, unnecessary re-renders, memory leaks
4. **Maintainability**: Readability, naming, code structure
5. **Type Safety**: Proper TypeScript types
6. **Test Coverage**: Missing test cases

## Output Format
- Critical: Must fix before merge
- Warning: Should address
- Suggestion: Optional improvement

Keep feedback constructive and specific with line references.
```

---

## Custom agent template

### code-reviewer.agent.md

```markdown
---
name: Code Reviewer
description: Review PR changes for bugs, security issues, and performance problems
tools:
  - read
  - search
---

# Code Reviewer Agent

Review the changes for bugs, security issues, performance problems, and missing tests.
Report actionable findings with line references and concrete fixes.
```
