# SE-NNNN: DSL Breakpoint Debugging

* Proposal: SE-NNNN
* Authors: [Apollo Zhu](https://github.com/ApolloZhu), [Richard Wei](https://github.com/rxwei)
* Review Manager: TBD
* Status: **Awaiting Review**
* Implementation: [apple/swift branch `dsl-breakpoint-debugging`](https://github.com/apple/swift/compare/dsl-breakpoint-debugging)

*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN) or [apple/swift-evolution-staging#NNNNN](https://github.com/apple/swift-evolution-staging/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)

## Introduction

We introduce a new result builder customization point that allows building breakpoint debuggable components, enabling domain specific languages embeded in Swift to provide a similar debugging experience as imperative Swift code.

```swift
@resultBuilder
enum Builder {
  /// Builds a breakpoint debuggable component where the debugInfoProvider can trigger breakpoints
  /// that are set on the original component.
  static func buildDebuggable(
    component: Component,
    debugInfoProvider: DSLDebugInfoProvider
  ) -> Component

  /// If the result builder supports buildFinalResult, builds a breakpoint debuggable final result
  /// where the debugInfoProvider can trigger breakpoints that are set on the ending curly brace.
  static func buildDebuggable(
    finalResult: FinalResult,
    debugInfoProvider: DSLDebugInfoProvider
  ) -> FinalResult
}
```

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/NNNN)

## Motivation

Domain specific languages are designed to make solving specific kinds of problems easier. For example, the RegexBuilder DSL provides a powerful abstraction over the string processing. Instead of describing how to solve a problem step by step manually:

```swift
let input = "123abc..."

func extractInformation() -> (String, String, ...)? {
  var result = ...
  var state = ...
  for character in input {
    if state.isMatchingNumber && character.isNumber {
      ...
    } else if state.isMatchingWord && character.isLetter {
      ...
    } else {
      ...
    }
  }
  return result
}
```

you can just describe the solution and the framework will do it for you:

```swift
import RegexBuilder

let regex = Regex {
  Capture {
    OneOrMore(.digit)
  }
  Capture {
    OneOrMore(.word)
  }
  ...
}

regex.firstMatch(in: input)
```

The abstraction over the underlying problem offered by DSLs makes developers’ life much easier, allowing them to implement complex logic with few lines of code. However, this very abstraction also makes DSLs hard to debug using normal means such as breakpoints.

With the manual implementation, you could set breakpoints inside the extraction function to follow string matching as it goes. But breakpoints don’t work the same when they are set on DSLs. Today, breakpoint debugging only works when result builders build a result, because it is directly transformed to imperative Swift code. However, it would be more helpful if we can debug the built result when it is used.

```swift
let regex = Regex {
  Capture {
    OneOrMore(.digit)
//  ^ breakpoint #1 on OneOrMore(.digit)
  }
  ...
}

regex.firstMatch(in: input)
//    ^ breakpoint #2 on firstMatch
```

When breakpoint #2 is triggered, it indicates that the built regex will be used to match the input String. However, if we continue the program execution, breakpoint #1 won't be triggered, preventing us from debugging the string matching process. This is because RegexBuilder already transformed the original expressions into their underlying representations. which don't share the same source location as the original expressions, thus the breakpoints won't stop during DSL execution.

## Proposed solution

The solution is to provide a way for DSL implementation to specify how breakpoints should be triggered when they use the built result. For builds compiled with no optimization, the compiler will synthesize `DSLDebugInfoProvider`s for each result in the result builder body, and the DSL implementation can choose to call the appropriate debug info provider to trigger the corresponding breakpoint and provide relevant debug information.

## Detailed design

Consider the following result builder body:

```swift
{
  expr1
  expr2
}
```

Today, it will be transformed to:

```swift
{
  let e1 = Builder.buildExpression(expr1)
  let e2 = Builder.buildExpression(expr2)
  let v1 = Builder.buildBlock(e1, e2)
  return Builder.buildFinalResult(v1)
}
```

where `buildFinalResult` and `buildExpression` are called only when the Builder defines them. With DSL breakpoint debugging, when the code is compiled with no optimization, it will be transformed instead to insert calls to `buildDebuggable`:

```swift
{
  let e1 = Builder.buildDebuggable(
            component: Builder.buildExpression(expr1),
            debugInfoProvider: <#compiler generated for expr1#>)
  let e2 = Builder.buildDebuggable(
            component: Builder.buildExpression(expr2),
            debugInfoProvider: <#compiler generated for expr2#>) 
  let v1 = Builder.buildBlock(e1, e2)
  return Builder.buildDebuggable(
            finalResult: Builder.buildFinalResult(v1),
            debugInfoProvider: <#compiler generated for closing curly brace#>)
}
```

Both of the `buildDebuggable` functions will be type checked to ensure that the return type matches the input type, so that the compilation mode does not change result builder’s component or final result type.

### Standard library addition

```swift
/// A type that can be called as a function to trigger breakpoints
/// that are set on the associated result builder component.
@frozen
public struct DSLDebugInfoProvider {
  /// Triggers breakpoints that are set on the *associated source location and* 
  /// provide debug information.
  ///
  /// - Parameters:
  ///   - context: The debugging variables to show.
  ///   - stackTrace: The additional stack trace to insert before providing
  ///     the context, where the last stack frame in the collection is the 
  ///     top of the additional stack. Defaults to no additional stack frames.
  ///   - then: The action to perform after providing the DSL debug context.
  ///     Defaults to no-op.
  public func callAsFunction(
    context: Any,
    stackTrace: some Collection<DSLDebugStackFrame> = [],
    then: () -> Void = {}
  )
}

/// A mechanism for DSLs to specify where a stack frame is for,
/// and the debugging context to show for such stack frame.
@frozen
public struct DSLDebugStackFrame {
  /// The debug info provider specifying the source location.
  public var provider: DSLDebugInfoProvider
  /// The debugging variables to show.
  public var context: Any
  
  public init(provider: DSLDebugInfoProvider, context: Any)
}
```

### Adoption example

Let’s extend the `HTMLBuilder` example from the original result builder proposal [SE-0289](https://github.com/apple/swift-evolution/blob/main/proposals/0289-result-builders.md).

```swift
enum HTML {
  case node(tag: String, children: [HTML])
  ...
  // First, we'll add two new cases to store the necessary information
  case finalResult
  indirect case debuggable(HTML, debugInfoProvider: DSLDebugInfoProvider)
}

@resultBuilder
struct HTMLBuilder {
  typealias Component = [HTML]
  ...

  // Then, add buildFinalResult to support setting breakpoint
  // on ending curly brace to debug using the final result
  typealias FinalResult = [HTML]

  static func buildFinalResult(_ component: Component) -> FinalResult {
    component
  }
  
  // Finally, add buildDebuggables to store debug info provider for each result
  static func buildDebuggable(
    component: Component,
    debugInfoProvider: DSLDebugInfoProvider
  ) -> Component {
    [.debuggable(component[0], debugInfoProvider: debugInfoProvider)]
  }
  
  static func buildDebuggable(
    finalResult: FinalResult,
    debugInfoProvider: DSLDebugInfoProvider
  ) -> FinalResult {
    finalResult + [.debuggable(.finalResult, debugInfoProvider: debugInfoProvider)]
  }
}

// Now, when the result is used, call the debugInfoProvider to trigger breakpoints
// at appropriate times and provide debug context.
extension HTML {
  func renderAsHTML(
    into stream: inout HTMLOutputStream,
    _ indentationLevel: Int = 0
  ) {
    switch self {
    case .debuggable(let component, let debugInfoProvider):
      // HTMLDSLDebugInfo is a custom, library defined struct
      // to aggregate the debugging variables. Definition ommitted.
      debugInfoProvider(context: HTMLDSLDebugInfo(
        stream: stream,
        current: component,
        indentationLevel: indentationLevel
      )) {
        component.renderAsHTML(into: &stream, indentationLevel)
      }
    case .finalResult:
      break
    case .node(let tag, let children):
      stream.write("<\(tag)>\n", with: indentationLevel)
      for child in children {
        child.renderAsHTML(into: &stream, indentationLevel + 1)
      }
      stream.write("</\(tag)>\n", with: indentationLevel)
    case ...
    }
  }
}
```

### Debugging walkthrough

With the above changes, now when we set some breakpoints as indicated by the comments:

```swift
let result = body {
  let chapter = spellOutChapter ? "Chapter " : ""
  division {
    if useChapterTitles {
      header1(chapter + "1. Loomings.")
    }
    paragraph {
      "Call me Ishmael. Some years ago" // BREAKPOINT 1 on on line 8
    } // BREAKPOINT 2 on line 9
    paragraph {
      "There is now your insular city"
    }
  }
  ...
} // BREAKPOINT 3 on line 15

result.renderAsHTML(into: &output) // BREAKPOINT 4 on line 17
```

in addition to the breakpoints that exists today, three additional ones are added automatically (the ones numbered as `x.2` below), one for each breakpoint inside the HTML result builder:

```
**(****lldb****)** **breakpoint list**
Current breakpoints:
1: file = 'main.swift', line = 8, locations = 2
1.1: where = closure #1 () -> Array<HTML> in closure #1 () -> Array<HTML> in closure #1 () -> Array<HTML> at main.swift:8:7
1.2: where = Array<HTML> #0 in closure #1 () -> Array<HTML> in closure #1 () -> Array<HTML> in closure #1 () -> Array<HTML> at main.swift:8:7

2: file = 'main.swift', line = 9, locations = 2
2.1: where = closure #1 () -> Array<HTML> in closure #1 () -> Array<HTML> in closure #1 () -> Array<HTML> at main.swift:9:5
2.2: where = Array<HTML> #1 in closure #1 () -> Array<HTML> in closure #1 () -> Array<HTML> in closure #1 () -> Array<HTML> at main.swift:9:5

3: file = 'main.swift', line = 15, locations = 2
3.1: where = closure #1 () -> Array<HTML> at main.swift:15:1
3.2: where = Array<HTML> #2 in closure #1 () -> Array<HTML> at main.swift:15:1

4: file = 'main.swift', line = 17, locations = 1
4.1: where = main.swift:17:8 
```

Once we hit breakpoint 4.1 and continue, it starts to render HTML into output and we hit the new breakpoint 1.2, and there’s a new frame variable called `dslDebugInfo` showing the custom debug information provided by the HTML DSL implementation:

```
**(****lldb****)** **thread backtrace**
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 3.2
 **** ***** **frame** **#0: [HTML] #0 in closure #1 in closure #1 in closure #1 in (dslDebugInfo=HTMLDSLDebugInfo ..., $1=() -> () ... at <compiler-generated>) at main.swift:8:7**
    frame #1: HTML.renderAsHTML(stream="...", indentationLevel=3, self=debuggable)
    frame #2: HTML.renderAsHTML(stream="...", indentationLevel=2, self=node)
    frame #3: closure #1 in HTML.renderAsHTML(component=node, stream="...", indentationLevel=2)
 **frame** **#4: [HTML] #1 in closure #1 in closure #1 in (dslDebugInfo=HTMLDSLDebugInfo ..., $1=() -> () ... at <compiler-generated>) at main.swift:7:5**
    frame #5: HTML.renderAsHTML(stream="...", indentationLevel=2, self=debuggable)
    frame #6: HTML.renderAsHTML(stream="...", indentationLevel=1, self=node)
    frame #7: closure #1 in HTML.renderAsHTML(component=node, stream="...", indentationLevel=1)
 **frame** **#8: [HTML] #0 in closure #1 in (dslDebugInfo=HTMLDSLDebugInfo ..., $1=() -> () ... at <compiler-generated>) at main.swift:3:3**
    frame #9: HTML.renderAsHTML(stream="...", indentationLevel=1, self=debuggable)
    frame #10: HTML.renderAsHTML(stream="...", indentationLevel=0, self=node)
    frame #11: HTMLNode.renderAsHTML(stream="...", self=HTMLNode ...)
    frame #12: main.swift:17:8
    frame #13: dyld`start + 2520
    
**(lldb) frame variable**
(HTLMBuilderExample.HTMLDSLDebugInfo) dslDebugInfo = {
  stream = "<body>\n  <div>\n    <h1>\n      Chapter 1. Loomings.\n    </h1>\n    <p>\n"
  current = text (text = "Call me Ishmael. Some years ago")
  indentationLevel = 3
}
```

From the debug information, we can see that it’s about to process the current HTML text component, and continue execution will hit breakpoint 2.2, with the `stream` now updated to contain the content of the text that just got rendered:

```
**(****lldb****)** **po dslDebugInfo**
▿ HTMLDSLDebugInfo
  - stream : "... <p>\n Call me Ishmael. Some years ago\n"
  - current : HTML.finalResult
  - indentationLevel : 3
```

Finally, after rendering all the HTML content, breakpoint 3.2 is hit with `stream` containing the final result:

```
**(****lldb****)** **po dslDebugInfo**
▿ HTMLDSLDebugInfo
  - stream : "... Some years ago\n <#everything after ...#>"
  - current : HTML.finalResult
  - indentationLevel : 1
```

### Additional stack trace

Note that the indirect `HTML` enum preserves the original tree structure, so the stack trace at breakpoint 2.1 above also shows a hierarchy that’s similar to the original result builder body. However, some DSLs flatten the underlying representation, requiring additional mechanism to present a similar stack trace. Thus, `DSLDebugInfoProvider` allows DSLs to insert additional synthetic stack frames when triggering a breakpoint:

```swift
currentDebugInfoProvider(context: currentDebugInfo, stackTrace: [
    DSLDebugStackFrame(provider: rootDebugInfoProvider,
                       context: rootDebugInfo),
    ...
    DSLDebugStackFrame(provider: parentDebugInfoProvider,
                       context: parentDebugInfo),
])
```

## Source compatibility

This is a strictly additive proposal with no source-breaking changes.

## Effect on ABI stability

This is a strictly additive proposal with no ABI-breaking changes.

## Effect on API resilience

Since both the `DSLDebugInfoProvider` and `DSLDebugStackFrame` types are frozen, they cannot add stored properties without breaking the ABI. However, the design should allow implementing additional APIs as functions and/or computed properties on these types for future needs. Therefore, the proposal has no immediate impact on API resilience and is future-proofed for further development.

## Alternatives considered

### Overload buildExpression and buildFinalResult to accept DSLDebugInfoProvider

`buildExpression` is designed to lift the results of expression-statements into the `Component` internal currency type, so we could require result builder implementations to define overloads of `buildExpression` with an extra `debugInfoProvider` argument to build debuggable `Component`. However, it’s common for result builders to accept multiple expression types. Consider the `HTMLBuilder` example from [SE-0289](https://github.com/apple/swift-evolution/blob/main/proposals/0289-result-builders.md#expression-statements) again, 3 types of expressions are accepted:

```swift
static func buildExpression(_ text: String) -> [HTML]
static func buildExpression(_ node: HTMLNode) -> [HTML]
static func buildExpression(_ value: HTML) -> [HTML]
```

To make the HTML DSL debuggable, it needs to double the amount of `buildExpression`s, providing one more overload for each expression type:

```swift
static func buildExpression(_ text: String, debugInfoProvider: DSLDebugInfoProvider) -> [HTML]
static func buildExpression(_ node: HTMLNode, debugInfoProvider: DSLDebugInfoProvider) -> [HTML]
static func buildExpression(_ value: HTML, debugInfoProvider: DSLDebugInfoProvider) -> [HTML]
```

However, only one `buildDebuggable` is needed no matter how many types of expressions are supported:

```swift
static func buildDebuggable(component: [HTML], debugInfoProvider: DSLDebugInfoProvider) -> [HTML]
```

In addition, it is possible to (accidentally) define overloads where the corresponding pair of `buildExpression`s return different `Component` types. While this might be okay (or even intentional) if the result builder tolerates this behavior, changing the built result type depending on the current optimization mode might lead to unexpected build failure when developer changes from debug to release. It’s possible for the compiler to try to identify the pairs of `buildExpression`s and compare their return types, but it would be much more complex than checking the single `buildDebuggable` returns the same type that it takes.

### Use `buildDebuggable(_:debugInfoProvider:)` for sharing implementations

Building a debuggable component generally differs from building a debuggable final result. DSL execution most likely want to call debug info provider for a component before using the component, and that for the final result after producing the final result. A probable attempt to share the implementation of `buildDebuggable`s would be:

```swift
static func buildDebuggable<D: DebuggableComponentOrFinalResult>(
  _ debuggable: D, 
  debugInfoProvider: DSLDebugInfoProvider
) -> D {
  debuggable.debugInfoProvider = debugInfoProvider
  return debuggable
}
```

However, this won’t work if a type can both be a `Component` and `FinalResult`. Not only would the type need to somehow store two debug info providers, it also can’t determine which of the debug info providers is for it being a component vs final result. One solution to address the second problem would be adding “this is component/final result” as a field in `DSLDebugInfoProvider`, but it will be an unnecessary binary size increase for most other uses cases. It will also be more efficient if this can be determined during builder transformation instead of needing to check every single time

Having an argument label also makes it easier for the compiler to check if the result builder implementation supports debuggable component/final result without performing type checking.

### `#debugInfoProvider` magic literal expression

A natural extension is to add a new magic literal expression like `#debugInfoProvider` where DSL implementations can add to any desired source location. However, there are not enough compelling examples to suggest that this is a necessary feature to implement as a part of this proposal.

## Future Directions

### Stepping

Unlike imperative code, it’s unclear what it means for DSL debugging to support step in, step over, and step out. Consider the following regex example:

```swift
struct CDoubleParser: CustomConsumingRegexComponent {
  func consuming(
    _ input: String, startingAt index: String.Index, in bounds: Range<String.Index>
  ) throws -> (upperBound: String.Index, output: Double)? {
    ... // line 5
  }
}

let inner = Regex { ... } // line 9
let outer = Regex {
  OneOrMore { // line 11
    CDoubleParser() // line 12
    
     One(.date(format: "\(month: .twoDigits)", locale: .current, timeZone: .gmt))
//  line 14 above is equivalent to this Foundation Date parser:
//  Date.ParseStrategy(format: "\(month: .twoDigits)", timeZone: .gmt)
  }
  inner // line 18
  ... // line 19
} // line 20

let result = outer.firstMatch(...).1
// line 22 above                   ^ 
```

If we are one line 18, the most intuitive stepping behavior is that step over should take us to line 19, and step in should take us to line 9. However, if the same logic applies when we are on line 11, then step over would take us to line 18, and to get to line 12 we’ll need to use step in. This might seem a little counterintuitive at first, but one can still get used to such stepping logic.

Line 12 and line 14 are similar, but step into line 12 should take us to line 5, where step into line 14 might jump to some assembly code for Foundation’s Date parser. However, either of them will crossover the boundary between DSL and imperative debugging unlike stepping into line 18. In addition, it would be preferred if stepping out of line 5 takes us back to DSL debugging on line 14, crossing the boundary again.

After presenting the final result on line 20, step over should jump back to line 22 right before accessing `.1`.

It’ unclear how all of the above could be done and stepping is not an essential part of this proposal. Nonetheless, the design of DSLDebugInfoProvider should be flexible enough to allow future proposals to address how libraries can define these behaviors.

## Acknowledgments


