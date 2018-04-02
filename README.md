# PHP RFC: Exception Coalesce Operator

- Version: 0.3
- Date: 2018-04-01
- Author: Félix Saparelli <felix@passcod.name>
- Status: _[Pre-draft](#pre-draft)_
- First published at: https://github.com/passcod/rfc-exception-coalesce-operator

## Introduction

Sometimes, it is desirable to assign the result of a computation to a variable, or an alternate value if the computation fails.
If the computation in question outputs a falsy value, the Ternary Operator `?:` or its short syntax can be used.
If the computation returns `null`, the Null Coalesce Operator `??` can also be used, or even preferred in cases where falsy values should be considered valid.
But when the computation throws an exception, only the verbose `try` block is available.

This RFC introduces the Exception Coalesce Operator `???`, which evaluates and returns its left-hand side unless an exception is thrown in the course of the evaluation, and returns its right-hand side in that case.

## Proposal

Introduce the `???` operator, such that the following two blocks of code are equivalent:

```php
try {
    $result = computation();
} catch (Throwable $e) {
    $result = 'alternate';
}


$result = computation() ??? 'alternate';
```

### Why `???`

I do not yet have a good proposal for the right syntax.
I do not believe that `???` is good syntax, as it is confusing, but it is used here as a placeholder until a better one is found.

### Why not use `@`?

The Error Suppressing Operator could be used to approximate the behaviour of the proposed `???` operator:

```php
$result = (@compute()) ?? 'fallback';
```

However, the `@` operator is too broad in its effects, as it not only suppresses exceptions, but all errors, and can suppress errors in surprising ways for much wider effects than the user likely intended, e.g. with `@include('file.php');`.
The construction shown above also absorbs `null` values which might be wanted.
Consider:

```php
function save($record) {
    if (!Database::query("INSERT INTO table VALUES ({$record->id}, {$record->body})")) {
        throw new DatabaseException('Failed to save: ' . Database::last_error());
    }
}
```

This function would return `null` on success, and throw on error.
`(@save($record)) ?? fallback()` would therefore execute `fallback()` regardless.
The proposed operator brings a distinctive feature by not falling through if the value is `null` (as with `??`) or falsy (as with `?:`).

Like `@`, the `???` operator may be considered [a "stfu" operator](https://secure.php.net/manual/en/language.operators.errorcontrol.php#112900), because it does discard the _message_ of exceptions.
In library code, it should be used with care; in scripting, application, or even quick hacking scenarios, though, it can become a useful and powerful tool through its increased expressiveness.

### Example 1: Combined with itself

The operator can be used several times within a single statement, evaluating each subsequent left-hand side as needed.
This example shows the increasing returns this proposal enables, eliminating large, deeply-nested, hard-to-read structures and enabling more fluent patterns.
Consider these as equivalent:

```php
$result = optionA() ??? optionB() ??? $fallback;

try {
    $result = optionA();
} catch (Throwable $e) {
    try {
        $result = optionB();
    } catch (Throwable $e) {
        $result = $fallback;
    }
}
```

### Example 2: Combined with other similar operators

The operator can be combined with the Null Coalesce and the Short Ternary in the usual way:

```php
$result = compute() ??? $alternate->property ?? $known ?: 'fallback';
```

If the successful result of the left-hand side should be further checked, parentheses can be used to obtain the desired effect.
Consider these as equivalent:

```php
$result = (compute() ?? $fallback) ??? 'otherwise';

try {
    $result = compute() ?? $fallback;
} catch (Throwable $e) {
    $result = 'otherwise';
}
```

### Example 3: non-existent functions

The operator doesn't only substitute scenarios expressed with try...catch.
It is sometimes useful to check for existence of a function, and try something else if it doesn't exist:

```php
$size = getimagesize($filename) ??? user_defined_imagesize($filename);


if (function_exists('getimagesize')) {
    $size = getimagesize ($filename);
} else {
    $size = user_defined_imagesize($filename);
}

// or:
$size = function_exists('getimagesize') ? getimagesize($filename) : user_defined_imagesize($filename);
```

The `???` operator can be chained, which is useful to add additional fallbacks:

```php
$result = foo() ??? bar() ??? baz() ??? 'oh no';
```

Finally, in the case of dynamic function or method selection, often used in routing mechanisms, the proposal offers some brevity:

```php
$name = 'dragons';
echo $name() ??? not_found();
```

### Example 4: conditions with deep checking

When checking the validity of input data, it is sometimes necessary to check if several deep properties exist together.
The `??` operator has greatly helped to make this kind of operation more streamlined, but the `???` operator improves ergonomics further.
Consider:

```php
if (($data->name['first-name'] ?? false) && ($data->name['last-name'] ?? false))
```

This could be more clearly written:

```php
if (($data->name['first-name'] && $data->name['last-name']) ??? false)
```

It is more visible with even more complex conditions, such as:

```php
if ((
    ($data->name['first-name'] && $data->name['last-name'])
    || $data->name['full-name']
    || ($store->overrides()['name'] && $options->can->override->name)
) ??? false)
```

## Backward Incompatible Changes

None.

## Proposed PHP Version

7.x.

## RFC Impact

### To SAPIs
None.

### To Existing Extensions
None.

### To Opcache
Unsure.

### New Constants
None.

### php.ini Defaults
None.

## Unaffected PHP Functionality

No existing functionality is affected by this, other than the new capabilities outlined in the proposal.

## Future Scope

Akin to the [Null Coalesce Equal Operator](https://wiki.php.net/rfc/null_coalesce_equal_operator), a future proposal could explore either an Assignment version of this operator: `$result ???= compute();`, or a unary suffix version: `$result = compute()???;`.

## Proposed Voting Choices

As this is a language change, a 2/3 majority is required.

## Patches and Tests

Here is a (very) hacky implementation, built on top of the `PHP-7.2.4` tag:

```diff
diff --git a/Zend/zend_ast.c b/Zend/zend_ast.c
index c0fdf48cba..e961320747 100644
--- a/Zend/zend_ast.c
+++ b/Zend/zend_ast.c
@@ -573,6 +573,7 @@ ZEND_API void zend_ast_apply(zend_ast *ast, zend_ast_apply_func fn) {
  *   90     right           = += -= *= /= .= %= &= |= ^= <<= >>= **=
  *  100     left            ? :
  *  110     right           ??
+ *  115     right           ???
  *  120     left            ||
  *  130     left            &&
  *  140     left            |
@@ -1426,6 +1427,7 @@ simple_list:
 		case ZEND_AST_YIELD_FROM:
 			PREFIX_OP("yield from ", 85, 86);
 		case ZEND_AST_COALESCE: BINARY_OP(" ?? ", 110, 111, 110);
+		case ZEND_AST_EXCEPTION_COALESCE: BINARY_OP(" ??? ", 110, 111, 110);
 		case ZEND_AST_STATIC:
 			smart_str_appends(str, "static $");
 			zend_ast_export_name(str, ast->child[0], 0, indent);
diff --git a/Zend/zend_ast.h b/Zend/zend_ast.h
index 08a8ab57f4..af512d6beb 100644
--- a/Zend/zend_ast.h
+++ b/Zend/zend_ast.h
@@ -115,6 +115,7 @@ enum _zend_ast_kind {
 	ZEND_AST_INSTANCEOF,
 	ZEND_AST_YIELD,
 	ZEND_AST_COALESCE,
+	ZEND_AST_EXCEPTION_COALESCE,
 
 	ZEND_AST_STATIC,
 	ZEND_AST_WHILE,
diff --git a/Zend/zend_compile.c b/Zend/zend_compile.c
index 95d93903de..821b02ff98 100644
--- a/Zend/zend_compile.c
+++ b/Zend/zend_compile.c
@@ -7313,6 +7313,41 @@ void zend_compile_coalesce(znode *result, zend_ast *ast) /* {{{ */
 }
 /* }}} */
 
+void zend_compile_exception_coalesce(znode *result, zend_ast *ast) /* {{{ */
+{
+	zend_ast *lhs = ast->child[0];
+	zend_ast *rhs = ast->child[1];
+
+	/* eval(try { $_exco = (LHS); } catch (\Throwable $e) { $_exco = id; })
+	=> eval without return always returns null... */
+	const char *lhs_prefix = "try { $_exco = (";
+	const char *lhs_suffix = "); } catch (\\Throwable $e) { $_exco = 892638795; }";
+
+	zend_string *lhs_eval_str = zend_ast_export(lhs_prefix, lhs, lhs_suffix);
+	zend_ast *lhs_eval = zend_ast_create_ex(ZEND_AST_INCLUDE_OR_EVAL, ZEND_EVAL,
+		zend_ast_create_zval_from_str(lhs_eval_str));
+
+	zend_ast_destroy(lhs);
+	// printf("lhs eval: %s\n", ZSTR_VAL(lhs_eval_str));
+
+	/* eval(return ($_exco === id) ? (RHS) : $_exco;)
+	=> ...after a ?? this will always execute AND be the final value! */
+	const char *rhs_prefix = "return ($_exco === 892638795) ? (";
+	const char *rhs_suffix = ") : $_exco;";
+
+	zend_string *rhs_eval_str = zend_ast_export(rhs_prefix, rhs, rhs_suffix);
+	zend_ast *rhs_eval = zend_ast_create_ex(ZEND_AST_INCLUDE_OR_EVAL, ZEND_EVAL,
+		zend_ast_create_zval_from_str(rhs_eval_str));
+
+	zend_ast_destroy(rhs);
+	// printf("rhs eval: %s\n", ZSTR_VAL(rhs_eval_str));
+
+	ast->child[0] = lhs_eval;
+	ast->child[1] = rhs_eval;
+	zend_compile_coalesce(result, ast);
+}
+/* }}} */
+
 void zend_compile_print(znode *result, zend_ast *ast) /* {{{ */
 {
 	zend_op *opline;
@@ -8266,6 +8301,9 @@ void zend_compile_expr(znode *result, zend_ast *ast) /* {{{ */
 		case ZEND_AST_COALESCE:
 			zend_compile_coalesce(result, ast);
 			return;
+		case ZEND_AST_EXCEPTION_COALESCE:
+			zend_compile_exception_coalesce(result, ast);
+			return;
 		case ZEND_AST_PRINT:
 			zend_compile_print(result, ast);
 			return;
diff --git a/Zend/zend_language_scanner.l b/Zend/zend_language_scanner.l
index 837df416e2..7223405fd5 100644
--- a/Zend/zend_language_scanner.l
+++ b/Zend/zend_language_scanner.l
@@ -1326,6 +1326,10 @@ NEWLINE ("\r"|"\n"|"\r\n")
 	RETURN_TOKEN(T_COALESCE);
 }
 
+<ST_IN_SCRIPTING>"???" {
+	RETURN_TOKEN(T_EXCEPTION_COALESCE);
+}
+
 <ST_IN_SCRIPTING>"new" {
 	RETURN_TOKEN(T_NEW);
 }
```

This implementation is only meant for demonstration and experimentation purposes.
It is also available at [passcod/php-src](https://github.com/passcod/php-src), branch `exception-coalesce`, where it is accompanied by some tests.
It has some minor limitations because of its internals:

- The local variable named `$_exco` will be overwritten.
- If the LHS evaluates to the magic value `892638795`, the RHS will be erroneously returned.
- If the LHS contains an IIFE invoked using `()` (`(function () {})()`) and not `call_user_func`, the pretty printer will generate code that cannot be re-parsed: `function () {}()`, and the construct will throw a Parse Error. There may be other edge cases where that occurs.

## References

[Rust](https://rust-lang.org) doesn't have exceptions, but its `Result` type has the [`unwrap_or`](https://doc.rust-lang.org/std/result/enum.Result.html#method.unwrap_or) family of methods, which behave similarly to this operator:

```rust
let result = compute().unwrap_or("fallback");
let another = compute().unwrap_or_else(|| fallback());
```

Ruby is an expression-oriented language, which makes the kind of patterns this operator enables also fairly common:

```ruby
result = begin
    compute
end || 'fallback'
```

It also supports the Assignment variant mentioned in **Future Scope**:

```ruby
result ||= begin
    compute
end
```

---

<sub><a name="pre-draft" href="https://blog.passcod.name/2018/apr/10/monthly-update#april-fools">T</a>his proposal has not yet been submitted for consideration to the PHP mailing lists. It was developed here and in this form to tease out its details and uses, and present a better case than the original ridiculous proposal.</sub>
