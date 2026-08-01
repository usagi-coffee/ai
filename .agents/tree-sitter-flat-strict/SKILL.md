---
name: tree-sitter-flat-strict
description: Create, extend, or review tree-sitter grammars and corpus tests. Use when editing grammar rules, precedence or conflict declarations, tokens, external scanners, corpus fixtures, or syntax-tree shape.
---

# Tree-sitter (Flat/Strict variant)

Write precise grammar rules with deliberate syntax-tree shape. Preserve stricter repository conventions.

## Workflow

1. Read the repository instructions and find its grammar entry point, rule modules, corpus tests, keyword helpers, and formatter or test scripts.
2. Inspect nearby rules and tests before inventing a new pattern.
3. Check the language reference and representative source. Do not derive the full syntax from one sample.
4. Parse a minimal existing example to understand the current tree.
5. Add corpus cases, then make the narrowest grammar change that supports them.
6. Run focused tests while editing, then the formatter, generation command, and full grammar suite.
7. Inspect the tree. Treat unexpected `ERROR` and `MISSING` nodes as failures.

Prefer repository scripts over raw tree-sitter commands. Do not remove or weaken valid tests to make a change pass.

## Model Exact Syntax

Keep required tokens, ordering, and repetition explicit:

```js
binding: ($) =>
  seq(
    "let",
    field("name", $.identifier),
    optional(seq(":", field("type", $._type))),
    optional(seq("=", field("value", $._expression))),
    ";",
  ),
```

Do not replace ordered options with a permissive loop:

```js
// Too broad: accepts either part repeatedly and in any order.
repeat(choice(seq(":", field("type", $._type)), seq("=", field("value", $._expression))));
```

- Use `optional`, `repeat`, and `repeat1` only where absence or repetition is legal.
- Avoid catch-all token sequences, broad regexes, and fallback branches.
- Keep delimiters and terminators explicit.
- Keep lexical choices in tokens and syntactic choices in grammar rules.
- Use an external scanner only when ordinary tokens cannot express the required lexical state.
- Follow the repository's keyword, case, and word-boundary policy.

## Organize Rules by Meaning

- Match public rule names to language-reference terminology.
- Use public rules for meaningful AST nodes and hidden rules for structural details.
- Keep feature-specific helpers close to their owner.
- Share a helper only when it represents genuinely shared syntax and tree shape.
- Follow an existing local-helper prefix convention in modular grammars.
- Prefer semantic plurals such as `_arguments` over `_argument_list`.
- Keep short one-line rules adjacent and avoid comments that restate the code.

For a comma-separated construct, expose repeated children through the same field:

```js
_arguments: ($) =>
  seq(
    field("argument", $._expression),
    repeat(seq(",", field("argument", $._expression))),
  ),
```

Do not reuse `_arguments` for another comma-separated construct unless it accepts the same syntax and should produce the same tree.

## Design the Tree Deliberately

### Valued parts

Use a field:

```js
// Avoid
seq(":", $._type);

// Prefer
seq(":", field("type", $._type));
```

Repeat the field for repeated values:

```js
seq(field("item", $.identifier), repeat(seq(",", field("item", $.identifier))));
```

### Trivial flags

Alias an anonymous flag directly:

```js
alias("?", $.optional);
```

Avoid a wrapper and an `_option` suffix:

```js
optional_marker_option: ($) => "?";
```

Starting keywords, terminators, and meaningful choices are not trivial flags.

### Semantic phrases

Alias the whole phrase when its parts belong to one semantic option:

```js
alias(seq("from", field("source", $.identifier)), $.from_clause);
```

For a reused phrase, keep its helper hidden and alias it at the call site:

```js
optional(alias($._from_clause, $.from_clause)),
_from_clause: ($) =>
  seq("from", field("source", $.identifier)),
```

Do not add a phrase node solely to wrap one field. Use it when the phrase itself is meaningful or owns optional or repeated children.

### Lexical aliases

Keep a token helper lexical and assign its AST identity at the call site:

```js
alias($._special_identifier, $.identifier),
_special_identifier: ($) => token(/#[a-z]+/),
```

This leaves other call sites free to give the token a different identity.

## Resolve Ambiguity in Order

1. Verify the documented syntax and token specificity.
2. Extract a named rule when precedence needs a stable target.
3. Use grammar-level precedence for competing syntactic forms.
4. Use associativity for recursive operators.
5. Add `conflicts` only when the earlier approaches are insufficient.

Use associativity when it is the actual language rule:

```js
binary_expression: ($) =>
  prec.left(seq(
    field("left", $._expression),
    field("operator", "+"),
    field("right", $._expression),
  )),
```

Do not scatter `prec(...)` wrappers around unrelated rules to silence conflicts. For a non-obvious precedence group, comment the ambiguity, a minimal source example, and the relevant language-reference rule.

When generation reports a conflict, focus on the concrete token sequence and named grammar symbols rather than generated helper names.

## Corpus Coverage

For new syntax, cover:

- the minimal form;
- meaningful alternatives;
- valued parts and flags;
- legal combinations and ordering;
- nesting or expression interaction;
- boundaries with similar constructs.

Do not add expected `ERROR` or `MISSING` nodes to a successful fixture. Use the repository's negative or recovery-test convention for intentionally invalid input.

## Final Check

- Supported forms parse with the intended tree.
- Obvious undocumented forms do not parse cleanly.
- Fields, aliases, and visible nodes are useful.
- Hidden helpers, punctuation, and terminators do not leak accidentally.
- Focused and full tests pass.
- The formatted diff contains no unrelated changes.
