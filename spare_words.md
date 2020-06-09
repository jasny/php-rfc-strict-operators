

## Introduction 



Making significant changes to the behavior of operators has severe
consequences to backward compatibility. Additionally, there is a
significant group of people who are in favor of the current method of
type juggling. Following the rationale of [PHP RFC: Scalar Type
Declarations](/rfc/scalar_type_hints_v5); an optional directive ensures
backward compatibility and allows people to choose the type checking
model that suits them best.






```
declare(strict_operators=1);

10 + '9 euro';  // TypeError("Unsupported type string for arithmetic operation")
10 == 'foo';    // TypeError("Unsupported type string for comparion operation")
'foo' == 'bar'; // TypeError("Unsupported type string for comparion operation")

10 + 2.2;       // 12.2
10 == 10.0;     // true
```