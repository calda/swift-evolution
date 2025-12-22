# `@ArrayBuilder`

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Cal Stephens](https://github.com/calda)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: TODO
* Pitch: [link](https://forums.swift.org/t/add-arraybuilder-to-the-standard-library/83811/3)

## Introduction

We should add a `@ArrayBuilder` result builder to the standard library, and a static method for using an `@ArrayBuilder` to create an array value. We should also improve the ergonomics of working with generic `@resultBuilder` types.

## Motivation

Since being introduced with the release of SwiftUI in 2019, and being formalized in [SE-0289](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0289-result-builders.md), result builders have set the standard for defining lists and trees in code. Compared to imperative code to define these structures, result builders are less verbose and easier to evolve.

Take this example of constructing a `[String]` array representing arguments for a [swift-format](https://github.com/swiftlang/swift-format) invocation:

```swift
var arguments: [String] {
    var arguments = [String]()
    arguments.append("format")

    arguments.append("--configuration")
    arguments.append(configurationPath)

    arguments.append("--in-place")

    if recursive {
        arguments.append("--recursive")
    }

    if let cachePath {
        arguments.append("--cache-path")
        arguments.append(cachePath)
    }

    return arguments
}
```

Compared to the declarative simplicity we are used to from result builders, this code is imperative and verbose.

If instead written using an `@ArrayBuilder`, the code is much simpler and focuses exclusively on the business logic rather than the imperative array manipulation.

```swift
@ArrayBuilder
var arguments: [String] {
    "format"

    "--configuration"
    configurationPath

    "--in-place"

    if recursive {
        "--recursive"
    }

    if let cachePath {
        "--cache-path"
        cachePath
    }
}
```

## Proposed solution

We should add an `ArrayBuilder` result builder to the standard library, and a static method for using an `@ArrayBuilder` to create an array value.

While it is [straightforward](https://github.com/search?q=ArrayBuilder+language%3ASwift&type=repositories&l=Swift) to define an `ArrayBuilder` type in your own project, this is a strong candidate to include in the standard library:

 - Initializing array values is such a common operation that this addition will be relevant to the vast majority of Swift projects.
 - Including `@ArrayBuilder` in the standard library enables it to become a standard best practice across the Swift ecosystem.
 - Standardizing on a single implementation reduces fragmentation.

## Detailed design

### `ArrayBuilder`

The `ArrayBuilder` type should accept all of the relevant `@resultBuilder` functionality, including:
 - `Element` values, using [`append`](https://developer.apple.com/documentation/swift/array/append(_:))
 - `Sequence<Element>` values, using [`append(contentsOf:)`](https://developer.apple.com/documentation/swift/array/append(contentsof:))
 - Conditional values (if statements, optional `Element` and `Sequence` values)
 - For loops

A future draft will include the full public API of the proposed `ArrayBuilder` type.

This enables the following `@ArrayBuilder` method:

```swift
@ArrayBuilder
var arguments: [String] {
    "format"
    "--in-place"

    if recursive {
        "--recursive"
    }
}
```


### `Array.build(_:)`

We should add a `static func build(@ArrayBuilder _ makeArray: () -> [Element]) -> [Element]` method to the `Array` type.

```swift
let arguments = Array.build {
    "format"
    "--in-place"

    if recursive {
        "--recursive"
    }
}

// Above example uses type inference to infer `Element` type.
// Other valid syntax includes:
let arguments = [String].build {
    ...
}

let arguments: [String] = .build {
    ...
}
```

## Improving ergonomics of generic `@resultBuilder`s

Today, when using a generic `@resultBuilder`, you must explicitly specify the generic arguments. Otherwise, you get an error like `"reference to generic type 'ArrayBuilder' requires arguments in <...>"`.

```swift
@ArrayBuilder<String>
var arguments2: [String] {
    "format"
    "--in-place"

    if recursive {
        "--recursive"
    }
}
```

This is redundant since the `ArrayBuilder.Result.Element` type can be inferred to be the `Element` type of the attached declaration's return type. Instead, we should allow this to be inferred so that a simpler syntax can be used:

```swift
@ArrayBuilder
var arguments2: [String] {
    "format"
    "--in-place"

    if recursive {
        "--recursive"
    }
}
```

This is useful regardless of whether or not we add `ArrayBuilder` to the standard library, since the enhancement would be available for user-defined `ArrayBuilder` types.

## Source compatibility

There is always the possibility of source breaks when adding new types or declarations to the standard library, however this is unlikely given user-defined types take precedence over standard library types during symbol lookup.

## ABI compatibility

Since this proposal is purely additive, it has no ABI impact.

## Implications on adoption

Since result builders have no runtime component, it will be possible to backport `@ArrayBuilder` support to existing operating system versions.

## Future directions

### Add `.build` to other types

We could also add `.build` static function to other sequence / collection types that currently have an `init` that accepts an `Array` / `Sequence`. `Set` is probably the best example. However, a `@SetBuilder` that can create a set in a single pass (rather than building an array and then converting it to a set) may be preferrable.

## Alternatives considered

### `CollectionBuilder`

Instead of only supporting arrays, we could create a way to construct any Swift collection. However, the [`Collection`](https://developer.apple.com/documentation/Swift/Collection) type provides no way to construct and manipulate collection values. We need at a minimum [`RangeReplaceableCollection`](https://developer.apple.com/documentation/swift/rangereplaceablecollection), which provides `init` and `append` requirements. 

`Array`, `ArraySlice`, `ContiguousArray`, `Slice`, `String`, and `Substring` are the only standard library types that conform to `RangeReplaceableCollection`. It seems potentially undesirable to support `String`, due to the Unicode-related implications. 

Given the `Array` use case would be multiple orders of magnitude more common than `ArraySlice`, `ContiguousArray`, and `Slice`, it seems reasonable to anchor on the `Array` use case. `ArrayBuilder` is a much better name for the `Array` use case than `CollectionBuilder`.

### `Array.init(build:)`

This proposal includes an `Array.build(_:)` method. Instead, we could name that method `Array.init(build:)`.

Including `build` in the name seems like a good idea to indicate that this is using a `@ArrayBuilder` / `@resultBuilder`. This method will commonly be called with a trailing closure, so the only way to ensure the name is visible is to use a `static function`:

```swift
let arguments = Array(build: {
    "format"
    "--in-place"

    if recursive {
        "--recursive"
    }
})

// Trailing closure: no `build`.
let arguments = Array {
    "format"
    "--in-place"

    if recursive {
        "--recursive"
    }
}
```

Trailing closure syntax is also not currently supported when using `[Element]` syntax:

```swift
// error: 'let' declarations cannot be computed properties
let arguments = [String] {
    "format"
    "--in-place"

    if recursive {
        "--recursive"
    }
}
```

While this can be worked around by using `let arguments: [String] = .init { ... }` syntax instead, ideally we select a syntax that supports type inference from the RHS value.

`Array.build` results in idiomatic call sites that include the `build` name as a signal to the `@ArrayBuilder` semantics.

```swift
let arguments = Array.build {
    "format"
    "--in-place"

    if recursive {
        "--recursive"
    }
}

let arguments = [String].build {
    "format"
    "--in-place"

    if recursive {
        "--recursive"
    }
}

let arguments: [String] = .build {
    "format"
    "--in-place"

    if recursive {
        "--recursive"
    }
}
```

## Acknowledgments

Thank you to the authors of [SE-0289](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0289-result-builders.md) for moving the language forward so much!
