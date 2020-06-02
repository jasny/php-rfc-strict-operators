# PHP RFC: Strict operators directive

  - Version: 1.2
  - Date: 2020-04-09 (first version: 2019-05-25)
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

If 

## Proposed PHP Version

This is proposed for PHP 8.0.

## FAQ

### What has been changed since the initial proposal?

  - Comparison operators `>`, `>=`, `<`, `<=` and `<=>` with string
    operands will throw a `TypeError`. [See
    motivation.](#why_aren_t_all_comparison_functions_available_for_strings)
  - Comparison operator `==` on array operands will always throw a
    `TypeError`.
  - Variable parsing in strings specified in double quotes and with
    heredoc, is also affected.
  - Concatenation operation (using `.`) on `null` does not throw a
    `TypeError`, but will cast `null` to an empty string.
  - There is no secondary vote for `switch`. If this RFC is accepted
    `strict_operators` will apply to `switch`.

### Why does

and \!= throw a TypeError instead of returning false? ==== In other
dynamically typed languages, like Python and Ruby, the `==` and `!=` do
a type check and always return `false` in case the type is different.
Throwing a `TypeError` is more common for statically typed languages
like Go.

The RFC tries to limit the cases where the behavior changes, rather than
that a `TypeError` is thrown, as much as possible. For instance `1 ==
true` and `null == 0` both evaluate to `true` without strict\_operators.
Having those statements be `false` based on the directive could lead to
unseen bugs.

### Why does the concatenation operator cast, but arithmetic operators don't?

The concatenation operator will cast integers, floats and *(when
`__toString` is defined)* objects to a string. This is a common use case
and operands are always typecasted to a string. Integers and floats
always have a proper string representation.

Arithmetic operators won't cast strings to an integer or float, because
not all strings can be properly represented as a number and a
`TypeError` must be thrown based on the operand type only, not the
value.

Both the concatenation operator and arithmetic operators throw a
`TypeError` for arrays, resources, and objects. Casting these to a
string or int/float doesn't give a proper representation of the value of
the operand.

Using a boolean or null as operand for both concatenation and arithmetic
operators also throws a `TypeError`. In most cases, the use of a boolean
or null indicates an error as many functions return `false` or `null` in
case of an error or when no result can be returned. This is different
from the function returning an empty string or `0`.
[`strpos`](https://php.net/strpos) is a well-known example.

### Will comparing a number to a numeric string work with strict operators?

No, this will throw a `TypeError`. Users that use string operators need
to explicitly typecast the string to an integer. If it concerns input
data, for instance from `$_POST`, it's recommended to use the [filter
functions](https://www.php.net/filter).

### Why aren't all comparison functions available for strings?

In many other languages, using `<`, `>`, `<=` and `=>` with string
operands performs an `strcmp` like operation. The `<=>` is even
described as strcmp as operators. Why do these operators throw an
exception with string operands when strict\_operators is enabled?

It's common to use these operators when both operands are numeric
strings. Having these statements return a different value based on the
directive, could lead to issues, especially when source code is
copy/pasted from an external codebase.

In case it concerns comparing text, it's better to use
`Collator::compare()`. For non-collation related strings, like date
strings, the `strcmp` function should be used.

Nikita Popov made the following argument:

\<blockquote\>Having `$str1 < $str2` perform a `strcmp()` style
comparison under strict\_operators is surprising. I think that overall
the use of lexicographical string comparisons is quite rare and should
be performed using an explicit `strcmp()` call. More likely than not,
writing `$str1 < $str2` is a bug and should generate a `TypeError`. Of
course, equality comparisons like `$str1 == $str2` should still work,
similar to the distinction you make for arrays.\</blockquote\>

### How can arrays be compared as unsorted hashmaps?

Arrays are sorted hashmaps in PHP but used as any type of collection
like unsorted maps, lists, and sets. Only comparing as unsorted hashmap
is supported using `==`, while other cases require the use of functions
like `sort()` and `array_values()`.

Strict comparison of arrays as unsorted hashmaps currently isn't
possible and requires sorting the array, prior to comparison.

With `==` unavailable when using `strict_operators`, sorting the array
would be the only option available.

Array functions might be added to compare arrays in different ways. But
that's outside the scope of this RFC.

### Why isn't is allowed to increment strings with strict\_operators?

Nikita Popov made the following argument:

\<blockquote\>String increment seems like a pretty niche use case, and I
believe that many people find the overflow behavior quite surprising. I
think it may be better to forbid string increment under
strict\_operators. \</blockquote\>

### Are built-in functions affected by strict\_operators?

No. Only operators (including the `case` statement)) in the source file
that has `declare(strict_operators=1)` are affected. Functions defined
elsewhere, including functions defined in extensions, are not affected.

Specifically `sort()` and `in_array()` will perform weak comparison as
usual and require the use of the `$sort_flags` and `$strict` arguments
to change the way the values are compared.

### Can relational operators be allowed for arrays?

If both arrays must have the same keys in the same order, using `<` or
`>` on two arrays can be useful. But, as shown in the examples, when
this is not the case, these type of comparisons will yield unexpected
results.

Throwing a `TypeError` only if the keys of the arrays don't match is not
in line with this RFC. The behavior of an operator should not depend on
the value of the operand, only on the type. Furthermore, a `TypeError`
would be misplaced here, as some arrays would be accepted but others
not, whereas a `TypeError` indicates no values of that type are
accepted.

### Are there cases where a statement doesn't throw a TypeError but yields a different result?

Yes, there are 3 cases where a comparison works differently in strict
operators mode.

``` 
  * Two numeric strings will be compared on equality as strings instead of numbers. Comparing two operands of the same type should always work. Under strict operators, the operation is only determined by the type and never by the value. Comparisons like ''"100" == "1e2"'' are unlikely to be intended and will return ''false'' with strict operators.
  * When comparing two objects no type juggling will be performed. This might result in ''false'' where ''true'' will be the result without strict operators.
  * With strict operators, the ''case'' in a ''switch'' statement will not do type conversion. It also doesn't compare numeric strings as numbers but as strings.
```

### Will this directive disable type juggling altogether?

No. Operators can still typecast under the given conditions. For
instance, the concatenation (`.`) operator will cast an integer or float
to a string, and boolean operators will cast any type of operand to a
boolean.

Typecasting is also done in other places: `if` and `while` statements
interpret expressions as boolean, and booleans and floats are cast to an
integer when used as array keys.

This RFC limits the scope to operators.

### Why is switch affected? It's not an operator.

Internally the `case` of a switch is handled as a comparison operator.
The issues with `case` are therefore similar (or even the same) to those
of comparison operators. The audience that `strict_operators` caters to,
likely want to get rid of this behavior completely.

## Unaffected PHP Functionality

This RFC

  - Does not affect any functionality concerning explicit typecasting.
  - Does not affect variable casting that occurs in (double-quoted)
    strings.
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
