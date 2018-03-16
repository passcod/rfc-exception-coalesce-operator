# PHP RFC: Exception Coalesce Operator

- Version: 0.1
- Date: 2018-04-01
- Author: Félix Saparelli <felix@passcod.name>
- Status: _[Unofficial](#unofficial)_
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

Extended and formal description TODO.

Alternative forms of the operator TODO.

### Why not use `@`?

The Error Supressing Operator could be used to approximate the behaviour of the proposed `???` operator:

```php
$result = (@compute()) ?? 'fallback';
```

However, the `@` operator is too broad in its effects, unclear in some cases, and overloads Null when it is used.

- Too broad: it does not only supress exceptions, but all errors,

### Example: Combined with itself

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
        $result = $fallback }
    }
}
```

### Example: Combined with other similar operators

The operator can be combined with the Null Coalesce and the Short Ternary in the usual way:

```php
$result = compute() ??? $alternate->property ?? $known ?: 'fallback';
```

If the succesful result of the left-hand side should be further checked, parentheses can be used to obtain the desired effect.
Consider these as equivalent:

```php
$result = (compute() ?? $fallback) ??? 'otherwise';

try {
    $result = compute() ?? $fallback;
} catch (Throwable $e) {
    $result = 'otherwise';
}
```

## Backward Incompatible Changes

None.

## Proposed PHP Version

Next major.

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

<details>
    <summary>Here is a hacky (very nasty) partial implementation, built on top of the `PHP-7.2.4` tag:</summary>

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
index 95d93903de..567e3955a1 100644
--- a/Zend/zend_compile.c
+++ b/Zend/zend_compile.c
@@ -7313,6 +7313,21 @@ void zend_compile_coalesce(znode *result, zend_ast *ast) /* {{{ */
 }
 /* }}} */
 
+void zend_compile_exception_coalesce(znode *result, zend_ast *ast) /* {{{ */
+{
+	zend_ast *expr_ast = ast->child[0];
+	zend_ast *default_ast = ast->child[1];
+
+	zend_ast *hack = zend_ast_create_ex(ZEND_AST_INCLUDE_OR_EVAL, ZEND_EVAL,
+		zend_ast_create_zval_from_str(
+			zend_ast_export("try { return ", expr_ast, "; } catch(\\Throwable $e) {}")));
+
+	ast->child[0] = hack;
+	zend_ast_destroy(expr_ast);
+
+	zend_compile_coalesce(result, ast);
+}
+/* }}} */
+
 void zend_compile_print(znode *result, zend_ast *ast) /* {{{ */
 {
 	zend_op *opline;
@@ -8266,6 +8281,9 @@ void zend_compile_expr(znode *result, zend_ast *ast) /* {{{ */
 		case ZEND_AST_COALESCE:
 			zend_compile_coalesce(result, ast);
 			return;
+		case ZEND_AST_EXCEPTION_COALESCE:
+			zend_compile_exception_coalesce(result, ast);
+			return;
 		case ZEND_AST_PRINT:
 			zend_compile_print(result, ast);
 			return;
diff --git a/Zend/zend_language_parser.y b/Zend/zend_language_parser.y
index 091d7f61e2..c0a0cece43 100644
--- a/Zend/zend_language_parser.y
+++ b/Zend/zend_language_parser.y
@@ -64,6 +64,7 @@ static YYSIZE_T zend_yytnamerr(char*, const char*);
 %left '=' T_PLUS_EQUAL T_MINUS_EQUAL T_MUL_EQUAL T_DIV_EQUAL T_CONCAT_EQUAL T_MOD_EQUAL T_AND_EQUAL T_OR_EQUAL T_XOR_EQUAL T_SL_EQUAL T_SR_EQUAL T_POW_EQUAL
 %left '?' ':'
 %right T_COALESCE
+%right T_EXCEPTION_COALESCE
 %left T_BOOLEAN_OR
 %left T_BOOLEAN_AND
 %left '|'
@@ -219,6 +220,7 @@ static YYSIZE_T zend_yytnamerr(char*, const char*);
 %token T_NS_SEPARATOR    "\\ (T_NS_SEPARATOR)"
 %token T_ELLIPSIS        "... (T_ELLIPSIS)"
 %token T_COALESCE        "?? (T_COALESCE)"
+%token T_EXCEPTION_COALESCE        "??? (T_EXCEPTION_COALESCE)"
 %token T_POW             "** (T_POW)"
 %token T_POW_EQUAL       "**= (T_POW_EQUAL)"
 
@@ -961,6 +963,8 @@ expr_without_variable:
 			{ $$ = zend_ast_create(ZEND_AST_CONDITIONAL, $1, NULL, $4); }
 	|	expr T_COALESCE expr
 			{ $$ = zend_ast_create(ZEND_AST_COALESCE, $1, $3); }
+	|	expr T_EXCEPTION_COALESCE expr
+			{ $$ = zend_ast_create(ZEND_AST_EXCEPTION_COALESCE, $1, $3); }
 	|	internal_functions_in_yacc { $$ = $1; }
 	|	T_INT_CAST expr		{ $$ = zend_ast_create_cast(IS_LONG, $2); }
 	|	T_DOUBLE_CAST expr	{ $$ = zend_ast_create_cast(IS_DOUBLE, $2); }
```

</details>

This implementation is only meant for demonstration and experimentation purposes.
It is also available at [passcod/php-src](https://github.com/passcod/php-src), branch `exception-coalesce`, where it is accompanied by some tests.
It has one important limitation: as it uses the `??` operator internally, these all evaluate to the RHS, while this proposal would have them evaluate to the LHS:

- `null ??? "wrong";`
- `function foo() { return null; } foo() ??? "wrong";`
- `function foo() {} foo() ??? "wrong";`

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

It also supports the Assignment variant mentioned in **Future Work**:

```ruby
result ||= begin
    compute
end
```

---

<sub><a name="unofficial">This is an April Fools' RFC! Besides not going through the correct process for PHP language changes, the `???` operator _syntax_ is ridiculous, although its _semantics_ are interesting.</a></sub>
