# Unit testing

The `WebSharper.Testing` library (available on [NuGet](https://www.nuget.org/packages/websharper.testing)) exposes a clean F# syntax to write unit tests for libraries, services and websites.
It is a wrapper around `QUnit` for presentation which provides transparent handling of synchronous and asynchronous expressions, and generating random data for property testing.
The tests page is meant to be ran in a browser, but the server-side of a website can be tested too by making remote calls from the tests.

All functionality within are accessible with the `WebSharper.Testing` namespace.
Code samples below are also assuming that the module or assembly containing them is annotated with `[<JavaScript>]` except
when referring to being server-side code.

## Contents

- [Categories and tests](#categories-and-tests): organizing and running tests
    - [TestCategory](#testcategory)
    - [Test](#test)
    - [Expect](#expect)
    - [Runner.RunTests](#runnerruntests)
- [Assertions](#assertions): simple assertions
    - [isTrue](#istrue)
    - [isFalse](#isfalse)
- [Equality checks](#equality-checks): equality/non-equality checkers
    - [equal](#equal)
    - [notEqual](#notequal)
    - [jsEqual](#jsequal)
    - [notJsEqual](#notjsequal)
    - [strictEqual](#strictequal)
    - [notStrictEqual](#notstrictequal)
    - [deepEqual](#deepequal)
    - [notDeepEqual](#notdeepequal)
    - [propEqual](#propequal)
    - [notPropEqual](#notpropequal)
    - [approxEqual](#approxequal)
    - [notApproxEqual](#notapproxequal)
- [Exception testing](#exception-testing): check for a raised exception
    - [raises](#raises)
- [Asynchronous tests](#asynchronous-tests): using `Async` values inside test workflows
- [Property testing](#property-testing): property testing with random values
    - [Do](#do)
    - [property](#property)
    - [propertyWith](#propertywith)
    - [propertyWithSample](#propertywithsample)
- [Looping](#looping): registering multiple tests with a loop
    - [forEach](#foreach)
- [Sample value generators](#sample-value-generators): compose complex test values 
    - [RandomValues.Generator](#randomvaluesgenerator)
    - [RandomValues.Sample](#randomvaluessample)
    - [Random generator constructors](#random-generator-constructors)
    - [Random generator combinators](#random-generator-combinators)

## Categories and tests

### TestCategory

Creates a definition for a single named test category.
You can think of it as similar to F#'s `lazy` keyword: the code body inside it is not executed immediately,
but when only when it will be passed to `Runner.RunTests`.
So the returned value (of type `TestCategory`) need to be assigned to local or module variable if you want to run it later in your project.

#### Example:

```fsharp
let MyTests =
    TestCategory "MyTests" {
        Test "Equality" {
            equal 1 1
        }
    }
```

### Test

Creates a definition for a single named test.
Similar to `TestCategory`, it delays the execution of its content.
Does not return anything, but registers tests in `QUnit`.
Do not evaluate it in a module context if you want the test to be under a category reliably.

#### Example:

```fsharp
let EqualityTests() =
    Test "Equality" {
        equal 1 1
        notEqual 1 2
    }
```

`Test "name"` is a computation expression builder, which defines many named custom operations to use for
individual assertions.
These all have four versions for optionally naming it and/or using it with an asynchronous argument:

* operations with `Msg` also accept a name for that single assertion,
* operations with `Async` accept a value of type `Async<'T>` as the actual value instead. Expected value is still provided as a value of `'T`.

### Runner.RunTests

Takes an array of `TestCategory` values
Returns an `IControlBody`, which can be used inside the `client <@ ... @>` helper to serve a page that runs the given tests,
or call its `ReplaceInDom` method directly on the client, to replace a placeholder DOM element to the content generated by `QUnit`.

#### Example:

```fsharp
let RunAllTests() =
    Runner.RunTests [|
        MyTests
    |]
```

Later, in server-side code, serve this on a [Sitelet](sitelets.md) endpoint:

```fsharp
    Content.Page(
        Title = "Unit tests",
        Body = client <@ RunAllTests() @>
    )
```

### Expect

Tells the testing framework how many test cases are expected to be registered for this category.
If the category would register no tests, `QUnit` reports it as a failed test case, unless you add `expect 0`.

#### Example:

```fsharp
let MyTests =
    TestCategory "MyTests" {
        Test "Equality" {
            expect 2
            equal 1 1
            notEqual 1 2
        }
        Test "Not failing" {
            expect 0
            // a call to some outside function:
            doNotFail() 
            // if this throws an error, that still fails the test case, otherwise ok
        }
    }
```

## Assertions

### isTrue

Checks if a single argument expression evaluates to `true`.

#### Example:

```fsharp
    Test "Equality" {
        isTrue (1 = 1)
        isTrueMsg (1 = 1) "One equals one"
        isTrueAsync (async { return 1 = 1 }) 
        isTrueMsgAsync (async { return 1 = 1 }) "One equals one, async version"
    }
```

### isFalse

The negated versions of the above, the test passes if the expression evaluates to `false`.

#### Example:

```fsharp
    Test "Equality" {
        isFalse (1 = 2)
        isFalseMsg (1 = 2) "One equals two is false"
        isFalseAsync (async { return 1 = 2 }) 
        isFalseMsgAsync (async { return 1 = 2 }) "One equals two is false, async version"
    }
```

## Equality checks

### equal

Checks if the two expressions evaluate to equal values as tested with WebSharper's equality.
This is the same as using the `=` operator from F# code, unions and records have structural equality
and overrides on `Equals` method or implementing `IEquatable` is respected.

#### Example:

```fsharp
    Test "Equality" {
        equal (Some 1) (Some 1)
        equalMsg (Some 1) (Some 1) "Option equality"
        equalAsync (async { return Some 1 }) (Some 1)
        equalMsgAsync (async { return Some 1 }) (Some 1) "Option equality, async version"
    }
```

### notEqual

The negated version of the above, fails if values are equal with WebSharper's equality.

### jsEqual

Checks if the two expressions evaluate to equal values as tested with JavaScript's `==` operator.
This is the same as using the `==.` operator in F# code (available with `open WebSharper.JavaScript`),
which is directly translated to `==` in JavaScript.

```fsharp
    Test "Equality" {
        jsEqual 1 1
        jsEqualMsg 1 1 "One equals one"
        jsEqualAsync (async { return 1 }) 1
        jsEqualMsgAsync (async { return 1 }) 1 "One equals one, async version"
    }
```

### notJsEqual

The negated version of the above, fails if values are equal with JavaScript's `==` operator.

### strictEqual

Checks if the two expressions evaluate to equal values as tested with JavaScript's `===` operator.
This is the same as using the `===.` operator in F# code (available with `open WebSharper.JavaScript`),
which is directly translated to `===` in JavaScript.

### notStrictEqual

The negated version of the above, fails if values are equal with JavaScript's `===` operator.

### deepEqual

Checks if the two expressions evaluate to equal values as tested with `QUnit`'s `deepEqual` function
which is a deep recursive comparison, working on primitive types, arrays, objects, regular expressions, dates and functions.

### notDeepEqual

The negated version of the above, uses `QUnit`'s `notDeepEqual` function instead.

### propEqual

Checks if the two expressions evaluate to equal values as tested with `QUnit`'s `propEqual` function
which compares just the direct properties on an object with strict equality (`===`).

### notPropEqual

The negated version of the above, uses `QUnit`'s `notPropEqual` function instead.

### approxEqual

Compares floating point numbers, where a difference of `< 0.0001` is accepted.

### notApproxEqual

The negated version of the above, fails if the difference of the two values is `< 0.0001`.

## Exception testing

### raises

Checks that the expression argument is raising any exception, passes test if it does.
Note that actual value arguments are always implicitly enclosed within a lambda function,
so don't need to pass a function explicitly to make sure that the value is only evaluated
when the test are running.

#### Example:

```fsharp
    Test "Exceptions" {
        raises (failwith "should fail")
        raisesMsg (failwith "should fail") "Failure is expected"
        raisesAsync (async { failwith "should fail" })
        raisesMsgAsync (async { failwith "should fail" }) "Failure is expected from inside async"
    }
```

## Asynchronous tests

`Test` computation expressions also allow binding an `async` workflow at any point.
If this is not used, and all assertions are synchronous then the entire test case will
run synchronously for optimal performance.

#### Example:

```fsharp
    Test "Equality" {
        equal 1 1 
        let! one = async { return 1 }
        equal one 1
    }
```

## Property testing

### Do

Using `Do` is very similar to using `Test "Name"`, it is a computation expression builder, enabling the same custom operations.
The difference is that it is not named, and also by itself does not registers tests,
as it's intended use is to create sub-tests, that can be executed inside a named test when doing property testing.

#### Example:

```fsharp
let SubTest x =
    Do {
        equal x x 
    }
```

### property

Auto-generates 100 random values based on a type and runs a sub-test with all of them.
Supported types are `int`, `float`, `bool`, `string`, `unit`, `obj` and also 
tuples, lists, arrays, options, `seq` and `ResizeArray` made from these.
Using the `obj` type results in values from mix of various types.
When using a non-supported type, it results in a compile-time error.

#### Example:

```fsharp
    Test "Equality on ints is reflexive" {
        property (fun (x: int) ->
            Do {
                equal x x 
            }
        )
    }
```

### propertyWith

Similar to above, but the generator logic is not inferred from type, but passed to the operation.
It takes as first argument a record value of type `RandomValues.Generator`, which is [described here](#randomgenerator).

There are also constructor functions and combinators in the `Random` module to get `Generator` values, 
allowing easier composition of complex testing values.

#### Example:

```fsharp
    Test "Equality on ints is reflexive" {
        propertyWith RandomValues.Int (fun (x: int) ->
            Do {
                equal x x 
            }
        )
    }
```

### propertyWithSample

Similar to above, but the argument is an exact sample on which the property is tested.
See [RandomValues.Sample](#randomsample) below.

#### Example:

```fsharp
    Test "Equality on ints is reflexive" {
        propertyWithSample (RandomValues.Sample [| 1; 2; 3 |]) (fun (x: int) ->
            Do {
                equal x x 
            }
        )
    }
```

## Looping

### forEach

You cannot use a regular `for` loop inside a `Test` computation expression, but you can emulate it with the `forEach` operation.
Its use is similar to property testing, you can use `Do` to define the body of the loop.

#### Example:

```fsharp
    Test "Equality on ints is reflexive" {
        forEach { 1 .. 3 } (fun x -> 
            Do {
                equal x x
            }
        )
    }
```

## Sample value generators

### RandomValues.Generator

A generator is a record that can hold an array of static values to always test against,
and a function that can return additional values to be tested dynamically.
It is defined as such:

```fsharp
// module Random
type Generator<'T> =
    {
        /// An array of values that must be tested against.
        Base: 'T []
        /// A function generating a new random value.
        Next: unit -> 'T
    }
```

### RandomValues.Sample

The `RandomValues.Sample` type is a thin wrapper around an array of values, exposing multiple constructors
to create from a given array or explicit or inferred generators. 

#### Example:

```fsharp
    let sampleFromArray = RandomValues.Saple([| 1; 2; 3 |]);
    let sampleFromGenerator = RandomValues.Sample(RandomValues.Int); // creates 100 values
    let sampleFromGenerator10 = RandomValues.Sample(RandomValues.Int, 10); // creates given number of values
    let sampleInferred = RandomValues.Sample<int>();
    let sampleInferred10 = RandomValues.Sample<int>(10); // creates given number of values
```

### Random generator constructors

The following values and function produce simple `RandomValues.Generator` values:

* `RandomValues.StandardUniform`: Standard uniform distribution sampler.
* `RandomValues.Exponential`: Exponential distribution sampler.
* `RandomValues.Boolean`: Generates random booleans.
* `RandomValues.Float`: Generates random doubles. 
* `RandomValues.FloatExhaustive`: Generates random doubles, including corner cases (NaN, Infinity).
* `RandomValues.Int`: Generates random int values.
* `RandomValues.Natural`: Generates random natural numbers (0, 1, ...).
* `RandomValues.Within low hi`: Generates integers within a range.
* `RandomValues.FloatWithin low hi`: Generates doubles within a range.
* `RandomValues.String`: Generates random strings.
* `RandomValues.ReadableString`: Generates random readable strings.
* `RandomValues.StringExhaustive`: Generates random strings including nulls.
* `RandomValues.Auto<'T>`: Auto-generate values based on type, same as `property` uses internally.
* `RandomValues.Anything`: Generates a mix of ints, floats, bools, strings and tuples, lists, arrays, options of these.

### Random generator combinators

* `RandomValues.Map mapping gen`: Maps a function over a generator.
* `RandomValues.SuchThat predicate gen`: Filter the values of a generator by a predicate.
* `RandomValues.ArrayOf gen`: Generates arrays (up to lengh 100), getting the items from the given generator.
* `RandomValues.ResizeArrayOf gen`: Similar to as above, generates `ResizeArray`s.
* `RandomValues.ListOf gen`: Similar to as above, generates `List`s.
* `RandomValues.Tuple2Of (a, b)`: Generates a 2-tuple, getting the items from the given generators.
* `RandomValues.Tuple3Of (a, b, c)`: Same as above for 3-tuples.
* `RandomValues.Tuple4Of (a, b, c, d)`: Same as above for 4-tuples.
* `RandomValues.OneOf arr`: Generates values from a given array.
* `RandomValues.Mix a b`: Mixes values coming from two generators, alternating between them.
* `RandomValues.MixMany gens`: Mixes values coming from an array of generators.
* `RandomValues.MixManyWithoutBases gens`: Same as above, but skips the constant base values.
* `RandomValues.Const x`: Creates a generator that always returns the same value.
* `RandomValues.OptionOf x`: Creates a generator has `None` as a base value, then maps items using `Some`.
