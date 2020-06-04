# PHP RFC: Strict operators directive

  - Version: 1.3
  - Date: 2020-06-04 (first version: 2019-05-25)
  - Author: Arnold Daniels, jasny@php.net
  - Status: Under Discussion
  - First Published at: <http://wiki.php.net/rfc/strict_operators>

## Introduction

PHP performs implicit type conversion for most operators. The rules of
conversion are complex, depending on the operator as well as on the type
and value of the operands. This can lead to surprising results, where a
statement seemingly contradicts itself. This RFC proposes a new
directive `strict_operators`, which limits the type juggling done by
operators and makes them throw a `TypeError` for unsupported types.

Making significant changes to the behavior of operators has severe
consequences to backward compatibility. Additionally, there is a
significant group of people who are in favor of the current method of
type juggling. Following the rationale of [PHP RFC: Scalar Type
Declarations](/rfc/scalar_type_hints_v5); an optional directive ensures
backward compatibility and allows people to choose the type checking
model that suits them best.

## Motivating examples

### Mixed type comparison

The meaning of comparison operators changes based on the type of each
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

  - Cast
      - silent
      - notice
      - warning
      - catchable error (fatal)
  - Operator specific (+ cast)
      - notice
      - warning
      - error
  - No operation

Please take a look at this [list of all combinations of operators and
operands](https://gist.github.com/jasny/bfd711844a8876f8206ed21357e2e2da).

## Proposal

By default, all PHP files are in weak type-checking mode for operators.
A new `declare()` directive is added, `strict_operators`, which takes
either `1` or `0`. If strict type-checking is not enabled, there will be
no change from the current behaviour of PHP. If strict type-checking is
enabled, the following stricter rules will be used;

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

```
declare(strict_operators=1);

10 + '9 euro';  // TypeError("Unsupported type string for arithmetic operation")
10 == 'foo';    // TypeError("Unsupported type string for comparion operation")
'foo' == 'bar'; // TypeError("Unsupported type string for comparion operation")

10 + 2.2;       // 12.2
10 == 10.0;     // true
```

### Comparison operators

Enabling strict operators does not affect the identical (`===`) and
not identical (`!==`) operators. Type conversion does not take place
for these two operators.

All other comparison operators (`==`, `!=`, `<`, `>`, `<=`, `>=`, `<=>`)
only support `int` or `float` operands when strict operators is enabled.
When used with a `bool`, `string`, `array`, `object`, or `resource`
operand, a `TypeError` is thrown.

### Arithmetic operators

Arithmetic operators only support `int` or `float` operands when strict
operators is enabled. Attempting to use an unsupported operand throws a
`TypeError`.

The `+` operator is still available for arrays as union operator,
requiring both values to be arrays.

### Incrementing/Decrementing operators

The incrementing/decrementing operators only support `int` or `float`
operands when strict operators is enabled. Attempting to use an
unsupported operand throws a `TypeError`.

### Bitwise Operators

Bitwise operators `&`, `|`, `^`, and `~` support `int` and `string`
operands when strict operators is enabled. The type of both operands
need to match. If the operands are of different types or when using
an unsupported operand, a `TypeError` is thrown.

The bitwise shift operators `>>` and `<<` only support `int` operands
when strict operators is enabled. Attempting to use an unsupported
operand throws a `TypeError`.

### String Operators

The concatenation operator `.` supports `null`, `int`, `float`,
`string`, and [stringable object](https://wiki.php.net/rfc/stringable)
operands when strict operators is enabled. If any of the operands is
a `bool`, `array`, `resource`, or non-stringable object, a `TypeError`
is thrown.

#### Variable parsing

When a string is specified in double quotes or with heredoc, variables
are parsed within it. Using the concatenation operator or double-quoted
string is considered interchangeable

### Logical Operators

The function of logical operators remains unchanged. All operands are
cast to booleans.

### Switch control structure

When strict operators is enabled, the `switch` statement will not
perform any type conversion. Comparison is done similar to the
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

## Backward Incompatible Changes

...

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

As this is a language change, a 2/3 majority is required.

The vote is a straight Yes/No vote for accepting the RFC and merging the
patch.

*Note: Voting on v1.0 of this RFC (targeting PHP 7.4) ended
prematurely.*
