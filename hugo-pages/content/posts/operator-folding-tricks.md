---
title: "Fold expressions for metaprogramming or \"The wrong reason to rightfully want extension methods in C++\""
date: 2023-04-22T01:27:54+03:00
draft: true
---

## Toy problem

Consider the following problem: you are asked to write a function that will accept an arbitrary number of arguments of arbitrary types satisfying the `std::totally_ordered` concept.
The function must return an `std::tuple` with max argument passed for each type.

Disclamer: we will not write this function for now.
All that's of the intereset for the purpose of the beginning this article is the return type of that function.
In short, for
```c++
template <std::totally_ordered... Types>
SomeStdTupleInstance f(Types...) {
    /* The implementation we don't care about */
}
```
we want to create an `std::tuple` that will contain every type from `Types` once and only once.
How do we do that?

## Toy solution

The first thing that comes to mind (for me it used to be that, at least) is to make some kind of recursive template.
We will go over all of the types passed to us and conditionally append them to the accumulator:
```c++
template <typename Tuple, typename... Types>
struct TupleSet {};

template <typename Type1, typename... Types, typename... TupleArgs>
struct TupleSet<std::tuple<TupleArgs...>, Type1, Types...> {
    using Type = TupleSet<
        std::conditional_t<
            (std::same_as<TupleArgs, Type1> || ... || false), // If `Type1` is already in the tuple
            std::tuple<TupleArgs...>,                         // Don't add it
            std::tuple<TupleArgs..., Type1>                   // Else append to the end of the tuple arguments
        >, Types...
    >::Type;
};

template <typename... TupleArgs>
struct TupleSet<std::tuple<TupleArgs...>> { // Terminating specialization
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
static_assert(std::same_as<TupleSetT<int, float>, std::tuple<int, float>>);
```
compiles and we can now declare our function as
```c++
template <std::totally_ordered... Types>
TupleSetT<Types...> f(Types...);
```

But we can also do better.

## Toy solution number two

In the previous solution, we had to write `struct TupleSet` three times!
This is unacceptable and unreadable.
We can do much better:
```c++
template <typename Type, typename... Types>
std::conditional_t<
    (std::same_as<Types, Type> || ... || false),
    std::tuple<Types...>, std::tuple<Types..., Type>
> operator+(std::tuple<Types...>, Type);

template<typename... Types>
using TupleSetT = decltype((std::declval<std::tuple<>>() + ... + std::declval<Types>()));
```

We did almost twice better in terms of lines of code written!
Readability has decreased, however, I wouldn't call it a dramatic decline since the original code didn't look all that nice either.
Most importantly, it completely replaces the recursive template that needed to be specialized to terminate the sequence: fold expression automatically has the length corresponding to the used parameter pack and needs no such tricks.
This is a handy little trick to organize recursive process on templates without explicit recursion.

For more examples of this trick, go to [Appendix 1](#appendix-1-more-toy-examples).

## Air pollution

What I've just described is good and all for toying around, but how will we go about it in a real codebase?
We probably will have to implement some trick for a custom template rather than `std::tuple`, or (why not?!) for an arbitrary template.
But now we wouldn't want our sneaky trick to prevent us from using `operator+` (or whatever else we've used) in a meaningful way on our types (or any arbitrary type, for that matter!).
For example, we might want to add a value of some type to our container, but we wouldn't be able because it still will fall into our `operator+` overload.

What do we do? Naturally, we need to make the trick `operator+` our little implementation secret.
Idea: we'll declare it the `detail` namespace and only use it in there!

```c++
namespace detail {
    template <typename Type, typename... Types>
    std::conditional_t<...> operator+(std::tuple<Types...>, Type);
}
```

How do we use it now?
It would be great if we had some way to specify the namespace for the infix operator like

```c++
std::tuple<>{} detail::+ int{}
```

But we don't.

Well, we could access it directly as a function via `detail::operator+()` but that defeats the entire purpose of being able to use it in a fold expression.
Our only reasonable option is to declare everything that uses it inside of `detail` namespace as well, and either provide it into parent namespace via `using` or to declare a separate helper.

```c++
namespace detail {
template<typename... Types>
using TupleSetT = decltype((std::declval<std::tuple<>>() + ... + std::declval<Types>()));
}
using detail::TupleSetT;
```

This doesn't look all that bad.
Unless, we wanted to use the `operator+` directly inside of a class. We can't -- there is no way to import this operator overload to be accessible at class scope and at class scope only!
We will still have to declare a named helper for this, even if we don't want to.

And what if there is a winning specification for one of the templates already somewhere ([like this example](https://godbolt.org/z/qK5f6xYqK))?
What do we do?

## I just want extensions. Please!

Let's take a step back and see how we could actually implement our `f()`.
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
But if we try to modify it somehow to accept, let's say, some generic class template that is tuple-like, the problem of a possible winning meaningful specialization of `operator+` arises.
Sure, we can always try another binary operator, but what if _all_ binary operators are defined for a certain template?
And what the hell is, for example, `std::tuple<>{} ^ ... ^ args` even supposed to mean from the readability standpoint.

Enter extension methods.
Well, not quite yet: they're not in C++, but we can get a pretty good idea of that they might look like from [deducing this](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p0847r6.html).
For our li'l old `std::tuple` we would probably write an extension method as somethings like this:
```c++
template <std::totally_ordered... Types>
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

In my humble opinion, `tup.pushElementOfUniqueType(2)` is more meaningful than `tup + 2`.
You can document it, and you don't necessarily have to hide it under the carpet of `detail` namespace.

There is, however, one catch.
To be able to do what we want we would need the ability to chain method/extension method calls via a fold expression somehow (we could already use fold expressions with `.*` and `->*` operators, but I struggle to find any way to use this for our purpose).
```c++
template <std::totally_ordered... Types>
constexpr auto f(Types... args) {
    return std::tuple<>{}. ...pushElementOfUniqueType(args); // or, maybe, even ...detail::pushElementOfUniqueType(args)?!
}
```

We would be able to provide the interface that we are almost 100% sure does not interfere with any existing code (precisely 100% if we would be able to add a namespace specifiaction to the call).

## Appendix 1: More toy examples

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
