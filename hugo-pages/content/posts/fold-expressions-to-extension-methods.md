---
title: "Fold Expressions for Metaprogramming or \"The Wrong Reason to Rightfully Want Extension Methods in C++\""
date: 2023-04-22T01:27:54+03:00
draft: false
---

*__Disclaimer:__ when writing code examples for that article, I wasn't really concerned with passing/returning references where it might make actual sense in real scenarios.
All examples here are for demonstration, only to provide you with an idea of the tricks that could be used to write actual code.
I have personally encountered cases where snippets written naively (by me), like the ones in this article, would break, as a result of being used on non-copyable types.
Still, I consider the basic concepts and ideas described here worth sharing.*

## Toy problem

Consider the following problem: you are asked to write a function that will accept an arbitrary number of arguments of arbitrary types satisfying the `std::totally_ordered` concept.
The function must return an `std::tuple` with max argument passed for each type.

Don't try to write a complete implementation for it yet (we will write it later).
I want you to start with performing another exercie: computing a return type for such function: for a black-box function
```c++
template <std::totally_ordered... Types>
SomeStdTupleInstance f(Types...) {
    /* The implementation we don't care about for now */
}
```
we need to write `SomeStdTupleInstance`.
How do we do that?

## Toy solution

The first thing that comes to mind (for me it used to be that, at least) is to make some kind of recursive template.
We will go over all of the types passed to us and conditionally append them to the accumulator:
```c++
// Declare the template
template <typename Tuple, typename... Types>
struct TupleSet {};

// Specialize for our case: carry our 'return value'
// in the first template argument, and all the info
// to process in the rest
template <typename Type1, typename... Types, typename... TupleArgs>
struct TupleSet<std::tuple<TupleArgs...>, Type1, Types...> {
    using Type = TupleSet<
        std::conditional_t<
            (std::same_as<TupleArgs, Type1> || ... || false),
            // If `Type1` is already in the tuple, don't add it
            std::tuple<TupleArgs...>,
            // Else append it to the end of the tuple
            std::tuple<TupleArgs..., Type1>
        >, Types...
    >::Type;
};

// Terminating specialization: nothing left to process
template <typename... TupleArgs>
struct TupleSet<std::tuple<TupleArgs...>> {
    using Type = std::tuple<TupleArgs...>;
};
```

Now, by adding
```c++
template <typename... Types>
usign TupleSetT = typename TupleSet<std::tuple<>, Types...>;
```
we have a nice working piece of code.
Indeed,
```c++
static_assert(std::same_as<TupleSetT<int, float>, std::tuple<int, float>>);
static_assert(std::same_as<TupleSetT<int, float, int>, std::tuple<int, float>>);
```
compiles and we can now declare our function as
```c++
template <std::totally_ordered... Types>
TupleSetT<Types...> f(Types...);
```

Not bad at all (it is pretty bad, IMO).
But we can also do better.

## One man's trash

In the previous solution, we had to write `struct TupleSet` three times!
Writing so much code that does nothing is totally unacceptable.
It is time to unveil the trick:
```c++
// Overload operator+
// Adding a `Type` to `std::tuple` procuces a new tuple
// with `Type` as the last argument if `Type` was not in
// tuple's arguments. The return type is the same as the
// original tuple otherwise
template <typename Type, typename... Types>
std::conditional_t<
    (std::same_as<Types, Type> || ... || false),
    std::tuple<Types...>, std::tuple<Types..., Type>
> operator+(std::tuple<Types...>, Type);
// Note that no definition is provided for this overload
// There's not going to be one: we only need this for type
// deduction and we don't want it to be called from the
// acutual running code.

// Use C++17 fold expressions to bypass writing a recursive template
template<typename... Types>
using TupleSetT = decltype((std::declval<std::tuple<>>() + ... + std::declval<Types>()));
```

We did almost twice better in terms of lines of code written!
Readability has decreased, however, but only so slightly: I wouldn't call it a dramatic decline since the original code didn't look all that nice either.
Most importantly, this completely replaces the recursive template that needed to be specialized to terminate the sequence: fold expression is automatically expanded to include each of the arguments.
This is a handy little trick to organize recursive process on templates without explicit recursion.

_Now, I will start slowly moving to motivating the necessity for extension methods, or, rather, an upgrade to operators `.` and `->`.
If you only care about the trick, either read the whole thing to learn about its problems, or go right to the [Appendix](#appendix-more-toy-examples) for other examples of using it._

## Air pollution

What I've just described is great for toying around, but how will we go about it in a real codebase?
We'll probably have to implement it (or something alike) for a custom template rather than `std::tuple`, or (why not?!) for an arbitrary template.
Also, we wouldn't want this to prevent us from using `operator+` (or whatever other operator we've picked) in a meaningful way on our types (or any arbitrary type, for that matter!).
For example, we might want to add a value of some type to our container, but we wouldn't be able because it still will fall into our `operator+` overload.

What do we do?
Naturally, we need to make the trick `operator+` our little implementation secret.
Idea: we'll declare it the `detail` namespace and only use it in there!

```c++
namespace detail {
    template <typename Type, typename... Types>
    std::conditional_t<...> operator+(std::tuple<Types...>, Type);
}
```

But we want to use it from the outside.
How?
It would be great if we had some way to specify the namespace for the infix operator like

```c++
// Looks ugly, but will do for a niche application
std::tuple<>{} detail::+ int{}
```

But we don't.

Well, we could access it directly as a function via `detail::operator+()` but that defeats the entire purpose of being able to use it in a fold expression.
Our only viable option is to declare everything that uses our hidden overload inside of the `detail` namespace as well, and either 'export' it into parent namespace via `using` or to declare a separate (properly named and accessible!) helper inside of `detail`.

```c++
namespace detail {
template<typename... Types>
using TupleSetT = decltype((std::declval<std::tuple<>>() + ... + std::declval<Types>()));
}
using detail::TupleSetT;
```

Phew, this doesn't look all that bad.
Unless we wanted to use the `operator+` directly inside of a class.
We can't -- there is no way to import this operator overload to be accessible at class scope and at class scope only!
We will still have to declare a named helper for this, even if we don't want to.

And what if there is a winning specification for one of the templates already somewhere ([like this example](https://godbolt.org/z/qK5f6xYqK))?
What do we do?

## Another man's treasure

As promised, let's take a step back and see how we could actually implement our `f()`.
It's relatively easy if we give a meaning to our already existing `operator+`:

```c++
namespace detail {
    template <std::totally_ordered Type, std::totally_ordered... Types>
    constexpr auto operator+(std::tuple<Types...> tup, Type t) {
        if constexpr ((std::same_as<Types, Type> || ... || false)) {
            auto& tupVal = std::get<Type>(tup);
            if (tupVal < t) tupVal = t;
            return tup;
        } else {
            return std::make_tuple(std::get<Types>(tup)..., t);
        }
    }
}

template <std::totally_ordered... Types>
constexpr auto f(Types... args) {
    using detail::operator+; // Thankfully, available at function-scope
    return (std::tuple<>{} + ... + args);
}

static_assert(f(1, 2, 2.3, 5, 'c', 1.2) == std::make_tuple(5, 2.3, 'c'));
```

Thanks to the fact that our functions aren't dummies anymore, we can get rid of `decltype` and `std::declval`.

Overall, this solution looks relatively clean.
But if we try to modify it somehow to accept, let's say, some generic class template that is tuple-like, the problem of a possible winning meaningful specialization of `operator+` (among other previously described and undescribed) arises.
Sure, we can always try another binary operator, but what if _all_ binary operators are defined for a certain template (unlikely, so here's a more reasonable concern: what if we aren't sure _what_ binary operators might be defined for a certain template)?
And what the hell is, for example, `std::tuple<>{} ^ ... ^ args` even supposed to mean from the readability standpoint?

Enter extension methods.
Well, not quite yet: they're not in C++, but we can get a pretty good idea of that they might look like from [deducing this](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p0847r6.html).
For our li'l old `std::tuple` we would probably write an extension method as somethings like this:
```c++
template <std::totally_ordered Type, std::totally_ordered... Types>
constexpr auto pushElementOfUniqueType(this std::tuple<Types...> tup, Type t) {
    if constexpr ((std::same_as<Types, Type> || ... || false)) {
        auto& tupVal = std::get<Type>(tup);
        if (tupVal < t) tupVal = t;
        return tup;
    } else {
        return std::make_tuple(std::get<Types>(tup)..., t);
    }
}
```

This is functionally identical, and far more readable and expressive then `+`.
You can document it, and you don't necessarily have to hide it under the carpet of `detail` namespace.

There is, however, one catch.
To be able to do what we want we would need the ability to chain method/extension method calls via a fold expression somehow (we could already use fold expressions with `.*` and `->*` operators, but I struggle to find any way to use this for our purpose).
```c++
// Just dreaming
template <std::totally_ordered... Types>
constexpr auto f(Types... args) {
    return std::tuple<>{}. ...pushElementOfUniqueType(args);
    // or, maybe, even with a namespace!
    return std::tuple<>{}. ...detail::pushElementOfUniqueType(args)?!
}
```

We would be able to provide the interface that we are almost 100% sure does not interfere with any existing code (precisely 100% if we would be able to add a namespace specifiaction to the call).

## No, we have UFCS at home!

Dreaming is nice, but what can we do today?
Well, while there is nothing even resmbling UFCS in C++ yet, we can go to... operator overloading again.
[We'll do it like here](https://cpptruths.blogspot.com/2017/01/folding-monadic-functions.html), but uglier, picking an operator that's highly unlikely to be used in another context with our arguments.
We get something like this:

```c++
template <typename T, typename Func>
requires std::invocable<Func, T>
constexpr auto operator>>=(T t, Func func) {
    return func(t);
}

constexpr auto f(auto... args) {
    constexpr auto op = [] <typename Type> (Type arg) {
        return [arg] <typename... Types> (std::tuple<Types...> tup) {
            if constexpr ((std::same_as<Types, Type> || ... || false)) {
                auto& tupVal = std::get<Type>(tup);
                if (tupVal < arg) tupVal = arg;
                return tup;
            } else {
                return std::make_tuple(std::get<Types>(tup)..., arg);
            }
        };
    };
    return (std::tuple<>{} >>= ... >>= op(args));
}

static_assert(f(1, 2, 2.3, 5, 'c', 1.2) == std::make_tuple(5, 2.3, 'c'));
```

We can also try to simplify `f()` even further by creating a UFCS-like interface and defining an `operator>>=` only for it:

```c++
template <typename Callable, typename... Args>
class CallPromise {
    std::tuple<Args...> args;
    Callable callable;
public:
    constexpr CallPromise(Callable callable_, Args... args_) : callable(callable_), args(args_...) {}

    template <typename T> requires std::invocable<Callable, T, Args...>
    constexpr auto operator()(T t) {
        return std::apply([this, t] (Args... targs) { return this->callable(t, targs...); }, args);
    }
};

template <typename T, typename Callable, typename... Args>
requires std::invocable<CallPromise<Callable, Args...>, T>
constexpr auto operator>>=(T t, CallPromise<Callable, Args...> promise) {
    return promise(t);
}

// Can't pass function template as an agrument - hence the lambda
constexpr auto pushElementOfUniqueType =
    []
    <std::totally_ordered Type, std::totally_ordered... Types>
    (std::tuple<Types...> tup, Type t) {
        if constexpr ((std::same_as<Types, Type> || ... || false)) {
            auto& tupVal = std::get<Type>(tup);
            if (tupVal < t) tupVal = t;
            return tup;
        } else {
            return std::make_tuple(std::get<Types>(tup)..., t);
        }
    };

constexpr auto f(auto... args) {
    return (std::tuple<>{} >>= ... >>= CallPromise(pushElementOfUniqueType, args));
}

static_assert(f(1, 2, 2.3, 5, 'c', 1.2) == std::make_tuple(5, 2.3, 'c'));
```

This is a lot of code but also arguably the least invasive solution: there are no operator overloads for types other than the ones provided by the interface itself.
In addition to a relatively large boilerplate for such conceptually simple task, there are other drawbacks: this will only work with lambdas (or other functor objects) as function templates cannot be passed directly as an argument to `CallPromise` constructor.
This is a big restiction to the way we interact with this API (and to our ability to construct a better one), dictated only by the lack of UFCS support in the language.

## We've been compromised

Extension methods are an opt-in compromise between UFCS and the complete lack of it.
It affects only user-specified functions, but might be seen by some as possibly breaking APIs.
There is, however, another contender.
[Pipeline-rewrite operator](https://open-std.org/JTC1/SC22/WG21/docs/papers/2020/p2011r0.html) provides a whole new syntax which could affect all free functions (there is nothing about folding in the proposal, so I would _dream_ that it is implied):

```c++
template <std::totally_ordered Type, std::totally_ordered... Types>
constexpr auto pushElementOfUniqueType(std::tuple<Types...> tup, Type t) {
    if constexpr ((std::same_as<Types, Type> || ... || false)) {
        auto& tupVal = std::get<Type>(tup);
        if (tupVal < t) tupVal = t;
        return tup;
    } else {
        return std::make_tuple(std::get<Types>(tup)..., t);
    }
};

constexpr auto f(auto... args) {
    return (std::tuple<>{} |> ... |> pushElementOfUniqueType(args));
}
```

I would still argue that this is less elegant than folding extension methods (also, `|>` would not be callable on the regular class methods), it is, to my liking, one of the nicer solutions to the problem.

_Pipeline-rewrite proposal also explores into the flaws of using `operator|` as a Pipeline Operator._

## One small miracle

As you might've guessed, I really want to see extension methods in C++.
I would also agree for pipeline-rewrite operator.
I might even go crazy and accepts UFCS for all free functions just out of despair.

I see the "perfect" extension methods implementation in C++ Standard (hopefully, some revision before I turn 30) as promotion of operators `.` and `->` to provide pipeline-rewrite syntax to functions declared with "Deducing this"-like syntax and fold expression support.
Depsite the fact that the possible use cases of fold expressions on regular methods might be obscure, I still find it reasonable not to introduce an entire new operator into the language just to do similar job to `.` and `->`, but with less flexibility (`|>` will not allow to extend certain types to be usable in generic functions using member-call syntax).

Aside from the crasy possibilities extension methods/pipeline-rewrite would offer (in the case they get a proper fold expression support), there are also much more grounded uses for them including (but not limited to!) wrapping C APIs, preparing your codebase to simplify future standard library updates (like adding a custom `.contains()` to the string until C++23 becomes usable) and reducing the amount of parenthesis required to write map-reduce (kind of already solved by ranges using... operator overloads).

There are possible problems with this, but, the way I see it, there is no unwritten rule extension methods violate that operator overloads do not already.
Most of them are related to the problematic method/operator being in scopre or not, and are solvable by allowing programmers to explicitly specify from which namesapace the extension/operator should be taken.

If introduced, extension methods, as most of C++, could be a powerful, but also dangerous in inexperienced hands tool that would allow programmers with proper understanding of the mechanism write cleaner, more readable code.

-------

_Dear C++ Santa Comittee!_

_Writes to you a little C++ programmer._
_Even though I come from a place where Christmas is not as widely celebrated and kids write to Granpa Frost instead of Santa, I couldn't think of a pun that would use this._
_Still, New Year and Christmas are pretty close, so why don't I give writing __you__ a shot?_

_The two things that I wish the most for the next C++ standard are fold expressions for object/pointer method calls and extension methods._
_There are a lot of cool features to tackle and want, but I am humble: give me this, and I'll probably be happy for several revisions._

_I wasn't naughty: I respected my supervisor and I've been writing C++._
_I haven't done bad things, oh no-no, I didn't._
_Except for that one time I made changes to a Rust project, but that doesn't count, right?_

_I am 23 year old, but I want to believe in a little CXX-mas miracle._
_Pretty please, Santa? You wouldn't leave a junior dev without a present would you?_

_Sincerely yours, GregTheMadMonk._

-------

## Appendix: More toy examples

These examples, especially [Find `std::tuple`'s Nth argument](#find-tuples-nth-argument-without-stdget), are not very useful for `std::tuple`.
They could be, however, implemented for any other template, where they may turn out to be very handy.

### Reverse tuple arguments

```c++
template <typename Type, typename... Types>
std::tuple<Type, Types...> operator+(std::tuple<Types...>, Type);

template <typename... Types>
auto TupleInvHelper(std::tuple<Types...>)
-> decltype(
    (std::declval<std::tuple<>>() + ... + std::declval<Types>())
);

template <typename TupleType>
using TupleInvT = decltype(TupleInvHelper(std::declval<TupleType>()));

static_assert(std::same_as<TupleInvT<std::tuple<int, char, float>>, std::tuple<float, char, int>>);
```

### Find tuple's Nth argument without `std::get`

```c++
template <std::size_t index, std::size_t find, typename T = void> struct Tag { using Type = T; };

template <std::size_t index, std::size_t find, typename TagT, typename Type>
Tag<index + 1, find, std::conditional_t<index == find, Type, TagT>>
operator+(Tag<index, find, TagT>, Type);

template <std::size_t index, typename... Types>
auto TupleGetHelper(std::tuple<Types...>)
-> decltype(
    (std::declval<Tag<0, index>>() + ... + std::declval<Types>())
);

template <std::size_t index, typename TupleType>
using TupleGetT = typename decltype(TupleGetHelper<index>(std::declval<TupleType>()))::Type;

static_assert(std::same_as<TupleGetT<0, std::tuple<int, float, char>>, int>);
static_assert(std::same_as<TupleGetT<1, std::tuple<int, float, char>>, float>);
static_assert(std::same_as<TupleGetT<2, std::tuple<int, float, char>>, char>);
```

_Thanks to u/scatters for continuing the discussion on this topic 4 months after the original post._
