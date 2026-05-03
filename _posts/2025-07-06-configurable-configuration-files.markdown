---
layout: post
mathjax: true
comments: true
title:  "Configurable configuration files"
author: John Z. Li
date:   2025-07-07 10:00:00 +0800
categories: programming
tags: trading, configuration, software
---
Any nontrivial piece of software would require some kind of configuration, usually by loading a configuration file in program starting up.
But what is a configuration file exactly?
A configuration file contains information that is part of the initial state of the program.
There are two sides of a configuration file: data that is contained in the file, and how the data gets mapped into program state.

The data representation part is easy.
Any Data Descriptive Language (DDL) can serve the purpose.
The concept of DDLs is often conflated with the so-called Data Serialization Formats (DSF), such as JSON and XML.
But the difference between them is not the focus of this post.

**The part of mapping data to program state is much harder.
The main reason is that program state is not only about pure data,
but also how the data should be interpreted in the context of code.**
For example, the configuration `MaxOrderQty = 1'000'000` for a
trading system bears meaning that can only be understood in the context of trading.
The meaning of it can not be reduced to "a number with its value being 1'000'000".
To understand the effect of this configuration, one needs to know the answers to questions like
- Is the interpretation of the  quantity subject to lot size of a stock?
- What will happen if a client order exceeds the limit?
And usually another group of people other than developers are responsible for determining a sensible value
for the configurable item.

In other words, configuration files often bear meaningful information of business logic.
A good DDL alone does not make a good configuration file format.
A good configuration file format should make it straightforward to express business requirements in a human readable manner,
reducing the friction between data representation and data interpretation.
It should also have some basic features to detect miss-configuration (required configuration
items are missing), and mis-configuration (incorrect configuration by human error like typo).

The key to provide a better solution is to realize the duality between a host program
and its configuration. A configuration file configures a program.
On the other hand, by interpreting the configuration file in the context of business logic, the host program,
in a sense, assigns a concrete and consistent interpretation to the configuration file.
The complementary nature of the two leads to the following insight:
*A well designed configuration file format should make it possible for the host program to configure the configuration file itself.*
Thus, comes the title of the post: Configurable Configuration Files.

In the following part of this post, I will give an description of an experimental
design of a small language to be used for configuration files.

## Basics
It is often said that example is better than precept.
Let us start this with some simple examples:
```txt
// This is a comment line
qty = 2  // 2 is a string
text = OK // OK is also a string
greeting = Hello World    // quote not needed
reason = This is perfect  // quote still not needed
msg = "line1\nline2"      // must be quoted because escape sequence '\n'
msg_in_chinese = "简体中文" // UTF-8 characters must be quoted and UTF-8 encoded.
Fix_seperator = "\x01"  // '01' is hex value of the SOH character
```
- What is started with "//" is a comment line. Comments can also be trailing within the same line.
- Expression in the form of `a = b` is called an assignment. The left hand side is called an "identifier", and the right side is always a string.
- Identifiers are strings themselves that consist of `[A-Z][a-z][0-9] and _ (underscore)`, but not starting with a number.
- The right hand side of an assignment expression is a string with or without quoting (more details below).
- We can indicate the host program to interpret the right hand string as an instance of a certain type (more details below).
- Each assignment expression must occupy at least one line, that is, `a=b c=d` is illegal.
- Every identifier can only be assigned once. After that, its value becomes immutable.
  Re-assignment to an identifier that already has a value is disallowed. Assigning a new value to an existing name triggers an error.

Note that the mini-language supports types other than strings.
When we say the right hand side of an assignment is always a string, we don't imply
that the type of the identifier is always of string type.

In the expression `qty = 2`, `2` is not a number, but a string.
To specify its actual type in the host program, we can use decorators,
`@int qty = 2` says that this `qty` will be interpreted as a `int`. The `@int` is
called a decorator. The decorator can also be in separate line as
```
@int    // decorator is effective to the next assignment
qty = 2
```
This seems cumbersome at first glance, but consider `@uin64_t`, `@float`, `@double`, `@BigInt`
or any user defined types that can be used as decorators . This design actually removes
a main restriction that infects most of DDLs. They often only allow a limited number of built-in data types.
There are other benefits too.
For example, `@int qty = 3'000'000'000` will trigger an error if the host language is C++,
saying that the value has exceeded the range of type `int` in host language, while it will work just find in Python.
For more details about decorators, see the "Lists and Decorators" section.

If a string only consists of the following characters, quote is not needed.
- `[A-Z][a-z][0-9]` and `_` (underscore)
- `'`, apostrophe, (also known as the single quote symbol)
- `-`, the dash symbol
- `.`, the dot symbol
- `/`, the slash symbol (note that double slash starts comments)
- Spaces and Tabs, (a Tab is treated as 4 Spaces).
- `:`, the colon symbol
- `,`, the comma symbol
- `;`, the semicolon symbol
- `{}` two braces symbols
- `$` the dollar sign
The set of these characters are called Basic Characters. And strings that only
contain Basic Characters are called Basic Strings. Basic Strings do not need quoting
when appearing on the right side of assignment.

All the below are valid without any quoting
```
A = 2.3
B = 1'000'000
C = get-max-value
D = symbol: 1101.T, qty: 1000, price: 2355.56
E = {NewOrder {symbol: 4567.T, qty: 100, price: 12345.67}}
F = 1, 2, 3; 4, 5, 6; 7, 8, 9;
```
When a string starts with `{` and ends with `}`, it can span multiple lines, for example:
```
action = {build-and-install {
             ./configure
		     ./make --all
			 ./make install
             }
		 }
```
In this case, the newline character `\n` is part of the string.
Such as string is called a Multi-line Basic String.
A Multi-line Basic String can contain literal newline characters, but if the newline
character appear at the beginning or end of the string, it is ignored.
A Basic String, multi-line or single-line, the string starts from the first non-blank character
and ends with the last non-blank character.
For example, the `action` in above starts with `{` and ends with the last matching `}`.
And the below
```
x =           0 1 2 3            // comments

```
defines the value of `x` to `"0 1 2 3"`.

A strings starting with `{` and ending with `}` is also a list of strings, or just `List`.
A list of strings is a string, but not vice versa.
Before going into more details about lists, we need to talk about units.

## Units
The dollar symbol `$` bears special meaning, for example
```txt
$USD = American dollar
$HKD = Hong Kong dollar
$Lot = Hong Kong stock market lot
// with this defined
price1 = 1000 $HKD
price2 = 34.56 $USD
qty = 1000 $Lot
```
The special symbol `$` in `$USD` is not part of identifier.
Instead, it is a identifier modifier, or just modifier.
It means that the identifier `USD` is a unit of measure, and the right hand side
is a brief description of its meaning. A string ended with `$USD` is interpreted
as a numeric value of that unit. Combining it with decorators, we have
```
@int64_t $USD = American dollar
price = 134.5 $USD
// The price is in US dollar, and rounded to 64-bit signed integer
```
We can do multiplication on units to specify their relationship:
```
@double $USD = American dollar
@int64_t Inflation_factor = 10'000
// IUSD is USD inflacted by 10 thousand
// IUSD values are stored as 64-bit signed integer
@int64_t $IUSD = $USD * %(Inflation_factor)
Price = 90'000 $IUSD // The price is actually $9 in USD
```
This way, if necessary, we can avoid the trouble of having to dealing with numbers with decimal points.
Furthermore, quantities won't be just a number, units can be specified to remove ambiguity.
After all, the area of a house being `1000` is meaningless unless you know it is in square-foot or square-meter.
Note that the `%(id)` form means "get the value of the identifier". (When not ambiguous,
The parentheses behind `%` can be omitted. `%name` is legal if `name` is an identifier that has been assigned a value.
And the `*`, the multiplication operator is only used to specify the ratio between
two units. The following is not allowed:
```
// Note allowed
@int qty = 1000
@int price = 23
@int order_value = %(qty) * %(price)
```
The reason is that although both `qty` and `price` are integers, the mini configuration
language does not know what `*` means. To the mini language, `@int` is just one
of types that the host program has registered. It knows that the host program has
a way to get a value of type `int` from its string representation, it does not
assume any semantics of this type `int`. It does not even assume two instances of
`int` can be multiplied. This sounds unnecessarily restrictive, but there is good
reasons for it. Multiplying two integers can lead to integer overflow. Whether this error condition can happen,
or whether it needs to be handled when happening, or how it should be handled, are all up to the host program to decide.
The configuration file can't just assume anything.
To safety multiplying two integers, the host program can register a function to do that.
For more details of functions, see the Function section.

Now we go back to lists and decorators.

## Lists and Decorators
It is mentioned before that unquoted strings beginning with `{` and ending with `}` are also lists.
This is true, but not the whole truth. Actually, we don't differentiate between unquoted strings and lists.
Thus `a = 1 2 3 4` is equivalent to `a = {1 2 3 4}`.
- Any of `;`, ',', ':' and space or tabs (and newline character if the list spans multi-line) is treated as list separators.
- And they have precedence from high to low as `; --> , --> : --> space`.
With this in mind,
```txt
A = {
1, 2, 3;
4, 5, 6;
}
```
specifies a list with first element being `1, 2, 3` and the second
element being `4, 5, 6`, with each of which being a list of 3 elements.

There are important exceptions:
1. If a string is quoted, it is not interpreted as also a list. This means
   `a = "1, 2, 3; 4, 5, 6"` is not a list, but a pure string.
2. If a list contains at least one quoted string, the enclosing braces can not be omitted:
```
// curly braces can not be omitted here
my_list =  {
name: John Z. Li;
emial: "my_email@some_company.com";
}
```

So, with respect to strings and lists:
1. A quoted string is not a `List`. If a `String` has quoted sub-string, it is not a `List`.
2. Unquoted strings without any quoted sub-strings are both `List` and `String`.
3. A `List` can contain quoted strings as elements, but surrounding curly braces can not be omitted.
The implementation of the mini-language marks the difference with internal types `String` and `List`.
For example, `a = "hello world"` is a `String`, while `b= Hello world` is a `list`, which is also a `String`.

Here is an example:
```
@HKD = Hong Kong Dollar
@vec_of_orders
orders = {
symbol: 1101, price: 1234.56$HKD, qty: 1000, side: buy, TIF: day ;
symbol: 1202, price: 345.34$HKD, qty: 200, side: sell, TIF: IOC ;
symbol: 2343, price: 345.67$HKD, qty: 400, side: shortsell, TIF: FOK;
}
```
The decorator `vec_orf_orders` specifies this list will be parsed to some `vec_of_orders`
type defined in the native language.
The top level separator `;` specifies the list has 3 elements, with each denoting a client order.
The second top level separator `,` specifies data members of each order.
And the third separator `:` specifies name-to-value pairs.
When parsing, leading and trailing blank characters are stripped away. That is,
extra spaces and tabs have no effect.

With the above definition, we can use `order1 = orders[0]` to retrieve the first
order, which is also a `List`. We can also use `order1_price = orders[0, 2, 2]`
to get the order price of the first order, which is of type `$HKD`.

If you are accessing an non-existing element, `nil` will be returned. `nil` is of type `Nil`.
`nil` when comparing with any other value, leads to `false`, and only `true` when
comparing with itself. `true` and `false` are of type `Bool`.

There is also another special type `Any`, and there can be only one value `any` of the type.
The value of `any` denotes any value that is not conceptually `nil`,
for example, a non-empty string is not `nil`, a list that has at least one element is not `nil`.
`any` is only used in pattern matching to specify a default value when all other patterns do not match.
One can not assign `any` to an identifier. (More about this later.)

The question mark symbol `?` and `==` (equal) and `!=` can be used to check types and values.
1. The `?` operator is for `nil` check for an identifier, `?(x)` returns `true` if `x` has been assigned a non-nil value, otherwise returns `false`.
   When it is not ambiguous, the parentheses can be omitted.
2. The `==` operator does string comparison for `String` types, and element-wise string comparison for `List` types.
3. And `!=` is negation of `==`.

There are also `and`, `or` and `not` for logical combinations.
The built-in function `type()` can be used to get the type of an value.
And function `assert()` can be used to do sanity checks.
```
matrix = {1, 2, 3; 4, 5, 6; 7, 8, 9;}
assert(?matrix)  // matrix is not empty
assert(matrix[3,3] == nil) // valid index is 0 to 2
assert(matrix[0,0] = 1 and matrix[0,1] == 2)
```
Writing `assert` is a good way to do sanity checks to ensure basic correctness.

Now we circle back to decorators. A decorator is just a function that the host program
registers into the configuration parser. The function is written in host language,
so there is no performance loss. The function takes a `List` or `String` as input, and returns
an object of the target type defined also in the host language. So, after the configuration
phase, values with decorators will be in native object formats that are readily usable.
The configuration parser only needs to call the registered function to convert a list
to a native type.

## Functions
We have encountered several functions, (operators are just shorthand for functions).
- The `%` operator, that is the string substitution function.
- The `?` operator, that is the null check function.
- The `==` and `!=` operator, they are the list/string comparison functions.
- The `and`, `or` and `not` operators, that are logical combination functions.
- The `[]` operator, that is the index accessing function for `List`.
- The `assert` function.
- Decorators are also functions, but they map to types in the host language.

We will encounter more operators and functions later.
Besides decorators, users have the freedom to define their own functions.
These functions are defined in the host language that satisfies the below:
- The parameters are one or more `List` or `String` objects.
- The return type is a single `List` or `String` type.

For example, the host language is C++, and we want to implement a function that
checks whether a string is a sub-string of another string. Assuming the data type
used to store `List` or `String` is a `std::string`, the function signature is like
the below
```c++
// returns "true" if a is a sub-string of b
// otherwise "false"
std::string is_sub_str(std::string& a, std::string& b);
```

After registering this function into the configuration parser, it become available for
configuration. The configuration parser will do type casts, such as casting string
`"true"` to Boolean `true`.

The point is, the mini-language itself needs not to be Turing-complete. We only need
to make ti possible to enrich it with the help of the host language.

Note that if the host language is C++, a function here should not be a function template,
but just free-standing functions that can be registered with function pointers.

## The Broadcasting operator
It becomes tedious to construct `List` that contains names with a certain pattern.
Say, one client of a trading system has 10 trading sessions configured, and they
are referred to as client_session_1, client_session_2,..., client_session_10.
It is not wise to type all these repetitive names manually. We can use the broadcasting
operator here.

The broadcasting operator `#` is used as below:
```txt
session = User#_session   // session = User_session, same as string concatenation
user_session = session#1:3 // user_session = {session1, session2}
// #1:3 means 1 (inclusive) to 3 (exclusive)
account = account#1:2:5 // account = {account1, account3}
// #1:2:5, means starting from 1, with step 2, not equal or more than 5
```
The broadcasting operator takes three forms:
1. `#string`, which is the same as string concatenation.
2. `#N:M`, where `N, M` are numbers and `N < M`, is to specify a starting number and upper bound (exclusive).
3. `#N:S:M` where `N, M` are as above, and `S` is a positive number,
is to specify a starting number, a step, and a upper bound (exclusive).

The broadcasting operator can be applied to `List`.
When applied to `List`, the surrounding curly braces can not be omitted.
Otherwise, the left operand is treated as a `String`.

When broadcasting operator is applied to s `List`,
each element of the `List` has the operand actioned on.
{% raw %}
```txt
session = {Jane_street, Cube}#_session
// is equivalent to session = {Jane_street_session, Cube_session}
session = {Jane_street, Cube}#1:3
// is equivalent to
session = {{Jane_street1, Jane_street2}, { Cube1, Cube2}}
```
{% endraw %}

The broadcasting operator can be combined together to make more complicated examples
{% raw %}
```txt
session = {Jane_street, Cube}#_session_#1:3
// is equivalent to
session = {{Jane_street_session_1, Jane_street_session_2}, {Cube_session_1, Cube_session_2}}
```
{% endraw %}

## set and map
We use `[namae]` to introduce a set. Which is also known as enumerations, or a universe.
For a given name, only a value inside the set is a valid one. For example, an Exchange
only supports 3 types of orders, that is `Buy`, `Sell` and `ShortSell`, we can do this by
```txt`
// the right hand side is just a List
[side] = {Buy, Sell, ShortSell}
// If we receive an order that has a side that is not in the list, it is an error
[account] = {JS1, JS2, JS3} // When trading, the Account field must be in the list
```
By doing this, we can provide basic protection against mis-configuration, for example
```txt
[account] = {JS1, JS2, JS3}
[allocation_account] = {allo1, allo2}
// Below we define a mapping
[account] -> [allocation_account]
JS1       -> allo1
JS2       -> allo2
JS3       -> allo2
```
Here, we are mapping `account` to `allocation_account`, where `->` is the mapping
operator.
1. If we make a typo, for example, write `JS1` as `Js1`, the parser will find
that there is no `Js1` in set `account`, and thus raise an error.
2. The parser will also be able to check that for each element in `account`, there is a mapped value
   in `allocation_account`, so that we can't forget configuring some accounts.
3. The parser also checks that for each element in `allocation_account`, there is
   at least one element in `account` mapping to it.
4. The name of the map is implicitly defined as "account_to_allocation_account_map".
5. `account_to_allocation_account_map[JS1]` is the notation the get the mapped value.

If you do not want to check that the mapping is a surjection (point 3 of the above),
you only need to replace `[allocation_account]` with `allocation_account`.

If you do not want to constrain the possible values of keys to the set of `account`,
replace `[account]` with `account`. In this case, the parser treats the definition of
set `account` is only suggestive, and any non-nil value is a valid key.

When a set is big, it will become tedious to explicitly specify all the mapping.
We can use `any` to specify a general pattern.  This can be used to specify a default
value for non explicitly specified values.
```txt
[account] -> [allocation_account]
JS1       -> allo1
any       -> allo2
// JS1 is mapped to allo1, everything else is mapped to allo2`.

```
Whenever we do not have an exact match for a non-nil key, we check whether there is a mapping for `any`.
If so, the mapped value is the fallback default.

Sometimes, we want to map the `nil` value, that is the missing of a value is required,
we can use `nil` as a key.

We can also use `()` to group multiple keys together. `(a, b)` will match `a` or `b`,
The above can be written equivalently as
```txt
[account]  -> [allocation_account]
JS1        -> allo1
(JS2, JS3) -> allo2 // JS2 and JS3 maps to the same allocation account
```
The combination constructor can be used with broadcasting `#`, where
`(JS#2:4)` is the same as `(JS2, JS3)`.
The enclosing `()` signifies that multiple keys inside, and the broadcasting operator
constructs them.

Below is an example that restricts keys to a set but left mapped values non-restricted.
```txt
// mapped-to values are not restricted to a set
[exchange_session] = session#1:3
[exchange_session] -> @IpPort ip_address
// @IpPort specifies the type of the mapped-to values in host language
session1  -> {10.10.234.56, 5112} // ip and port
session2  -> {10.10.234.57, 5113}
```

Below is an example that neither restricts key nor value to a set.
```txt
@employee employee -> @string address
// Note both 'emplyee' and 'address' are not bounded in a set
{John, 0100} -> "NO. 1, Queen street"
{John, 2344} - > "NO. 2, Prince street"
// Key is name and employee number
// You can only get en employee's address if
// you know both his name and employee number
```

Finally, we can register a function to do the mapping, The registered function
should return string `"true"` for a match and string  `"false"` for a non-match,
and takes the return type of the decorator as input if a decorator is specified,
or `List` or a `String` as input type if no decorator is specified.
In other words, the function should be a predicate.
For example
```txt
@employee employee -> @string title
// all interns have the same title
// IS_INTERN is a registered function
IS_INTERN()        -> Junior Engineer
// ...

```

## Classification
Data classification is another usual task that we want to do in configuration files.
For example, for a given stock symbol, we want to know on which stock exchange the
symbol is traded. Say, we have two markets to support, let us call them TOI and TWE,
A supported symbol either belongs to TOI or TWE. We have the following configuration:
```txt
[market] = {TOI, TWE}
[symbol] = {s1, s2, ...., } // a lot of symbols
// We use the following syntax to introduce a classification:
[<symbol>] -> [market]
 s1        -> TOI
 s2        -> TWE
 ....
```
The syntax is similar to that of mapping, except key name is enclosed in a pair
of `<>`, that is the angle bracket. The name of it is implicitly defined as
"symbol_to_market_map".

A classification is a special kind of mapping.
The different between mapping and classification is that the value set of classification
is usually rather small. In our example above, we have thousands of stock symbols, but they are only
classified  to 2 possible markets. Also, a classification requires that there is
at least one key is mapped to a value in the value set. A non-used value in the value
set will raise an error. So, if we map all the symbols to one market, it is an error.

The reason that we differentiate between maps and classification is that the latter
should usually implemented as an attribute of the key type, while the former is usually
implemented as hash tables.

The beauty of classification is that we can use operator `in` to specify a subset,
for example,
```txt
[broker] = {KGI, YT}
[symbol]         ->        [broker]
in market[TOI]   ->         KGI
in_market[TWE]   ->         YT
```
Assuming that there are 2 brokers, all symbols that are classified to `TOI` are mapped to broker `KGI`,
and all symbols that are classified to `TWE` are mapped to broker `YT`.

The `in` operator can be combined with logical operator `and`, `or` `not` to combine multiple classifications.
Suppose that for all the symbols, we create another classification `[CFD] = {true, false}`,
we can express the idea that a symbol both belongs to `TSE` and is also a `CFD` symbol as
`in market[TSE] and in CFD[true]`.

