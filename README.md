toml11
======

[![Build Status on TravisCI](https://travis-ci.org/ToruNiina/toml11.svg?branch=master)](https://travis-ci.org/ToruNiina/toml11)
[![Build status on Appveyor](https://ci.appveyor.com/api/projects/status/m2n08a926asvg5mg/branch/master?svg=true)](https://ci.appveyor.com/project/ToruNiina/toml11/branch/master)
[![Build status on CircleCI](https://circleci.com/gh/ToruNiina/toml11/tree/master.svg?style=svg)](https://circleci.com/gh/ToruNiina/toml11/tree/master)
[![Version](https://img.shields.io/github/release/ToruNiina/toml11.svg?style=flat)](https://github.com/ToruNiina/toml11/releases)
[![License](https://img.shields.io/github/license/ToruNiina/toml11.svg?style=flat)](LICENSE)
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.1209136.svg)](https://doi.org/10.5281/zenodo.1209136)

toml11 is a C++11 header-only toml parser/encoder depending only on C++ standard library.

compatible to the latest version of
[TOML v0.5.0](https://github.com/toml-lang/toml/blob/master/versions/en/toml-v0.5.0.md)
after version 2.0.0.

It passes [the language agnostic test suite for TOML parsers by BurntSushi](https://github.com/BurntSushi/toml-test).
Not only the test suite itself, a TOML reader/encoder also runs on [CircleCI](https://circleci.com/gh/ToruNiina/toml11).
You can see the error messages about invalid files and serialization results of valid files at
[CircleCI](https://circleci.com/gh/ToruNiina/toml11).

## Example

```cpp
#include <toml.hpp>
#include <iostream>

int main()
{
    const auto data = toml::parse("example.toml");

    // title = "an example toml file"
    std::string title = toml::find<std::string>(data, "title");
    std::cout << "the title is " << title << std::endl;

    // nums = [1, 2, 3, 4, 5]
    std::vector<int> nums  = toml::find<std::vector<int>>(data, "nums");
    std::cout << "the length of `nums` is" << nums.size() << std::endl;

    return 0;
}
```

## Table of Contents

- [Integration](#integration)
- [Decoding a toml file](#decoding-a-toml-file)
  - [In the case of syntax error](#in-the-case-of-syntax-error)
  - [Invalid UTF-8 Codepoints](#invalid-utf-8-codepoints)
- [Finding a toml value](#finding-a-toml-value)
  - [Finding a value in a table](#finding-a-value-in-a-table)
  - [In case of error](#in-case-of-error)
  - [Dotted keys](#dotted-keys)
- [Casting a toml value](#casting-a-toml-value)
- [Checking value type](#checking-value-type)
- [More about conversion](#more-about-conversion)
  - [Converting an array](#converting-an-array)
  - [Converting a table](#converting-a-table)
  - [Getting an array of tables](#getting-an-array-of-tables)
  - [Cost of conversion](#cost-of-conversion)
  - [Converting datetime and its variants](#converting-datetime-and-its-variants)
- [Getting with a fallback](#getting-with-a-fallback)
- [Expecting conversion](#expecting-conversion)
- [Visiting a toml::value](#visiting-a-tomlvalue)
- [Constructing a toml::value](#constructing-a-tomlvalue)
- [Preserving Comments](#preserving-comments)
- [Customizing containers](#customizing-containers)
- [TOML literal](#toml-literal)
- [Conversion between toml value and arbitrary types](#conversion-between-toml-value-and-arbitrary-types)
- [Formatting user-defined error messages](#formatting-user-defined-error-messages)
- [Serializing TOML data](#serializing-toml-data)
- [Underlying types](#underlying-types)
- [Unreleased TOML features](#unreleased-toml-features)
- [Breaking Changes from v2](#breaking-changes-from-v2)
- [Running Tests](#running-tests)
- [Contributors](#contributors)
- [Licensing Terms](#licensing-terms)

## Integration

Just include the file after adding it to the include path.

```cpp
#include <toml11/toml.hpp> // that's all! now you can use it.
#include <iostream>

int main()
{
    const auto data  = toml::parse("example.toml");
    const auto title = toml::find<std::string>(data, "title");
    std::cout << "the title is " << title << std::endl;
    return 0;
}
```

The convenient way is to add this repository as a git-submodule or to install
it in your system by CMake.

## Decoding a toml file

To parse a toml file, the only thing you have to do is
to pass a filename to the `toml::parse` function.

```cpp
const std::string fname("sample.toml");
const toml::value data = toml::parse(fname);
```

If it encounters an error while opening a file, it will throw `std::runtime_error`.

You can also pass a `std::istream` to the  `toml::parse` function.
To show a filename in an error message, however, it is recommended to pass the
filename with the stream.

```cpp
std::ifstream ifs("sample.toml", std::ios_base::binary);
assert(ifs.good());
const auto data = toml::parse(ifs, /*optional -> */ "sample.toml");
```

**Note**: When you are **on Windows, open a file in binary mode**.
If a file is opened in text-mode, CRLF ("\r\n") will automatically be
converted to LF ("\n") and this causes inconsistency between file size
and the contents that would be read. This causes weird error.

### In the case of syntax error

If there is a syntax error in a toml file, `toml::parse` will throw
`toml::syntax_error` that inherits `std::exception`.

toml11 has clean and informative error messages inspired by Rust and
it looks like the following.

```console
terminate called after throwing an instance of 'toml::syntax_error'
  what():  [error] toml::parse_table: invalid line format # error description
 --> example.toml                                         # file name
 3 | a = 42 = true                                        # line num and content
   |        ^------ expected newline, but got '='.        # error reason
```

If you (mistakenly) duplicate tables and got an error, it is helpful to see
where they are. toml11 shows both at the same time like the following.

```console
terminate called after throwing an instance of 'toml::syntax_error'
  what():  [error] toml::insert_value: table ("table") already exists.
 --> duplicate-table.toml
 1 | [table]
   | ~~~~~~~ table already exists here
 ...
 3 | [table]
   | ~~~~~~~ table defined twice
```

When toml11 encounters a malformed value, it tries to detect what type it is.
Then it shows hints to fix the format. An error message while reading one of
the malformed files in [the language agnostic test suite](https://github.com/BurntSushi/toml-test).
is shown below.

```console
what(): [error] bad time: should be HH:MM:SS.subsec
 --> ./datetime-malformed-no-secs.toml
 1 | no-secs = 1987-07-05T17:45Z
   |                     ^------- HH:MM:SS.subsec
   | 
Hint: pass: 1979-05-27T07:32:00, 1979-05-27 07:32:00.999999
Hint: fail: 1979-05-27T7:32:00, 1979-05-27 17:32
```

You can find other examples in a job named `output_result` on
[CircleCI](https://circleci.com/gh/ToruNiina/toml11).

Since the error message generation is generally a difficult task, the current
status is not ideal. If you encounter a weird error message, please let us know
and contribute to improve the quality!

### Invalid UTF-8 codepoints

It throws `syntax_error` if a value of an escape sequence
representing unicode character is not a valid UTF-8 codepoint.

```console
  what():  [error] toml::read_utf8_codepoint: input codepoint is too large.
 --> utf8.toml
 1 | exceeds_unicode = "\U0011FFFF example"
   |                              ^--------- should be in [0x00..0x10FFFF]
```

## Finding a toml value

After parsing successfully, you can obtain the values from the result of
`toml::parse` using `toml::find` function.

```toml
# sample.toml
answer  = 42
pi      = 3.14
numbers = [1,2,3]
time    = 1979-05-27T07:32:00Z
```

``` cpp
const auto data      = toml::parse("sample.toml");
const auto answer    = toml::find<std::int64_t    >(data, "answer");
const auto pi        = toml::find<double          >(data, "pi");
const auto numbers   = toml::find<std::vector<int>>(data, "numbers");
const auto timepoint = toml::find<std::chrono::system_clock::time_point>(data, "time");
```

By default, `toml::find` returns a `toml::value`.

```cpp
const toml::value& answer = toml::find(data, "answer");
```

When you pass an exact TOML type that does not require type conversion,
`toml::find` returns a reference without copying the value.

```cpp
const auto  data   = toml::parse("sample.toml");
const auto& answer = toml::find<toml::integer>(data, "answer");
```

If the specified type requires conversion, you can't take a reference to the value.
See also [underlying types](#underlying-types).

**NOTE**: For some technical reason, automatic conversion between `integer` and
`floating` is not supported. If you want to get a floating value even if a value
has integer value, you need to convert it manually after obtaining a value,
like the followings.

```cpp
const auto vx = toml::find(data, "x");
double x = vx.is_floating() ? vx.as_floating(std::nothrow) :
           static_cast<double>(vx.as_integer()); // it throws if vx is neither
                                                 // floating nor integer.
```

### Finding a value in a table

There are several way to get a value defined in a table.
First, you can get a table as a normal value and find a value from the table.

```toml
[fruit]
name = "apple"
[fruit.physical]
color = "red"
shape = "round"
```

``` cpp
const auto  data  = toml::parse("fruit.toml");
const auto& fruit = toml::find(data, "fruit");
const auto  name  = toml::find<std::string>(fruit, "apple");

const auto& physical = toml::find(fruit, "physical");
const auto  color    = toml::find<std::string>(fruit, "color");
const auto  shape    = toml::find<std::string>(fruit, "shape");
```

Here, variable `fruit` is a `toml::value` and can be used as the first argument
of `toml::find`.

Second, you can pass as many arguments as the number of subtables to `toml::find`.

```cpp
const auto data  = toml::parse("fruit.toml");
const auto color = toml::find<std::string>(data, "fruit", "physical", "color");
const auto shape = toml::find<std::string>(data, "fruit", "physical", "shape");
```

### In case of error

If the value does not exist, `toml::find` throws an error with the location of
the table.

```console
terminate called after throwing an instance of 'std::out_of_range'
  what():  [error] key "answer" not found
 --> example.toml
 6 | [tab]
   | ~~~~~ in this table
```

**Note**: It is recommended to find a table as `toml::value` because it has much information
compared to `toml::table`, which is an alias of
`std::unordered_map<std::string, toml::value>`. Since `toml::table` does not have
any information about toml file, such as where the table was defined in the file.

----

If the specified type differs from the actual value contained, it throws
`toml::type_error` that inherits `std::exception`.

Similar to the case of syntax error, toml11 also displays clean error messages.
The error message when you choose `int` to get `string` value would be like this.

```console
terminate called after throwing an instance of 'toml::type_error'
  what():  [error] toml::value bad_cast to integer
 --> example.toml
 3 | title = "TOML Example"
   |         ~~~~~~~~~~~~~~ the actual type is string
```

**NOTE**: In order to show this kind of error message, all the toml values have
a pointer to represent its range in a file. The entire contents of a file is
shared by `toml::value`s and remains on the heap memory. It is recommended to
destruct all the `toml::value` classes after configuring your application
if you have a large TOML file compared to the memory resource.

### Dotted keys

TOML v0.5.0 has a new feature named "dotted keys".
You can chain keys to represent the structure of the data.

```toml
physical.color = "orange"
physical.shape = "round"
```

This is equivalent to the following.

```toml
[physical]
color = "orange"
shape = "round"
```

You can get both of the above tables with the same c++ code.

```cpp
const auto physical = toml::find(data, "physical");
const auto color    = toml::find<std::string>(physical, "color");
```

The following code does not work for the above toml file.

```cpp
// XXX this does not work!
const auto color = toml::find<std::string>(data, "physical.color");
```

The above code works with the following toml file.

```toml
"physical.color" = "orange"
# equivalent to {"physical.color": "orange"},
# NOT {"physical": {"color": "orange"}}.
```

## Casting a toml value

### `toml::get`

`toml::parse` returns `toml::value`. `toml::value` is a union type that can
contain one of the following types.

- `toml::boolean` (`bool`)
- `toml::integer` (`std::int64_t`)
- `toml::floating` (`double`)
- `toml::string` (a type convertible to std::string)
- `toml::local_date`
- `toml::local_time`
- `toml::local_datetime`
- `toml::offset_datetime`
- `toml::array` (by default, `std::vector<toml::value>`)
  - It depends. See [customizing containers](#customizing-containers) for detail.
- `toml::table` (by default, `std::unordered_map<toml::key, toml::value>`)
  - It depends. See [customizing containers](#customizing-containers) for detail.

To get a value inside, you can use `toml::get<T>()`. The usage is the same as
`toml::find<T>` (actually, `toml::find` internally uses `toml::get` after casting
a value to `toml::table`).

``` cpp
const toml::value  data    = toml::parse("sample.toml");
const toml::value  answer_ = toml::get<toml::table >(data).at("answer");
const std::int64_t answer  = toml::get<std::int64_t>(answer_);
```

When you pass an exact TOML type that does not require type conversion,
`toml::get` returns a reference through which you can modify the content
(if the `toml::value` is `const`, it returns `const` reference).

```cpp
toml::value   data    = toml::parse("sample.toml");
toml::value   answer_ = toml::get<toml::table >(data).at("answer");
toml::integer& answer = toml::get<toml::integer>(answer_);
answer = 6 * 9; // write to data.answer. now `answer_` contains 54.
```

If the specified type requires conversion, you can't take a reference to the value.
See also [underlying types](#underlying-types).

It also throws a `toml::type_error` if the type differs.

### `as_xxx`

You can also use a member function to cast a value.

```cpp
const std::int64_t answer = data.as_table().at("answer").as_integer();
```

It also throws a `toml::type_error` if the type differs. If you are sure that
the value `v` contains a value of the specified type, you can suppress checking
by passing `std::nothrow`.

```cpp
const auto& answer = data.as_table().at("answer");
if(answer.is_integer() && answer.as_integer(std::nothrow) == 42)
{
    std::cout << "value is 42" << std::endl;
}
```

If `std::nothrow` is passed, the functions are marked as noexcept.

The full list of the functions is below.

```cpp
namespace toml {
class value {
    // ...
    const boolean&         as_boolean()         const&;
    const integer&         as_integer()         const&;
    const floating&        as_floating()        const&;
    const string&          as_string()          const&;
    const offset_datetime& as_offset_datetime() const&;
    const local_datetime&  as_local_datetime()  const&;
    const local_date&      as_local_date()      const&;
    const local_time&      as_local_time()      const&;
    const array&           as_array()           const&;
    const table&           as_table()           const&;
    // --------------------------------------------------------
    // non-const version
    boolean&               as_boolean()         &;
    // ditto...
    // --------------------------------------------------------
    // rvalue version
    boolean&&              as_boolean()         &&;
    // ditto...

    // --------------------------------------------------------
    // noexcept versions ...
    const boolean&         as_boolean(const std::nothrow_t&) const& noexcept;
    boolean&               as_boolean(const std::nothrow_t&) &      noexcept;
    boolean&&              as_boolean(const std::nothrow_t&) &&     noexcept;
    // ditto...
};
} // toml
```

### `at()`

You can access to the element of a table and an array by `toml::basic_value::at`.

```cpp
const toml::value v{1,2,3,4,5};
std::cout << v.at(2).as_integer() << std::endl; // 3

const toml::value v{{"foo", 42}, {"bar", 3.14}};
std::cout << v.at("foo").as_integer() << std::endl; // 42
```

If an invalid key (integer for a table, string for an array), it throws
`toml::type_error` for the conversion. If the provided key is out-of-range,
it throws `std::out_of_range`.

Note that, although `std::string` has `at()` member function, `toml::value::at`
throws if the contained type is a string. Because `std::string` does not
contain `toml::value`.

## Checking value type

You can check the type of a value by `is_xxx` function.

```cpp
const toml::value v = /* ... */;
if(v.is_integer())
{
    std::cout << "value is an integer" << std::endl;
}
```

The complete list of the functions is below.

```cpp
namespace toml {
class value {
    // ...
    bool is_boolean()         const noexcept;
    bool is_integer()         const noexcept;
    bool is_floating()        const noexcept;
    bool is_string()          const noexcept;
    bool is_offset_datetime() const noexcept;
    bool is_local_datetime()  const noexcept;
    bool is_local_date()      const noexcept;
    bool is_local_time()      const noexcept;
    bool is_array()           const noexcept;
    bool is_table()           const noexcept;
    bool is_uninitialized()   const noexcept;
    // ...
};
} // toml
```

Also, you can get `enum class value_t` from `toml::value::type()`.

```cpp
switch(data.at("something").type())
{
    case toml::value_t::integer:  /*do some stuff*/ ; break;
    case toml::value_t::floating: /*do some stuff*/ ; break;
    case toml::value_t::string :  /*do some stuff*/ ; break;
    default : throw std::runtime_error(
        "unexpected type : " + toml::stringize(data.at("something").type()));
}
```

The complete list of the `enum`s can be found in the section
[underlying types](#underlying-types).

The `enum`s can be used as a parameter of `toml::value::is` function like the following.

```cpp
toml::value v = /* ... */;
if(v.is(toml::value_t::boolean)) // ...
```

## More about conversion

Since `toml::find` internally uses `toml::get`, all the following examples work
with both `toml::get` and `toml::find`.

### Converting an array

You can get any kind of `container` class from a `toml::array`
except for `map`-like classes.

``` cpp
// # sample.toml
// numbers = [1,2,3]

const auto numbers = toml::find(data, "numbers");

const auto vc  = toml::get<std::vector<int>  >(numbers);
const auto ls  = toml::get<std::list<int>    >(numbers);
const auto dq  = toml::get<std::deque<int>   >(numbers);
const auto ar  = toml::get<std::array<int, 3>>(numbers);
// if the size of data.at("numbers") is larger than that of std::array,
// it will throw toml::type_error because std::array is not resizable.
```

Surprisingly, you can convert `toml::array` into `std::pair` and `std::tuple`.

```cpp
// numbers = [1,2,3]
const auto tp = toml::get<std::tuple<short, int, unsigned int>>(numbers);
```

This functionality is helpful when you have a toml file like the following.

```toml
array_of_arrays = [[1, 2, 3], ["foo", "bar", "baz"]] # toml allows this
```

What is the corresponding C++ type?
Obviously, it is a `std::pair` of `std::vector`s.

```cpp
const auto array_of_arrays = toml::find(data, "array_of_arrays");
const auto aofa = toml::get<
    std::pair<std::vector<int>, std::vector<std::string>>
    >(array_of_arrays);
```

If you don't know the type of the elements, you can use `toml::array`,
which is a `std::vector` of `toml::value`, instead.

```cpp
const auto a_of_a = toml::get<toml::array>(array_of_arrays);
const auto first  = toml::get<std::vector<int>>(a_of_a.at(0));
```

You can change the implementation of `toml::array` with `std::deque` or some
other array-like container. See [Customizing containers](#customizing-containers)
for detail.

### Converting a table

When all the values of the table have the same type, toml11 allows you to
convert a `toml::table` to a `map` that contains the convertible type.

```toml
[tab]
key1 = "foo" # all the values are
key2 = "bar" # toml String
```

```cpp
const auto data = toml::parse("sample.toml");
const auto tab = toml::find<std::map<std::string, std::string>>(data, "tab");
std::cout << tab["key1"] << std::endl; // foo
std::cout << tab["key2"] << std::endl; // bar
```

But since `toml::table` is just an alias of `std::unordered_map<toml::key, toml::value>`,
normally you don't need to convert it because it has all the functionalities that
`std::unordered_map` has (e.g. `operator[]`, `count`, and `find`). In most cases
`toml::table` is sufficient.

```cpp
toml::table tab = toml::get<toml::table>(data);
if(data.count("title") != 0)
{
    data["title"] = std::string("TOML example");
}
```

You can change the implementation of `toml::table` with `std::map` or some
other map-like container. See [Customizing containers](#customizing-containers)
for detail.

### Getting an array of tables

An array of tables is just an array of tables.
You can get it in completely the same way as the other arrays and tables.

```toml
# sample.toml
array_of_inline_tables = [{key = "value1"}, {key = "value2"}, {key = "value3"}]

[[array_of_tables]]
key = "value4"
[[array_of_tables]]
key = "value5"
[[array_of_tables]]
key = "value6"
```

```cpp
const auto data = toml::parse("sample.toml");
const auto aot1 = toml::find<std::vector<toml::table>>(data, "array_of_inline_tables");
const auto aot2 = toml::find<std::vector<toml::table>>(data, "array_of_tables");
```

### Cost of conversion

Although conversion through `toml::(get|find)` is convenient, it has additional
copy-cost because it copies data contained in `toml::value` to the
user-specified type. Of course in some cases this overhead is not ignorable.

```cpp
// the following code constructs a std::vector.
// it requires heap allocation for vector and element conversion.
const auto array = toml::find<std::vector<int>>(data, "foo");
```

By passing the exact types, `toml::get` returns reference that has no overhead.

``` cpp
const auto& tab     = toml::find<toml::table>(data, "tab");
const auto& numbers = toml::find<toml::array>(data, "numbers");
```

Also, `as_xxx` are zero-overhead because they always return a reference.

``` cpp
const auto& tab     = toml::find(data, "tab"    ).as_table();
const auto& numbers = toml::find(data, "numbers").as_array();
```

In this case you need to call `toml::get` each time you access to
the element of `toml::array` because `toml::array` is an array of `toml::value`.

```cpp
const auto& num0 = toml::get<toml::integer>(numbers.at(0));
const auto& num1 = toml::get<toml::integer>(numbers.at(1));
const auto& num2 = toml::get<toml::integer>(numbers.at(2));
```

### Converting datetime and its variants

TOML v0.5.0 has 4 different datetime objects, `local_date`, `local_time`,
`local_datetime`, and `offset_datetime`.

Since `local_date`, `local_datetime`, and `offset_datetime` represent a time
point, you can convert them to `std::chrono::system_clock::time_point`.

Contrary, `local_time` does not represents a time point because they lack a
date information, but it can be converted to `std::chrono::duration` that
represents a duration from the beginning of the day, `00:00:00.000`.

```toml
date = 2018-12-23
time = 12:30:00
l_dt = 2018-12-23T12:30:00
o_dt = 2018-12-23T12:30:00+09:30
```

```cpp
const auto data = toml::parse("sample.toml");

const auto date = toml::get<std::chrono::system_clock::time_point>(data.at("date"));
const auto l_dt = toml::get<std::chrono::system_clock::time_point>(data.at("l_dt"));
const auto o_dt = toml::get<std::chrono::system_clock::time_point>(data.at("o_dt"));

const auto time = toml::get<std::chrono::minutes>(data.at("time")); // 12 * 60 + 30 min
```

toml11 defines its own datetime classes.
You can see the definitions in [toml/datetime.hpp](toml/datetime.hpp).

## Getting with a fallback

`toml::find_or` returns a default value if the value is not found or has a
different type.

```cpp
const auto data = toml::parse("example.toml");
const auto num  = toml::find_or(data, "num", 42);
```

Also, `toml::get_or` returns a default value if `toml::get<T>` failed.

```cpp
toml::value v("foo"); // v contains String
const int value = toml::get_or(v, 42); // conversion fails. it returns 42.
```

These functions automatically deduce what type you want to get
from the default value you passed.

To get a reference through this function, take care about the default value.

```cpp
toml::value v("foo"); // v contains String
toml::integer& i = toml::get_or(v, 42); // does not work because binding `42`
                                        // to `integer&` is invalid
toml::integer opt = 42;
toml::integer& i = toml::get_or(v, opt); // this works.
```

## Expecting conversion

By using `toml::expect`, you will get your expected value or an error message
without throwing `toml::type_error`.

```cpp
const auto value = toml::expect<std::string>(data.at("title"));
if(value.is_ok()) {
    std::cout << value.unwrap() << std::endl;
} else {
    std::cout << value.unwrap_err() << std::endl;
}
```

Also, you can pass a function object to modify the expected value.

```cpp
const auto value = toml::expect<int>(data.at("number"))
    .map(// function that receives expected type (here, int)
    [](const int number) -> double {
        return number * 1.5 + 1.0;
    }).unwrap_or(/*default value =*/ 3.14);
```

## Visiting a toml::value

toml11 provides `toml::visit` to apply a function to `toml::value` in the
same way as `std::variant`.

```cpp
const toml::value v(3.14);
toml::visit([](const auto& val) -> void {
        std::cout << val << std::endl;
    }, v);
```

The function object that would be passed to `toml::visit` must be able to
recieve all the possible TOML types. Also, the result types should be the same
each other.

## Constructing a toml::value

`toml::value` can be constructed in various ways.

```cpp
toml::value v(true);     // boolean
toml::value v(42);       // integer
toml::value v(3.14);     // floating
toml::value v("foobar"); // string
toml::value v(toml::local_date(2019, toml::month_t::Apr, 1)); // date
toml::value v{1, 2, 3, 4, 5};                                 // array
toml::value v{{"foo", 42}, {"bar", 3.14}, {"baz", "qux"}};    // table
```

When constructing a string, you can choose to use either literal or basic string.
By default, it will be a basic string.

```cpp
toml::value v("foobar", toml::string_t::basic  );
toml::value v("foobar", toml::string_t::literal);
```

Datetime objects can be constructed from `std::tm` and
`std::chrono::system_clock::time_point`. But you need to specify what type
you use to avoid ambiguity.

```cpp
const auto now = std::chrono::system_clock::now();
toml::value v(toml::local_date(now));
toml::value v(toml::local_datetime(now));
toml::value v(toml::offset_datetime(now));
```

Since local time is not equivalent to a time point, because it lacks date
information, it will be constructed from `std::chrono::duration`.

```cpp
toml::value v(toml::local_time(std::chrono::hours(10)));
```

You can construct an array object not only from `initializer_list`, but also
from STL containers.

```cpp
std::vector<int> vec{1,2,3,4,5};
toml::value v = vec;
```

All the elements of `initializer_list` should be convertible into `toml::value`.

## Preserving comments

After toml11 v3, you can choose whether comments are preserved or not.

```cpp
const auto data1 = toml::parse<toml::discard_comments >("example.toml");
const auto data2 = toml::parse<toml::preserve_comments>("example.toml");
```

Comments related to a value can be obtained by `toml::value::comments()`.
The return value has the same interface as `std::vector<std::string>`.

```cpp
const auto& com = v.comments();
for(const auto& c : com)
{
    std::cout << c << std::endl;
}
```

Comments just before and just after (within the same line) a value are kept in a value.

```toml
# this is a comment for v1.
v1 = "foo"

v2 = "bar" # this is a comment for v2.
# Note that this comment is NOT a comment for v2.

# this comment is not related to any value
# because there are empty lines between v3.
# this comment will be ignored even if you set `preserve_comments`.

# this is a comment for v3
# this is also a comment for v3.
v3 = "baz" # ditto.
```

Each comment line becomes one element of a `std::vector`.

Hash signs will be removed, but spaces after hash sign will not be removed.

```cpp
v1.comments().at(0) == " this is a comment for v1."s;

v2.comments().at(1) == " this is a comment for v1."s;

v3.comments().at(0) == " this is a comment for v3."s;
v3.comments().at(1) == " this is also a comment for v3."s;
v3.comments().at(2) == " ditto."s;
```

Note that a comment just after an opening brace of an array will not be a
comment for the array.

```toml
# this is a comment for a.
a = [ # this is not a comment for a. this will be ignored.
  1, 2, 3,
  # this is a comment for `42`.
  42, # this is also a comment for `42`.
  5
] # this is a comment for a.
```

You can also append and modify comments.
The interfaces are the same as `std::vector<std::string>`.

```cpp
toml::basic_value<toml::preserve_comments> v(42);
v.comments().push_back(" add this comment.");
// # add this comment.
// i = 42
```

Also, you can pass a `std::vector<std::string>` when constructing a
`toml::basic_value<toml::preserve_comments>`.

```cpp
std::vector<std::string> comments{"comment 1", "comment 2"};
const toml::basic_value<toml::preserve_comments> v1(42, std::move(comments));
const toml::basic_value<toml::preserve_comments> v2(42, {"comment 1", "comment 2"});
```

When `toml::discard_comments` is chosen, comments will not be contained in a value.
`value::comments()` will always be kept empty.
All the modification on comments would be ignored.
All the element access in a `discard_comments` causes the same error as accessing
an element of an empty `std::vector`.

The comments will also be serialized. If comments exist, those comments will be
added just before the values.

__NOTE__: Result types from `toml::parse(...)` and
`toml::parse<toml::preserve_comments>(...)` are different.

## Customizing containers

Actually, `toml::basic_value` has 3 template arguments.

```cpp
template<typename Comment, // discard/preserve_comment
         template<typename ...> class Table = std::unordered_map,
         template<typename ...> class Array = std::vector>
class basic_value;
```

This enables you to change the containers used inside. E.g. you can use
`std::map` to contain a table object instead of `std::unordered_map`.
And also can use `std::deque` as a array object instead of `std::vector`.

You can set these parameters while calling `toml::parse` function.

```cpp
const auto data = toml::parse<
    toml::preserve_comments, std::map, std::deque
    >("example.toml");
```

Needless to say, the result types from `toml::parse(...)` and
`toml::parse<Com, Map, Cont>(...)` are different (unless you specify the same
types as default).

Note that, since `toml::table` and `toml::array` is an alias for a table and an
array of a default `toml::value`, so it is different from the types actually
contained in a `toml::basic_value` when you customize containers.
To get the actual type in a generic way, use
`typename toml::basic_type<C, T, A>::table_type` and
`typename toml::basic_type<C, T, A>::array_type`.

## TOML literal

toml11 supports `"..."_toml` literal.
It accept both a bare value and a file content.

```cpp
using namespace toml::literals::toml_literals;

// `_toml` can convert a bare value without key
const toml::value v = u8"0xDEADBEEF"_toml;
// v is an Integer value containing 0xDEADBEEF.

// raw string literal (`R"(...)"` is useful for this purpose)
const toml::value t = u8R"(
    title = "this is TOML literal"
    [table]
    key = "value"
)"_toml;
// the literal will be parsed and the result will be contained in t
```

The literal function is defined in the same way as the standard library literals
such as `std::literals::string_literals::operator""s`.

```cpp
namespace toml
{
inline namespace literals
{
inline namespace toml_literals
{
toml::value operator"" _toml(const char* str, std::size_t len);
} // toml_literals
} // literals
} // toml
```

Access to the operator can be gained with `using namespace toml::literals;`,
`using namespace toml::toml_literals`, and `using namespace toml::literals::toml_literals`.

Note that a key that is composed only of digits is allowed in TOML.
And, unlike the file parser, toml-literal allows a bare value without a key.
Thus it is difficult to distinguish arrays having integers and definitions of
tables that are named as digits.
Currently, literal `[1]` becomes a table named "1".
To ensure a literal to be considered as an array with one element, you need to
add a comma after the first element (like `[1,]`).

```cpp
"[1,2,3]"_toml;   // This is an array
"[table]"_toml;   // This is a table that has an empty table named "table" inside.
"[[1,2,3]]"_toml; // This is an array of arrays
"[[table]]"_toml; // This is a table that has an array of tables inside.

"[[1]]"_toml;     // This literal is ambiguous.
                  // Currently, it becomes a table that has array of table "1".
"1 = [{}]"_toml;  // This is a table that has an array of table named 1.
"[[1,]]"_toml;    // This is an array of arrays.
"[[1],]"_toml;    // ditto.
```

NOTE: `_toml` literal returns a `toml::value`  that does not have comments.

## Conversion between toml value and arbitrary types

You can also use `toml::get` and other related functions with the types
you defined after you implement a way to convert it.

```cpp
namespace ext
{
struct foo
{
    int         a;
    double      b;
    std::string c;
};
} // ext

const auto data = toml::parse("example.toml");

// to do this
const foo f = toml::find<ext::foo>(data, "foo");
```

There are 2 ways to use `toml::get` with the types that you defined.

The first one is to implement `from_toml(const toml::value&)` member function.

```cpp
namespace ext
{
struct foo
{
    int         a;
    double      b;
    std::string c;

    void from_toml(const toml::value& v)
    {
        this->a = toml::find<int        >(v, "a");
        this->b = toml::find<double     >(v, "b");
        this->c = toml::find<std::string>(v, "c");
        return;
    }
};
} // ext
```

In this way, because `toml::get` first constructs `foo` without arguments,
the type should be default-constructible.

The second is to implement specialization of `toml::from` for your type.

```cpp
namespace ext
{
struct foo
{
    int         a;
    double      b;
    std::string c;
};
} // ext

namespace toml
{
template<>
struct from<ext::foo>
{
    static ext::foo from_toml(const value& v)
    {
        ext::foo f;
        f.a = find<int        >(v, "a");
        f.b = find<double     >(v, "b");
        f.c = find<std::string>(v, "c");
        return f;
    }
};
} // toml
```

In this way, since the conversion function is defined outside of the class,
you can add conversion between `toml::value` and classes defined in another library.

Note that you cannot implement both of the functions because the overload
resolution of `toml::get` will be ambiguous.

If you want to convert any versions of `toml::basic_value`,
you need to templatize the conversion function as follows.

```cpp
struct foo
{
    template<typename C, template<typename ...> class M, template<typename ...> class A>
    void from_toml(const toml::basic_value<C, M, A>& v)
    {
        this->a = toml::find<int        >(v, "a");
        this->b = toml::find<double     >(v, "b");
        this->c = toml::find<std::string>(v, "c");
        return;
    }
};
// or
namespace toml
{
template<>
struct from<ext::foo>
{
    template<typename C, template<typename ...> class M, template<typename ...> class A>
    static ext::foo from_toml(const basic_value<C, M, A>& v)
    {
        ext::foo f;
        f.a = find<int        >(v, "a");
        f.b = find<double     >(v, "b");
        f.c = find<std::string>(v, "c");
        return f;
    }
};
} // toml
```

----

The opposite direction is also supported in a similar way. You can directly
pass your type to `toml::value`'s constructor by introducing `into_toml` or
`toml::into<T>`.

```cpp
namespace ext
{
struct foo
{
    int         a;
    double      b;
    std::string c;

    toml::table into_toml() const // you need to mark it const.
    {
        return toml::table{{"a", this->a}, {"b", this->b}, {"c", this->c}};
    }
};
} // ext

ext::foo    f{42, 3.14, "foobar"};
toml::value v(f);
```

The definition of `toml::into<T>` is similar to `toml::from<T>`.

```cpp
namespace ext
{
struct foo
{
    int         a;
    double      b;
    std::string c;
};
} // ext

namespace toml
{
template<>
struct into<ext::foo>
{
    static toml::table into_toml(const ext::foo& f)
    {
        return toml::table{{"a", f.a}, {"b", f.b}, {"c", f.c}};
    }
};
} // toml

ext::foo    f{42, 3.14, "foobar"};
toml::value v(f);
```

Any type that can be converted to `toml::value`, e.g. `int`, `toml::table` and
`toml::array` are okay to return from `into_toml`.

## Formatting user-defined error messages

When you encounter an error after you read the toml value, you may want to
show the error with the value.

toml11 provides you a function that formats user-defined error message with
related values. With a code like the following,

```cpp
const auto value = toml::find<int>(data, "num");
if(value < 0)
{
    std::cerr << toml::format_error("[error] value should be positive",
                                    data.at("num"), "positive number required")
              << std::endl;
}
```

you will get an error message like this.

```console
[error] value should be positive
 --> example.toml
 3 | num = -42
   |       ~~~ positive number required
```

When you pass two values to `toml::format_error`,

```cpp
const auto min = toml::find<int>(range, "min");
const auto max = toml::find<int>(range, "max");
if(max < min)
{
    std::cerr << toml::format_error("[error] max should be larger than min",
                                    data.at("min"), "minimum number here",
                                    data.at("max"), "maximum number here");
              << std::endl;
}
```

you will get an error message like this.

```console
[error] max should be larger than min
 --> example.toml
 3 | min = 54
   |       ~~ minimum number here
 ...
 4 | max = 42
   |       ~~ maximum number here
```

### Obtaining location information

You can also format error messages in your own way by using `source_location`.

```cpp
struct source_location
{
    std::uint_least32_t line()      const noexcept;
    std::uint_least32_t column()    const noexcept;
    std::uint_least32_t region()    const noexcept;
    std::string const&  file_name() const noexcept;
    std::string const&  line_str()  const noexcept;
};
// +-- line()       +--- length of the region (here, region() == 9)
// v            .---+---.
// 12 | value = "foo bar" <- line_str() returns the line itself.
//              ^-------- column() points here
```

You can get this by
```cpp
const toml::value           v   = /*...*/;
const toml::source_location loc = v.location();
```

## Serializing TOML data

toml11 enables you to serialize data into toml format.

```cpp
const toml::value data{{"foo", 42}, {"bar", "baz"}};
std::cout << data << std::endl;
// bar = "baz"
// foo = 42
```

toml11 automatically makes a small table and small array inline.
You can specify the width to make them inline by `std::setw` for streams.

```cpp
const toml::value data{
    {"qux",    {{"foo", 42}, {"bar", "baz"}}},
    {"quux",   {"small", "array", "of", "strings"}},
    {"foobar", {"this", "array", "of", "strings", "is", "too", "long",
                "to", "print", "into", "single", "line", "isn't", "it?"}},
};

// the threshold becomes 80.
std::cout << std::setw(80) << data << std::endl;
// foobar = [
// "this","array","of","strings","is","too","long","to","print","into",
// "single","line","isn't","it?",
// ]
// quux = ["small","array","of","strings"]
// qux = {bar="baz",foo=42}


// the width is 0. nothing become inline.
std::cout << std::setw(0) << data << std::endl;
// foobar = [
// "this",
// ... (snip)
// "it?",
// ]
// quux = [
// "small",
// "array",
// "of",
// "strings",
// ]
// [qux]
// bar = "baz"
// foo = 42
```

It is recommended to set width before printing data. Some I/O functions changes
width to 0, and it makes all the stuff (including `toml::array`) multiline.
The resulting files becomes too long.

To control the precision of floating point numbers, you need to pass
`std::setprecision` to stream.

```cpp
const toml::value data{
    {"pi", 3.141592653589793},
    {"e",  2.718281828459045}
};
std::cout << std::setprecision(17) << data << std::endl;
// e = 2.7182818284590451
// pi = 3.1415926535897931
std::cout << std::setprecision( 7) << data << std::endl;
// e = 2.718282
// pi = 3.141593
```

There is another way to format toml values, `toml::format()`.
It returns `std::string` that represents a value.

```cpp
const toml::value v{{"a", 42}};
const std::string fmt = toml::format(v);
// a = 42
```

Note that since `toml::format` formats a value, the resulting string may lack
the key value.

```cpp
const toml::value v{3.14};
const std::string fmt = toml::format(v);
// 3.14
```

To control the width and precision, `toml::format` receives optional second and
third arguments to set them. By default, the witdh is 80 and the precision is
`std::numeric_limits<double>::max_digit10`.

```cpp
const auto serial = toml::format(data, /*width = */ 0, /*prec = */ 17);
```

When you pass a comment-preserving-value, the comment will also be serialized.
An array or a table containing a value that has a comment would not be inlined.

## Underlying types

The toml types (can be used as `toml::*` in this library) and corresponding `enum` names are listed in the table below.

| TOML type      | underlying c++ type                | enum class                       |
| -------------- | ---------------------------------- | -------------------------------- |
| Boolean        | `bool`                             | `toml::value_t::boolean`         |
| Integer        | `std::int64_t`                     | `toml::value_t::integer`         |
| Float          | `double`                           | `toml::value_t::floating`        |
| String         | `toml::string`                     | `toml::value_t::string`          |
| LocalDate      | `toml::local_date`                 | `toml::value_t::local_date`      |
| LocalTime      | `toml::local_time`                 | `toml::value_t::local_time`      |
| LocalDatetime  | `toml::local_datetime`             | `toml::value_t::local_datetime`  |
| OffsetDatetime | `toml::offset_datetime`            | `toml::value_t::offset_datetime` |
| Array          | `array-like<toml::value>`          | `toml::value_t::array`           |
| Table          | `map-like<toml::key, toml::value>` | `toml::value_t::table`           |

`array-like` and `map-like` are the STL containers that works like a `std::vector` and
`std::unordered_map`, respectively. By default, `std::vector` and `std::unordered_map`
are used. See [Customizing containers](#customizing-containers) for detail.

`toml::string` is effectively the same as `std::string` but has an additional
flag that represents a kind of a string, `string_t::basic` and `string_t::literal`.
Although `std::string` is not an exact toml type, still you can get a reference
that points to internal `std::string` by using `toml::get<std::string>()` for convenience.
The most important difference between `std::string` and `toml::string` is that
`toml::string` will be formatted as a TOML string when outputed with `ostream`.
This feature is introduced to make it easy to write a custom serializer.

`Datetime` variants are `struct` that are defined in this library.
Because `std::chrono::system_clock::time_point` is a __time point__,
not capable of representing a Local Time independent from a specific day.

It is recommended to get `datetime`s as `std::chrono` classes through `toml::get`.

## Unreleased TOML features

There are some unreleased features in toml-lang/toml:master.
Currently, the following features are available after defining
`TOML11_USE_UNRELEASED_TOML_FEATURES` macro flag.

- Leading zeroes in exponent parts of floats are permitted.
  - e.g. `1.0e+01`, `5e+05`
- Allow raw tab characters in basic strings and multi-line basic strings.

## Breaking Changes from v2

Although toml11 is relatively new library (it's three years old now), it had
some confusing and inconvenient user-interfaces because of historical reasons.

Between v2 and v3, those interfaces are rearranged.

- `toml::parse` now returns a `toml::value`, not `toml::table`.
- `toml::value` is now an alias of `toml::basic_value<discard_comment, std::vector, std::unordered_map>`.
  - See [Customizing containers](#customizing-containers) for detail.
- The elements of `toml::value_t` are renamed as `snake_case`.
  - See [Underlying types](#underlying-types) for detail.
- Supports for the CamelCaseNames are dropped.
  - See [Underlying types](#underlying-types) for detail.
- `(is|as)_float` has been removed to make the function names consistent with others.
  - Since `float` is a keyword, toml11 named a float type as `toml::floating`.
  - Also a `value_t` corresponds to `toml::floating` is named `value_t::floating`.
  - So `(is|as)_floating` is introduced and `is_float` has been removed.
  - See [Casting a toml::value](#casting-a-tomlvalue) and [Checking value type](#checking-value-type) for detail.
- An overload of `toml::find` for `toml::table` has been dropped. Use `toml::value` version instead.
  - Because type conversion between a table and a value causes ambiguity while overload resolution
  - Since `toml::parse` now returns a `toml::value`, this feature becomes less important.
  - Also because `toml::table` is a normal STL container, implementing utility function is easy.
  - See [Finding a toml::value](#finding-a-toml-value) for detail.
- An overload of `operator<<` and `toml::format` for `toml::table`s are dropped.
  - Use `toml::value` instead.
  - See [Serializing TOML data](#serializing-toml-data) for detail.
- Interface around comments.
  - See [Preserving Comments](#preserving-comments) for detail.
- An ancient `from_toml/into_toml` has been removed. Use arbitrary type conversion support.
  - See [Conversion between toml value and arbitrary types](#conversion-between-toml-value-and-arbitrary-types) for detail.

Such a big change will not happen in the coming years.

## Running Tests

To run test codes, you need to clone toml-lang/toml repository under `build/` directory
because some of the test codes read a file in the repository.

```sh
$ mkdir build
$ cd build
$ git clone https://github.com/toml-lang/toml.git
$ cmake ..
$ make
$ make test
```

To run the language agnostic test suite, you need to compile
`tests/check_toml_test.cpp` and pass it to the tester.

## Contributors

I appreciate the help of the contributors who introduced the great feature to this library.

- Guillaume Fraux (@Luthaf)
  - Windows support and CI on Appvayor
  - Intel Compiler support
- Quentin Khan (@xaxousis)
  - Found & Fixed a bug around ODR
  - Improved error messages for invaild keys to show the location where the parser fails
- Petr Beneš (@wbenny)
  - Fixed warnings on MSVC
- Ivan Shynkarenka (@chronoxor)
  - Fixed Visual Studio 2019 warnings
- @khoitd1997
  - Fixed warnings while type conversion
- @KerstinKeller
  - Added installation script to CMake
- J.C. Moyer (@jcmoyer)
  - Fixed an example code in the documentation

## Licensing terms

This product is licensed under the terms of the [MIT License](LICENSE).

- Copyright (c) 2017-2019 Toru Niina

All rights reserved.
