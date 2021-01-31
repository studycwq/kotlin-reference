---
type: doc
layout: reference
category: "JavaScript"
title: "Calling JavaScript from Kotlin"
---

# Calling JavaScript from Kotlin

Kotlin was first designed for easy interoperation with the Java platform: it sees Java classes as Kotlin classes, and Java sees Kotlin classes as Java classes.

However, JavaScript is a dynamically typed language, which means it does not check types at compile time. You can freely talk to JavaScript from Kotlin via  [dynamic](dynamic-type.html) types. If you want to use the full power of the Kotlin type system, you can create external declarations for JavaScript libraries which will be understood by the Kotlin compiler and the surrounding tooling.

An experimental tool to automatically create Kotlin external declarations for npm dependencies which provide type definitions (TypeScript / `d.ts`) called [Dukat](js-external-declarations-with-dukat.html) is also available.

## Inline JavaScript

You can inline some JavaScript code into your Kotlin code using the [`js("...")`](/api/latest/jvm/stdlib/kotlin.js/js.html) function.
For example:

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
fun jsTypeOf(o: Any): String {
    return js("typeof o")
}
```
</div>

Because the parameter of `js` is parsed at compile time and translated to JavaScript code "as-is", it is required to be a string constant. So, the following code is incorrect:

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
fun jsTypeOf(o: Any): String {
    return js(getTypeof() + " o") // error reported here
}
fun getTypeof() = "typeof"
```
</div>

Note that invoking `js()` returns a result of type [`dynamic`](dynamic-type.html), which provides no type safety at compile time.

## `external` modifier

To tell Kotlin that a certain declaration is written in pure JavaScript, you should mark it with the `external` modifier.
When the compiler sees such a declaration, it assumes that the implementation for the corresponding class, function or
property is provided externally (by the developer or via an [npm dependency](js-project-setup.html#npm-dependencies)), and therefore does not try to generate any JavaScript code from the declaration. This is also why `external` declarations can't have a body. For example:

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
external fun alert(message: Any?): Unit

external class Node {
    val firstChild: Node

    fun append(child: Node): Node

    fun removeChild(child: Node): Node

    // etc
}

external val window: Window
```
</div>

Note that the `external` modifier is inherited by nested declarations. This is why in the example `Node` class, we do not need to add the `external` modifier before member functions and properties.

The `external` modifier is only allowed on package-level declarations. You can't declare an `external` member of a non-`external` class.


### Declaring (static) members of a class

In JavaScript you can define members either on a prototype or a class itself:

<div class="sample" markdown="1" theme="idea" data-highlight-only>
``` javascript
function MyClass() { ... }
MyClass.sharedMember = function() { /* implementation */ };
MyClass.prototype.ownMember = function() { /* implementation */ };
```
</div>

There is no such syntax in Kotlin. However, in Kotlin we have [`companion`](object-declarations.html#companion-objects) objects. Kotlin treats companion objects
of `external` classes in a special way: instead of expecting an object, it assumes members of companion objects to be members of the class itself. `MyClass` from the example above can be described as follows:

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
external class MyClass {
    companion object {
        fun sharedMember()
    }

    fun ownMember()
}
```
</div>


### Declaring optional parameters

If you are writing an external declaration for a JavaScript function which has an optional parameter, use `definedExternally`. This delegates the generation of the default values to the JavaScript function itself:

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
external fun myFunWithOptionalArgs(
    x: Int,
    y: String = definedExternally,
    z: String = definedExternally
)
```
</div>

With this external declaration, you can call `myFunWithOptionalArgs` with one required argument and two optional arguments, where the default values are calculated by the JavaScript implementation of `myFunWithOptionalArgs`.


### Extending JavaScript classes

You can easily extend JavaScript classes as if they were Kotlin classes. Just define an `external open` class and
extend it by a non-`external` class. For example:

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
open external class Foo {
    open fun run()
    fun stop()
}

class Bar: Foo() {
    override fun run() {
        window.alert("Running!")
    }

    fun restart() {
        window.alert("Restarting")
    }
}
```
</div>

There are some limitations:

- When a function of an external base class is overloaded by signature, you can't override it in a derived class.
- You can't override a function with default arguments.
- Non-external classes can't be extended by external classes.

### `external` interfaces

JavaScript does not have the concept of interfaces. When a function expects its parameter to support two methods `foo`
and `bar`, you would just pass in an object that actually has these methods.

You can use interfaces to express this concept in statically typed Kotlin:

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
external interface HasFooAndBar {
    fun foo()

    fun bar()
}

external fun myFunction(p: HasFooAndBar)
```
</div>

A typical use case for external interfaces is to describe settings objects. For example:

<div class="sample" markdown="1" theme="idea" data-highlight-only auto-indent="false">
```kotlin
external interface JQueryAjaxSettings {
    var async: Boolean

    var cache: Boolean

    var complete: (JQueryXHR, String) -> Unit

    // etc
}

fun JQueryAjaxSettings(): JQueryAjaxSettings = js("{}")

external class JQuery {
    companion object {
        fun get(settings: JQueryAjaxSettings): JQueryXHR
    }
}

fun sendQuery() {
    JQuery.get(JQueryAjaxSettings().apply {
        complete = { (xhr, data) ->
            window.alert("Request complete")
        }
    })
}
```
</div>

External interfaces have some restrictions:

- They can't be used on the right-hand side of `is` checks. 
- They can't be passed as reified type arguments.
- They can't be used in class literal expressions (such as `I::class`).
- `as` casts to external interfaces always succeed.
    Casting to external interfaces produces the "Unchecked cast to external interface" compile time warning. The warning can be suppressed with the `@Suppress("UNCHECKED_CAST_TO_EXTERNAL_INTERFACE")` annotation.

    IntelliJ IDEA can also automatically generate the `@Suppress` annotation. Open the intentions menu via the light bulb icon or Alt-Enter, and click the small arrow next to the "Unchecked cast to external interface" inspection. Here, you can select the suppression scope, and your IDE will add the annotation to your file accordingly.

### Casting
In addition to the ["unsafe" cast operator](/docs/reference/typecasts.html#unsafe-cast-operator) `as`, which throws a `ClassCastException` in case a cast is not possible, Kotlin/JS also provides [`unsafeCast<T>()`](/api/latest/jvm/stdlib/kotlin.js/unsafe-cast.html). When using `unsafeCast`, _no type checking is done at all_ during runtime. For example, consider the following two methods:

<div class="sample" markdown="1" theme="idea" data-highlight-only auto-indent="false">

```kotlin
fun usingUnsafeCast(s: Any) = s.unsafeCast<String>()
fun usingAsOperator(s: Any) = s as String
```
</div>

They will be compiled accordingly:
<div class="sample" markdown="1" theme="idea" mode="java">

``` javascript
function usingUnsafeCast(s) {
    return s;
}

function usingAsOperator(s) {
    var tmp$;
    return typeof (tmp$ = s) === 'string' ? tmp$ : throwCCE();
}
```

</div>