# PHP RFC: Strict operators directive

  - Version: 1.4
  - Date: 2020-06-09 (first version: 2019-05-25)
  - Author: Arnold Daniels, jasny@php.net
  - Status: Under Discussion
  - First Published at: <http://wiki.php.net/rfc/strict_operators>

## Introduction

The rules PHP uses for type juggling with operators are complex, varying by operator as well as the types and values of the operands.

This can lead to surprising results, where a logical statement can have an illogical result. See the "Motivating examples" section below for details.
 
This RFC proposes a new directive `strict_operators`, which limits the type juggling done by
operators to avoid unexpected results.

## Proposal

Add a new  `declare()` directive `strict_operators`, which accepts either `0` or `1` as the value to indicate that the strict_operators mode should be disabled or enabled.

The default value for this directive is 0, i.e. strict_operators not enabled and there will be no change from the current behaviour of PHP i.e. by default, PHP files do not use the new strict_operators rules.

If strict_operators is enabled, the following stricter rules will be used;

  - Operators may perform type casting, but not type juggling:
    - Type casting is not based on the type of the other operand
    - Type casting is not based on the value of any of the operands
  - Operators will throw a `TypeError` for unsupported types

In case an operator can work with several (or all) types, the operands
need to match as no casting will be done by those operators.

The one exception is that [widening primitive
conversion](http://docs.oracle.com/javase/specs/jls/se7/html/jls-5.html#jls-5.1.2)
is allowed for `int` to `float`. When doing a operation with an `int`
and a `float`, the `int` will silently casted to a `float`.

When the strict_operators directive is enabled, the following rules will be used.

## Details for operands

This section details which types operands support, and what type juggling will be supported (if any) for them.

### Arithmetic operators

Arithmetic operators `+`, `-`, `*`, `/`, `%`, and `**` will only support `int` or `float` operands. Attempting to use an unsupported operand will throw a `TypeError`.

The `+` operator will still be available for arrays as union operator, requiring both values to be arrays.

### Incrementing/Decrementing operators

The incrementing/decrementing operators `++` and `--` will only support `int` or `float`operands. Attempting to use an
unsupported operand will throw a `TypeError`.

### Bitwise Operators

The bitwise operators `&`, `|`, `^`, and `~` will support `int` and `string`
operands . The type of both operands need to match. If the operands are of different types or when using
an unsupported operand, a `TypeError` will be thrown.

The bitwise shift operators `>>` and `<<` will only support `int` operands. Attempting to use an unsupported
operand will throw a `TypeError`.

### Comparison operators

Enabling the strict_operators directive will not affect the identical (`===`) and
not identical (`!==`) operators.

All other comparison operators (`==`, `!=`, `<`, `>`, `<=`, `>=`, `<=>`)
will only support `int` or `float` operands.

When used with a `bool`, `string`, `array`, `object`, or `resource`
operand, a `TypeError` will be thrown.


### String concatenation

The concatenation operator `.` will only support concatenating `null`, `int`, `float`,
`string`, and [stringable object](https://wiki.php.net/rfc/stringable)
operands.

If any of the operands is a `bool`, `array`, `resource`, or non-stringable object, a `TypeError`will be thrown.


#### String interpolation

When a string is specified in double quotes or with heredoc, variables
are parsed within it. The string interpolation is performed with the same rules as for string concatenation.

i.e. this code

```php
echo "He drank some $juice juice.";
```

has the same rules as using the string concatenation operator.

```php
echo "He drank some " . $juice . " juice.";
```

### Logical Operators

There a no changes to the logical operators `&&`, `||`, `!`, `and`, `or`, and `xor`.

### Ternary / Null Coalescing Operator

There a no changes to the ternary (`?:`) and null coalescing (`??`) operator.

### Switch control structure

The `switch` statement will not perform any type conversion. Comparison will be done similar to the
identical (`===`) operator, rather the equal (`==`) operator.

``` php
function match($value)
{
  switch ($value) {
    case ["foo" => 42, "bar" => 1]:
      return "foobar";
      break;
    case null:
      return "null";
      break;
    case 0:
      return "zero";
      break;
    case "1e1":
      return "1e1";
      break;
    case "10":
      return "10";
      break;
    default:
      throw new Exception("Unexpected value");
  }
}

match(["bar" => 1, "foo" => 42]); // "foobar"
match(0);                         // "zero"
match("10");                      // "10"
match("foo");                     // Exception("Unexpected value")
```

## Motivating examples

The following section demonstrates code that is written logical, but has illogical results.

### Mixed type comparison

The meaning of comparison operators currently change based on the type of each
operand. Strings are compared as byte sequence. If one of the operands
is an integer the operator performs a numeric comparison.

This allows for statements that defy mathematical logic and may be
experienced as unexpected behavior.

``` php
$a = '42';
$b = 10;
$c = '9 eur';

if (($a > $b) && ($b > $c) && ($c > $a)) {
   // Unexpected 
}
```

### Numeric string comparison

Non-strict comparison uses a "smart" comparison method that treats
strings as numbers if they are numeric. The meaning of the operator
changes based on the value of both operands.

This can lead to issues when numeric comparison is not expected, for
example between two hexidecimal values. The hexidecimal value is
instead interpreted as number with scientific notation.

```
$red = '990000';
$purple = '9900e2';

$red == $purple; // true
```

It may also cause issues with sorting, as the meaning of the
comparison operators differers based on the operands (similar to mixed
type comparison).

``` php
function sorted(array $arr) {
  usort($arr, function($x, $y) { return $x <=> $y; });
  return $arr;
}

sorted(['100', '5 eur', '62']); // ['100', '5 eur', '62']
sorted(['100', '62', '5 eur']); // ['5 eur', '62', '100']
sorted(['62', '100', '5 eur']); // ['62', '100', '5 eur']
```

### Array comparison

Using the `>`, `>=`, `<`, `<=` and `<=>` operators on arrays or objects
that don't have the same keys in the same order gives unexpected
results.

In the following example `$a` is both greater than and less than `$b`

``` php
$a = ['x' => 1, 'y' => 22];
$b = ['y' => 10, 'x' => 15];

$a > $b; // true
$a < $b; // true
```

The logic of relational operators other than `==`, `===`, `!=` and `!==`
has limited practical use.

### Strict vs non-strict comparison of arrays

Strict comparison requires that arrays have keys occurring in the same
order, while non-strict comparison allows out-of-order keys.

``` php
['a' => 'foo', 'b' => 'bar'] == ['b' => 'bar', 'a' => 0]; // true
```

To compare the values of two arrays in a strict way while not concerned
about the order, requires ordering the array by key prior to comparison.

### Switch control structure

The `switch` statement does a non-strict comparison. This can lead to
unexpected results;

``` php
function match($value)
{
  switch ($value) {
    case 2:
      return "double";
      break;
    case 1:
      return "single";
      break;
    case 0:
      return "none";
      break;
    default:
      throw new Exception("Unexpected value");
  }
}

match("foo"); // "none"
```

### Inconsistent behavior

Operators can do any of the following for unsupported operands

  - Cast (silent)
  - Cast with notice / warning
  - Cast with catchable error (fatal)
  - Operator specific notice / warning
  - Operator specific error (fatal)
  - No operation

Please take a look at this [list of all combinations of operators and
operands](https://gist.github.com/jasny/bfd711844a8876f8206ed21357e2e2da).
_TODO: run for to PHP8 from master branch_

## Backward Incompatible Changes

None known. As this RFC proposes a new directive, it should only affect code is new or updated to use the strict_operators directive.

## Proposed PHP Version

This is proposed for PHP 8.0.

## FAQ

[See FAQ.md](FAQ.md)

## Unaffected PHP Functionality

This RFC

  - Does not affect any functionality concerning explicit typecasting.
  - Is largely unaffected by other proposals like [PHP RFC: Saner string
    to number comparisons](/rfc/string_to_number_comparison) that focus
    on improving type juggling at the cost of breaking BC.

## Implementation

<https://github.com/php/php-src/pull/4375>

## Proposed Voting Choices

Primary vote: Accept the RFC and merge the patch? Yes/No. Requires a 2/3 majority.

Secondary vote: Should strict_operators affect the switch statement? Yes/No Requires a 50%+1 majority.

