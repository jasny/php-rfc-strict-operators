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
