+++
title = "Don't be afraid of goyacc: making user-facing DSL for querying data"
date = 2026-02-02
tags = ["Greener", "Go"]
draft = false
+++

One of the core features of my [Greener](https://github.com/cephei8/greener) project is user-facing DSL for querying database with test results.<br />
For example, `status = "fail" AND name = "login_test"` fetches results of the "login_test" test that failed.

Surprisingly, Go's port of classic YACC parser generator was a relatively simple and straightforward way to parse user queries to further convert them in SQL queries.

Note: this post covers a very simplified approach for educational purposes.


## Goal {#goal}

The goal is to turn this:

```nil
status = "fail" AND name = "login_test"
```

Into this:

```sql
SELECT * FROM testcases WHERE status = 'fail' AND name = 'login_test'
```

The workflow consists of:

1.  Tokenizing query string
2.  Parsing tokens into abstract syntax tree (AST)
3.  Generating SQL from AST

However, the the problem will be approached in a slightly different order:

1.  Define AST
2.  Write goyacc grammar
3.  Implement lexer (tokenizer)
4.  Generate parser using goyacc
5.  Generate SQL from AST


## Implementation {#implementation}


### Define AST {#define-ast}

The AST represents the structure of the parsed query.<br />
For out simple DSL we need just several types:

```go
type Query struct {
	Condition Condition
}

type Condition any

type FieldCondition struct {
	Field string
	Value string
}

type AndCondition struct {
	Left  Condition
	Right Condition
}
```

`FieldCondition` represents `field = "value"`.<br />
`AndCondition` combines two conditions with `AND`.<br />
`Condition` is either `FieldCondition` or `AndCondition` (unfortunately Go doesn't have sum types; we could make it an interface, but for the sake of this example `any` would also work fine).

For example, `status = "fail" AND name = "login_test"` becomes:

```nil
           AndCondition
              /     \
  FieldCondition    FieldCondition
  (status="fail")   (name="login_test")
```


### Write goyacc grammar {#write-goyacc-grammar}

The grammar defines what valid queries look like:

```text
%{
package main
%}

%union {
    str       string
    condition Condition
}

%token <str> tokIDENT tokSTRING
%token tokAND tokEQ

%type <condition> query condition term

%left tokAND

%%

query:
    condition
    {
        yylex.(*lexer).result = &Query{Condition: $1}
    }

condition:
    condition tokAND term
    {
        $$ = &AndCondition{Left: $1, Right: $3}
    }
    | term
    {
        $$ = $1
    }

term:
    tokIDENT tokEQ tokSTRING
    {
        $$ = &FieldCondition{Field: $1, Value: $3}
    }

%%
```

To break it down:

-   `%union` defines what data types tokens and rules can carry. `str` is a string value, `condition` is AST node.
-   `%token` declares terminal symbols (tokens from the lexer). `<str>` means these tokens carry strings.
-   `%type` declares what type each grammar rule produces.
-   `%left tokAND` sets AND as left-associative (`a AND b AND c` groups as `(a AND b) AND c`).
-   The `%%...%%`  section contains the grammar rules. Each rule has a pattern and Go code in braces that builds AST nodes.
    -   `$1`, `$3` refer to matched elements by position, `$$` is the rule's output.


### Implement lexer (tokenizer) {#implement-lexer--tokenizer}

The lexer breaks input into tokens.<br />
It must implement the following interface:

```go
type yyLexer interface {
	Lex(lval *yySymType) int
	Error(s string)
}
```

Simple lexer can look like the following:

```go
type lexer struct {
	input  string
	pos    int
	result *Query
	err    error
}

func newLexer(input string) *lexer {
	return &lexer{input: input}
}

func (l *lexer) Lex(lval *yySymType) int {
	for l.pos < len(l.input) && (l.input[l.pos] == ' ' || l.input[l.pos] == '\t') {
		l.pos++
	}

	if l.pos >= len(l.input) {
		return 0 // EOF
	}

	if l.input[l.pos] == '=' {
		l.pos++
		return tokEQ
	}

	if l.input[l.pos] == '"' {
		l.pos++ // skip opening quote
		start := l.pos
		for l.pos < len(l.input) && l.input[l.pos] != '"' {
			l.pos++
		}
		lval.str = l.input[start:l.pos]
		l.pos++ // skip closing quote
		return tokSTRING
	}

	isAlpha := func(c byte) bool {
		return (c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z')
	}

	if isAlpha(l.input[l.pos]) {
		start := l.pos
		for l.pos < len(l.input) && (isAlpha(l.input[l.pos]) || l.input[l.pos] == '_') {
			l.pos++
		}
		word := l.input[start:l.pos]

		if strings.ToUpper(word) == "AND" {
			return tokAND
		}
		lval.str = word
		return tokIDENT
	}

	l.pos++
	return 0
}

func (l *lexer) Error(s string) {
	l.err = fmt.Errorf("parse error: %s", s)
}
```

The lexer recognizes tokens defined in the grammar.<br />
If a token carries data (string), it is stored in `lval.str`.


### Generate parser using goyacc {#generate-parser-using-goyacc}

Run `goyacc` to generate the parser (the grammar was stored in `query.y`):

```sh
goyacc -o parser.go query.y
```

For convenience, `Parse` function can be defined to call `yyParse` from the generated parser:

```go
func Parse(input string) (*Query, error) {
	l := newLexer(input)
	yyParse(l)
	if l.err != nil {
		return nil, l.err
	}
	return l.result, nil
}
```


### Generate SQL from AST {#generate-sql-from-ast}

Finally, traverse the AST to build SQL:

```go
func BuildSQL(q *Query, table string) (string, []any) {
	whereClause, args := toSQL(q.Condition)
	return fmt.Sprintf("SELECT * FROM %s WHERE %s", table, whereClause), args
}

func toSQL(c Condition) (string, []any) {
	switch v := c.(type) {
	case *FieldCondition:
		return fmt.Sprintf("%s = ?", v.Field), []any{v.Value}
	case *AndCondition:
		leftSQL, leftArgs := toSQL(v.Left)
		rightSQL, rightArgs := toSQL(v.Right)
		return fmt.Sprintf("(%s AND %s)", leftSQL, rightSQL), append(leftArgs, rightArgs...)
	default:
		return "", nil
	}
}
```

Putting it all together:

```go
query, _ := Parse(`status = "fail" AND name = "login_test"`)
sql, args := BuildSQL(query, "testcases")
// sql:  "SELECT * FROM testcases WHERE (status = ? AND name = ?)"
// args: ["fail", "login_test"]
```


## Conclusion {#conclusion}

Goyacc is likely sufficient for typical parsing needs. It is relatively easy to use, and (importantly) comes as a "standard" tool in Go ecosystem.

In several hundreds lines of code it is possible to implemeted simple DSL for querying database data.<br />
Although for such a simple task it is likely and overkill, the solution scales well for more complicated queries (see [Greener](https://github.com/cephei8/greener) codebase for more complex DSL implementation).
