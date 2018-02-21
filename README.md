# PHP RFC: Exception Coalesce Operator

- Version: 0.1
- Date: 2018-04-01
- Author: Félix Saparelli <felix@passcod.name>
- Status: Unofficial
- First published at: https://github.com/passcod/rfc-exception-coalesce-operator

## Introduction

Sometimes, it is desirable to assign the result of a computation to a variable, or an alternate value if the computation fails.
If the computation in question outputs a falsy value, the Ternary Operator `?:` or its short syntax can be used.
If the computation returns `null`, the Null Coalesce Operator `?:` can also be used, or even preferred in cases where falsy values should be considered valid.
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
TODO.

### New Constants
None.

### php.ini Defaults
None.

## Unaffected PHP Functionality

No existing functionality is affected by this, other than the new capabilities outlined in the proposal.

## Future Scope

TODO.

## Proposed Voting Choices

As this is a language change, a 2/3 majority is required.

## Patches and Tests

TODO.

## References

TODO. Rust `unwrap_or` and similar.
