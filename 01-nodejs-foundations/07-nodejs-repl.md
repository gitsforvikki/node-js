# Lesson 7 — Node.js REPL

## What is REPL?

REPL stands for:

```text
Read
Evaluate
Print
Loop
```

It is an interactive Node.js shell where you can execute JavaScript immediately.

Run:

```bash
node
```

You may see:

```text
>
```

Now execute:

```js
2 + 3
```

Result:

```text
5
```

## Why REPL Matters

The REPL is excellent for:
- quickly testing JavaScript expressions
- checking Node APIs
- experimenting with functions
- debugging data transformations
- testing regular expressions
- checking module behavior

## Examples

```js
> process.version
> process.platform
> Math.max(10, 20, 5)
> [1, 2, 3].map(x => x * 2)
```

## Multi-line Functions

```js
> function add(a, b) {
... return a + b;
... }
```

Then:

```js
> add(10, 20)
30
```

## Useful REPL Commands

```text
.help
.exit
.clear
.break
.save
.load
```

Exit using:

```text
.exit
```

or press Ctrl+D.

## Special Variable

`_` contains the previous result.

Example:

```js
> 10 + 20
30
> _ * 2
60
```

## Industry Use

Suppose you are debugging a timestamp:

```js
> new Date(1760000000000).toISOString()
```

or testing a regex:

```js
> /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test("dev@example.com")
true
```

The REPL gives you fast feedback without creating a temporary file.

## Interview Importance

Low to medium.

You should know what REPL means and how it is useful, but it is not a deep interview area.

## Summary

```text
REPL = interactive Node.js execution environment
```

Use it as a fast experimentation and debugging tool.
