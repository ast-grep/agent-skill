---
name: ast-grep
description: Structural code search with ast-grep. Use for AST pattern search (`run --pattern`), YAML rules (`scan`) for relational/composite queries, and parsing JSON matches with captured metavariables.
---

# ast-grep Code Search

## Overview

This skill translates natural language queries into ast-grep patterns and rules. ast-grep matches code by its Abstract Syntax Tree (AST) structure rather than text, enabling precise search across large codebases.

## When to Use This Skill

Use this skill when users:
- Need to search for code patterns using structural matching (e.g., "find all async functions that don't have error handling")
- Want to locate specific language constructs (e.g., "find all function calls with specific parameters")
- Request searches that require understanding code structure or particular AST characteristics rather than just text
- Need to perform complex code queries that traditional text search cannot handle

## General Workflow

Follow this process to help users find code with ast-grep. Start with simple pattern searches and escalate to YAML rules only when the query needs them:

### Step 1: Understand the Query

Clearly understand what the user wants to find. Ask clarifying questions if needed:
- What specific code pattern or structure are they looking for?
- Which programming language?
- Are there specific edge cases or variations to consider?
- What should be included or excluded from matches?

### Step 2: Create Example Code

Write a snippet representing what the user wants to match.

**Example:**
If searching for "async functions that use await", create a test file:

```javascript
// test_example.js
async function example() {
  const result = await fetchData();
  return result;
}
```

### Step 3: Try a Pattern Search

For simple, single-node matches, use `run --pattern`. ast-grep infers the language from the file extension; `--lang` is only required for `--stdin` input:

```bash
ast-grep run --pattern 'console.log($ARG)' --lang javascript /path/to/project
```

- A snippet isn't required for a pattern search, but a quick one helps verify the pattern's node shape
- If the pattern returns zero matches, see the "Zero Matches?" tip in Tips and Troubleshooting — the node shape may differ from the pattern (e.g. bare vs fully-qualified paths)
- For programmatic use, add `--json`; see "Reading --json Output" for the schema
- If you know the target node `kind` but not the code shape, use `run --kind <KIND>` (also accepts ESQuery selectors like `call_expression:has(identifier)`)
- If the query needs relational or composite logic, continue to Step 4

### Step 4: Escalate to Rules for Complex Queries

Escalate to `scan` rules when you need relational (`inside`/`has`), composite (`all`/`any`/`not`), or negative logic.

**Write the ast-grep Rule**

Translate the pattern into an ast-grep rule. Start simple and add complexity as needed.

**Key principles:**
- Always use `stopBy: end` for relational rules (`inside`, `has`) — see "Always Use stopBy: end" in Tips and Troubleshooting
- Use `pattern` for simple structures
- Use `kind` with `has`/`inside` for complex structures
- Break complex queries into smaller sub-rules using `all`, `any`, or `not`

**Example rule file (test_rule.yml):**
```yaml
id: async-with-await
language: javascript
rule:
  kind: function_declaration
  has:
    pattern: await $EXPR
    stopBy: end
```

See `references/rule_reference.md` for comprehensive rule documentation.

**Test the Rule**

Use ast-grep CLI to verify the rule matches the snippet from Step 2. There are two main approaches:

**Option A: Test with inline rules (for quick iterations)**

Pipe a snippet through `scan --stdin` — see "Test Rules (scan with --stdin)" in ast-grep CLI Commands below.

**Option B: Test with rule files (recommended for complex rules)**
```bash
ast-grep scan --rule test_rule.yml test_example.js
```

**Debugging if no matches:**
1. Simplify the rule (remove sub-rules)
2. Add `stopBy: end` to relational rules if not present
3. Use `run --debug-query=cst` to understand the AST structure (see below)
4. Check if `kind` values are correct for the language
5. For zero matches from `run --pattern` (not rules), see the "Zero Matches?" tip below

### Step 5: Search the Codebase with the Rule

Once the rule matches the example code correctly, search the actual codebase:

```bash
ast-grep scan --rule my_rule.yml /path/to/project
```

For inline rules without creating a file, see "Search with Rules (scan)" under ast-grep CLI Commands.

## ast-grep CLI Commands

### Inspect Code Structure (--debug-query)

Dump the AST structure to understand how code is parsed:

```bash
ast-grep run --pattern 'async function example() { await fetch(); }' \
  --lang javascript \
  --debug-query=cst
```

**Available formats:**
- `cst`: Concrete Syntax Tree (shows all nodes including punctuation)
- `ast`: Abstract Syntax Tree (shows only named nodes)
- `pattern`: Shows how ast-grep interprets your pattern

**Use this to:**
- Find the correct `kind` values for nodes
- Understand the structure of code you want to match
- Debug why patterns/rules aren't matching (metavariables, node `kind`, relational direction)

**Example:**
```bash
# See the structure of your target code
ast-grep run --pattern 'class User { constructor() {} }' \
  --lang javascript \
  --debug-query=cst

# See how ast-grep interprets your pattern
ast-grep run --pattern 'class $NAME { $$$BODY }' \
  --lang javascript \
  --debug-query=pattern
```

### Test Rules (scan with --stdin)

Test a rule against code snippet without creating files:

```bash
echo "const x = await fetch();" | ast-grep scan --inline-rules "id: test
language: javascript
rule:
  pattern: await \$EXPR" --stdin
```

**Add --json for structured output:**
```bash
echo "const x = await fetch();" | ast-grep scan --inline-rules "..." --stdin --json
```

### Search with Patterns or Kinds (run)

Simple search for a single AST node, by `--pattern` or by `--kind`:

```bash
# Pattern search
ast-grep run --pattern 'console.log($ARG)' --lang javascript .

# Search specific files
ast-grep run --pattern 'class $NAME' --lang python /path/to/project

# Match by node kind (ESQuery selectors supported)
ast-grep run --kind call_expression --lang javascript .

# JSON output for programmatic use
ast-grep run --pattern 'function $NAME($$$)' --lang javascript --json .
```

**When to use:** a single-node match where a plain pattern or kind suffices — no relational rules (`inside`/`has`) or composite logic.

### Reading `--json` Output

`--json` prints a bare JSON **array** of matches (no `matches` wrapper). For pattern `foo($ARG, $$$REST)`:

```json
[{
  "text": "foo(\"value\", 1, 2)",
  "file": "src/app.js",
  "range": { "start": { "line": 41, "column": 10 }, "end": { "line": 41, "column": 27 } },
  "metaVariables": {
    "single": { "ARG": { "text": "\"value\"" } },
    "multi": { "REST": [ { "text": "1" }, { "text": "2" } ] }
  }
}]
```

`scan --json` uses the same schema.

(Each match also includes `lines` and `language`.) `range.start.line` is 0-based. Named metavariables (`$ARG`) land in `single`, list metavariables (`$$$REST`) in `multi` as a list (empty if nothing captured). Extract with jq:

```bash
ast-grep run --pattern 'foo($ARG)' --lang javascript --json . \
  | jq -r '.[] | "\(.file):\(.range.start.line + 1): \(.metaVariables.single.ARG.text)"'
```

### Search with Rules (scan)

YAML rule-based search for complex structural queries:

```bash
# With rule file
ast-grep scan --rule my_rule.yml /path/to/project

# With inline rules
ast-grep scan --inline-rules "id: find-async
language: javascript
rule:
  kind: function_declaration
  has:
    pattern: await \$EXPR
    stopBy: end" /path/to/project

# JSON output
ast-grep scan --rule my_rule.yml --json /path/to/project
```

**When to use:**
- Complex structural searches
- Relational rules (inside, has, precedes, follows)
- Composite logic (all, any, not)
- When you need the power of full YAML rules

## Tips and Troubleshooting

### Zero Matches? Patterns Match Whole AST Nodes

`run --pattern` matches complete AST nodes, not text substrings, so zero matches can mean "code absent" or "pattern has the wrong node shape":

- Qualified paths are whole nodes: `env::var($ENV)` does NOT match `std::env::var("X")`. Try both bare and fully-qualified forms, or use `$$$` to absorb extra intermediate nodes.
- To confirm, test the pattern on a known snippet and inspect it with `run --debug-query=pattern`; for rules, use the Step 4 checklist.

### Always Use stopBy: end

For relational rules, always use `stopBy: end` unless there's a specific reason not to:

```yaml
has:
  pattern: await $EXPR
  stopBy: end
```

This ensures the search traverses the entire subtree rather than stopping at the first non-matching node.

### Start Simple, Then Add Complexity

Begin with the simplest rule that could work:
1. Try a `pattern` first
2. If that doesn't work, try `kind` to match the node type
3. Add relational rules (`has`, `inside`) as needed
4. Combine with composite rules (`all`, `any`, `not`) for complex logic

### Use the Right Rule Type

- **Pattern**: For simple, direct code matching (e.g., `console.log($ARG)`)
- **Kind + Relational**: For complex structures (e.g., "function containing await")
- **Composite**: For logical combinations (e.g., "function with await but not in try-catch")

### Escaping in Inline Rules

When using `--inline-rules`, escape metavariables in shell commands:
- Use `\$VAR` instead of `$VAR` (shell interprets `$` as variable)
- Or use single quotes: `'$VAR'` works in most shells

**Example:**
```bash
# Correct: escaped $
ast-grep scan --inline-rules "rule: {pattern: 'console.log(\$ARG)'}" .

# Or use single quotes
ast-grep scan --inline-rules 'rule: {pattern: "console.log($ARG)"}' .
```

## Common Use Cases

### Find Functions with Specific Content

Find async functions that use await:
```bash
ast-grep scan --inline-rules "id: async-await
language: javascript
rule:
  all:
    - kind: function_declaration
    - has:
        pattern: await \$EXPR
        stopBy: end" /path/to/project
```

### Find Code Inside Specific Contexts

Find console.log inside class methods:
```bash
ast-grep scan --inline-rules "id: console-in-class
language: javascript
rule:
  pattern: console.log(\$\$\$)
  inside:
    kind: method_definition
    stopBy: end" /path/to/project
```

### Find Code Missing Expected Patterns

Find async functions without try-catch:
```bash
ast-grep scan --inline-rules "id: async-no-trycatch
language: javascript
rule:
  all:
    - kind: function_declaration
    - has:
        pattern: await \$EXPR
        stopBy: end
    - not:
        has:
          pattern: try { \$\$\$ } catch (\$E) { \$\$\$ }
          stopBy: end" /path/to/project
```

## Resources

### references/
Contains detailed documentation for ast-grep rule syntax:
- `rule_reference.md`: Comprehensive ast-grep rule documentation covering atomic rules, relational rules, composite rules, and metavariables

Load these references when detailed rule syntax information is needed.
