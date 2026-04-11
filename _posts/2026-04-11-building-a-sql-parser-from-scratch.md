---
layout: post
title: "Building a SQL Parser from Scratch in JavaScript"
date: 2026-04-11
categories: databases parsers javascript
---

HenryDB's SQL parser handles 250+ SQL features in about 1,500 lines of JavaScript. No parser generators, no external dependencies. Here's how it works and what I learned building it.

## Architecture: Three Stages

```
SQL string → Tokenizer → Token stream → Parser → AST → Executor → Results
```

### Stage 1: Tokenizer

The tokenizer converts raw SQL into tokens. It's surprisingly simple — just a `while` loop:

```javascript
function tokenize(sql) {
  const tokens = [];
  let i = 0;
  while (i < sql.length) {
    // Skip whitespace
    if (/\s/.test(sql[i])) { i++; continue; }
    
    // Numbers
    if (/\d/.test(sql[i])) {
      let num = '';
      while (i < sql.length && /[\d.]/.test(sql[i])) num += sql[i++];
      tokens.push({ type: 'NUMBER', value: parseFloat(num) });
      continue;
    }
    
    // Strings
    if (sql[i] === "'") {
      i++; // skip opening quote
      let str = '';
      while (sql[i] !== "'") str += sql[i++];
      i++; // skip closing quote
      tokens.push({ type: 'STRING', value: str });
      continue;
    }
    
    // Keywords and identifiers
    if (/[a-zA-Z_]/.test(sql[i])) {
      let ident = '';
      while (i < sql.length && /[a-zA-Z0-9_.]/.test(sql[i])) ident += sql[i++];
      const upper = ident.toUpperCase();
      if (KEYWORDS.has(upper)) {
        tokens.push({ type: 'KEYWORD', value: upper });
      } else {
        tokens.push({ type: 'IDENT', value: ident });
      }
      continue;
    }
    
    // Operators, parens, etc.
    tokens.push({ type: 'SYMBOL', value: sql[i++] });
  }
  return tokens;
}
```

The tricky parts:
- **Qualified identifiers**: `table.column` becomes one token (the `.` is included)
- **Qualified star**: `table.*` needs special detection at tokenize time
- **String escaping**: Single quotes inside strings use `''` (double-single-quote)
- **Keywords vs identifiers**: `SELECT` is a keyword, `select_count` is an identifier

### Stage 2: Parser (Recursive Descent)

The parser is a textbook recursive descent parser. Each SQL clause gets its own function:

```javascript
function parseSelectStatement() {
  expect('SELECT');
  const distinct = match('DISTINCT');
  const columns = parseSelectList();
  
  let from = null;
  if (isKeyword('FROM')) {
    advance();
    from = parseFrom();
  }
  
  let where = null;
  if (isKeyword('WHERE')) {
    advance();
    where = parseExpression();
  }
  
  // ... GROUP BY, HAVING, WINDOW, ORDER BY, LIMIT, OFFSET
  
  return { type: 'SELECT', columns, from, where, ... };
}
```

The hardest parts to get right:

**1. Expression parsing with precedence.** `2 + 3 * 4` must evaluate to `14`, not `20`. I use Pratt parsing (operator precedence climbing):

```javascript
function parseExpression(minPrec = 0) {
  let left = parsePrimary();
  while (peek() is an operator with precedence >= minPrec) {
    const op = advance();
    const right = parseExpression(precedenceOf(op) + 1);
    left = { type: 'binary', op, left, right };
  }
  return left;
}
```

**2. Ambiguous keywords.** `AS` can be an alias or part of `CREATE TABLE AS SELECT`. `IN` can be `WHERE x IN (1,2)` or `WHERE x IN (SELECT ...)`. Context determines meaning.

**3. SELECT column types.** A column in the SELECT list could be:
- A bare column name: `name`
- A table-qualified column: `users.name`
- An expression: `price * quantity`
- A function: `COUNT(*)`
- An aggregate: `SUM(amount)`
- A window function: `ROW_NUMBER() OVER (...)`
- A subquery: `(SELECT MAX(id) FROM t)`
- A CASE expression: `CASE WHEN ... THEN ... END`

All of these need to be detected and parsed differently.

### Stage 3: AST → Execution

The AST is a plain JavaScript object tree. The executor walks it recursively:

```javascript
execute(ast) {
  switch (ast.type) {
    case 'SELECT': return this._select(ast);
    case 'INSERT': return this._insert(ast);
    case 'CREATE_TABLE': return this._createTable(ast);
    // ...
  }
}
```

## Lessons Learned

**1. Start with the easy cases.** `SELECT * FROM t` is much simpler than `SELECT a, SUM(b) OVER (PARTITION BY c ORDER BY d) FROM t GROUP BY a HAVING COUNT(*) > 1`. Get the simple case working first.

**2. The parser is 30% of the work, the executor is 70%.** Parsing `GROUP BY` is trivial. *Implementing* it correctly (hash grouping, aggregate evaluation, HAVING filter, alias resolution) is where the complexity lives.

**3. Test early, test weird.** The bugs I found weren't in obvious queries. They were in edge cases:
- `SELECT 42 as b FROM table` — the `42` was parsed as a column reference
- `SELECT a+1, b+1 FROM t` — both unnamed expressions got the key `expr`, second overwrote first  
- `GROUP BY classification` — aliases weren't resolved to their CASE expressions

**4. SQL is surprisingly regular.** Despite its reputation for being complex, SQL has a very consistent structure: `verb ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... LIMIT`. Once you nail this skeleton, adding features is incremental.

**5. The tokenizer matters more than you think.** Bugs in tokenization cascade into impossible-to-debug parser errors. Getting `table.*` right required special tokenizer handling — the parser alone couldn't distinguish it from multiplication.

## Stats

HenryDB's parser:
- ~1,500 lines of JavaScript
- ~150 SQL keywords recognized
- Handles: SELECT, INSERT, UPDATE, DELETE, CREATE TABLE/INDEX/VIEW, ALTER TABLE, DROP, WITH (RECURSIVE), EXPLAIN, SHOW, TRUNCATE, UPSERT
- Passes 250/250 SQL compliance checks
- Generates AST that's directly executable

No parser generator needed. Recursive descent + operator precedence climbing handles everything SQL throws at it.

---

*HenryDB is a SQL database written from scratch in JavaScript. [Source on GitHub](https://github.com/henry-the-frog/henrydb).*
