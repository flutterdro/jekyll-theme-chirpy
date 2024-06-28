---
layout: post
title: Overengineering a simple problem using expression templates
---
Have you ever encountered a problem which can be done manually in 10 mins, 
but when you sit down to do it, you suddenly feel something is amiss?
And 10 min job suddenly turns into an hour of beautiful agony and 5 mins of 
actually getting the work done.  
After that you have the urge to design a generic solution which will turn that
10 min job into an instant one. 

I did...

## 2 days ago...

The exact problem I faced requires a bit of introduction and explanation, so instead
I will introduce a toy problem which in its essense is the same. 

Now, imagine you work for a shop that sells christmas lights. Your task is to group different
kinds of lights based on their properties. Now, also imagine that your team lead 
was obsessed with data oriented design, like, swiftie level of obsessed. So he forces everything
to be an array. As the result, each group has to be found by indexing into the array.

Your job is to turn those different properties (which we will also call options) into a unique
index.

Let's start with a simpler version.

## All options are independent

What does that mean? It is really simple.

Assume we have 2 options. First, our lights can be different colors:
{% highlight cpp %}
enum class color {
    red,
    green,
    blue,
    yellow,
};
{% endhighlight %}

Second, our lights can be different shapes:
{% highlight cpp %}
enum class shape {
    star,
    sphere,
    strawberry,
};
{% endhighlight %}

Those two options can be called independent if for each shape we can use every colors.

It is easy to see that in this case the total amount of different lights
is 3 shapes * 4 colors = 12. It is also easy to map every combination to index:
```cpp
constexpr auto index(color color, shape shape) -> std::size_t {
    return 4 * static_cast<std::size_t>(shape) + static_cast<std::size_t>(color);
}
// it will map like this:
// 0 - {star,       red}, 1 - {star,       green}, 2  - {star,       blue}, 3  - {star,       yellow},
// 4 - {sphere,     red}, 5 - {sphere,     green}, 6  - {sphere,     blue}, 7  - {sphere,     yellow},
// 8 - {strawberry, red}, 9 - {strawberry, green}, 10 - {strawberry, blue}, 11 - {strawberry, yellow},
```

You can see `static_cast<std::size_t>` is used. So it is reasonble to ask: why not just use a plain
`enum` instead of `enum class`. The reason is simple, I like my types like I like my coffee:
strong and without namespace pollution.

Anyway, back to out chickens. There is a small trick that you can use with enums that can 
help us get rid of that magic 4.

{% highlight cpp %}
enum class color {
    red,
    green,
    blue,
    yellow,

    count,
};
{% endhighlight %}

By adding enum member count at the end, we can count amount of names in the enum. We 
will add one to every option. Might as well define this little helper:
```cpp
template<typename T>
inline constexpr std::size_t enum_limit = static_cast<std::size_t>(T::count);
```
Using it, our function will look like:
```cpp
constexpr auto index(color color, shape shape) -> std::size_t {
    return enum_limit<color> * static_cast<std::size_t>(shape) + static_cast<std::size_t>(color);
}
```
This version is better than the previous one, since if we were to add a new color,
our mapping will adjust accordingly.

Robust, flexible, simple and readable. What else can we possibly need?

### MOAR options

Now we have to deal only with `shape` and `color`, but what if we add another one?
Like option that defines whether lights are dimmable or not.
```cpp
enum class dim {
    off,
    on,

    count,
};
```
Amount of different kinds of christmas lights just doubled. And mapping function now looks
like this:
```cpp
constexpr auto index(color color, shape shape, dim dim) -> std::size_t {
    return enum_limit<color> * enum_limit<shape> * static_cast<std::size_t>(dim) +
    enum_limit<color> * static_cast<std::size_t>(shape) + 
    static_cast<std::size_t>(color);
}
```
And what if we want to add another one? And one more? It will quickly get out of hand.

But what else can we do?

### Expression templates
Hell yeah! We are pulling out big guns for a minor inconvinience.
That's what c++ is all about!

If you don't know what expression templates are, then here is a simple
example:
```cpp
struct my_num {
    constexpr auto calculate() const noexcept {
        return m_num;
    }
    int m_num;
};
template<typename T1, typename T2>
struct add {
    constexpr auto calculate() const noexcept {
        return m_operand1.calculate() + m_operand2.calculate();
    }
    T1 m_operand1;
    T2 m_operand2;
};

template<typename T1, typename T2>
constexpr auto operator+(T1 operand1, T2 operand2) {
    return add{operand1, operand2};
}

auto main() -> int {
    auto num1 = my_num{0};
    auto num2 = my_num{1};
    auto num3 = my_num{2};

    auto delayed_calculation = num1 + num2 + num3;
}
```
You may ask. All of this vodoo to just add 3 numbers?  
And I will answer. Huhuhu, all of this vodoo was used to NOT add 3 numbers. 

You see, the actual addition was never performed. We only constructed the 
expression itself. To calculate the value of the expression we need to call
`.calculate()`. 

Yes! This is the way to bring lazy calculation to c++. But this is not all.
Depending on the way you design these four points:
- handling of the basic operand (in our example it is my_num struct)
- expression classes (in our example it is add struct)
- expression composing (in our example it is operator+ overload)
- expression collapsing (in our example it is calculate member)

you can achieve many great things, including, but not limited to:
- lazy evaluation
- complete elimination of temporaries
- ability to pass expressions as functions

Expression templates are great, powerful, flexible, whatever else you want them
to be. Because of this they are extensively used in different math libraries. If you
want to know more about them, then google is your friend! I am pretty sure there will be 
a metric ton of blog posts about them.

Now, throw all of that out of the window. We will not use any of that. We will 
only use them to create a syntactic sugar.

### Game plan 

Our end goal is to make something that looks like that:
```cpp
constexpr auto index(color color, shape shape, dim dim) -> std::size_t {
    return (color * shape * dim).collapse();
}
```
We will word from top to bottom. So our next step is to define `operator*`
which is pretty simple:
```cpp
namespace option_collapsing {
typename<typename T1, typename T2>
constexpr auto operator*(T1 option1, T2 option2) {
    return independent{option1, option2};
}
} // namespace option_collapsing
```
We don't want a rogue `operator*` in the global namespace to grab everything
it sees, so we will hide it behind a namespace, as well as everything that
will be written next(though it will be ommited in the snippets).

So how will that `independent` look like? Something like this:
```cpp
template<typename T1, typename T2>
struct independent {
    constexpr auto collapse() {
        // ???
    }
    T1 m_option1;
    T2 m_option2;
};
```
To understand how that collapse function should look like, we should look
at our _naive_ solutions for 2 and 3 options.

```
(color, shape).collapse() -> enum_limit<color> * static_cast<std::size_t>(shape) + static_cast<std::size_t>(color)

(color, shape, dim).collapse() -> enum_limit<color> * enum_limit<shape> * static_cast<std::size_t>(dim) +
    enum_limit<color> * static_cast<std::size_t>(shape) + 
    static_cast<std::size_t>(color)
```
There is a lot of clutter in the form of `static_cast` and `enum_limit`, so we will replace them
with pseudo member functions `.collapse()` and `.limit()` respectively.
```
(color, shape).collapse() -> color.limit() * shape.collapse() + color.collapse()

(color, shape, dim).collapse() -> color.limit() * shape.limit() * dim.collapse() +
    color.limit() * shape.collapse() + color.collapse() ->
    color.limit() * shape.limit() * dim.collapse() + (color, shape).collapse() ->
    (color, shape).limit() * dim.collapse() + (color, shape).collapse()
```
That `(color, shape).limit()` is amount of different combinations of different colors and shapes.
So we found a pattern. And we get:
```cpp
template<typename T1, typename T2>
struct independent {
    static constexpr auto limit() {
        return m_option1.limit() * m_option2.limit();
    }
    constexpr auto collapse() {
        return m_option1.limit() * m_option2.collapse() + m_option1.collapse();
    }
    T1 m_option1;
    T2 m_option2;
};
```
And don't worry, you can call static functions like regular member functions. 

Though I recently saw an interesting trick here. Instead of using a static function
we can use:
```cpp
static constexpr auto limit = std::integral_constant<std::size_t, m_option1.limit() * m_option2.limit()>{};
```
It will still allow us to call `.limit()` to get the value of the limit since 
`std::integral_constant` has `operator()` which return underlying value. It also allows
us to do classical type metaprogramming.  
It is not like I really need it in this case, but... come on, look at how ugly it is,
you just can't help but love it. This trick is like that one ugly cat, you want
to shower it with all the affection in the world.

### Wrapping up

There is a small detail left. Enums don't have `.collapse()` and `.limit()` functions.
So we have to wrap them:
```cpp
template<auto Val>
using constant = std::integral_constant<decltype(Val), Val>; // alias because this name is too long
template<typename T>
struct limiter {
    static constexpr auto limit = constant<static_cast<std::size_t>(T::count)>{};
    constexpr auto collapse() {
        return static_cast<std::size_t>(m_value);
    }
    T m_value;
};
```
Now, here is the question. At which point should we wrap our enums? And how?

We have 3 options:
- In the very beginning
    ```cpp
    constexpr auto index(color color, shape shape) {
        using namespace option_collapsing; // enable operator*
        return limiter{color} * limiter{shape};
    }
    ```
Straightforward and intuitive solution. But kinda repetetive and the whole point 
is to add syntactic sugar and avoid typing. 

- In the `operator*`  
This one is a bit problematic because we have to chose what needs wrapping.
For this one we will use a little bit of metaprogramming.
    ```cpp
    // concept that detect whether type is already wraped
    template<typename T>
    concept limited = requires(T&& val) {
        {T::limit()}     -> std::same_as<std::size_t>;
        {val.collapse()} -> std::same_as<std::size_t>;
    };
    // if our type is already wrapped - do nothing
    // otherwise - wrap in limiter<T>
    template<typename T>
    using wrap_or_t = std::conditional_t<limited<T>, T, limiter<T>>;

    template<typename T1, typename T2>
    constexpr auto operator*(T1 option1, T2 option2) {
        return independent<wrap_or_t<T1>, wrap_or_t<T2>>{option1, option2};
    }
    ```

    And for it to work we need to add a converting constructor 

    ```cpp
    template<typename T>
    struct limiter {
        //explicit(false) is redundant but it helps to convey the intent
        explicit(false) constexpr limiter(T value) : m_value{value} {}
        // rest of the class 
        // ...
    }
    ```

Second option will work... And it is probably what we need, but there is one trick
which I discovered accidentally. It is pretty cool but I am not sure whether
it is legal.
#### Wrapping with CTAD
Now, let's rollback `operator*` a little:
```cpp
template<typename T1, typename T2>
constexpr auto operator*(T1 option1, T2 option2) {
    return independent{option1, option2};
}
```
In this place we rely on something called a Class Template Argument Deduction(CTAD).
Compiler sees that the type of `option1` is `T1` and the type of `option2` is `T2`, 
so it replaces `independent{option1, option2}` with `independent<T1, T2>{option1, option2}`.

What if we constrained template types of `independent` like this:
```cpp
template<typename T1, typename T2>
    requires (limited<T1> && limited<T2>)
struct independent { /* ... */ };
```
It will try to use the default deduction guide and guess `independent<T1, T2>` which
will then fail the substitution. Now, if we introduce our custom deduction guide:
```cpp
template<typename T1, typename T2>
independent(T1, T2) -> independent<wrap_or_t<T1>, wrap_or_t<T2>>;
```
After first substitution failure, it wil attempt our guide and using it,
it will guess `independent<wrap_or_t<T1>, wrap_or_t<T2>>` which will satisfy
constraints and will wrap our options if needed.

It does work with clang and gcc(I couldn't care less about msvc so I didn't test with it).
I have no clue why it works and if it is even legal, but it is pretty cool. So I will use 
it.

### Intermediate result
```cpp
namespace option_collapsing {
template<auto Val>
using constant = std::integral_constant<decltype(Val), Val>;

template<typename T>
concept limited = requires(T&& val) {
    {T::limit()}     -> std::same_as<std::size_t>;
    {val.collapse()} -> std::same_as<std::size_t>;
};

template<typename T>
using wrap_or_t = std::conditional_t<limited<T>, T, limiter<T>>;

template<typename T>
struct limiter {
    explicit(false) constexpr limiter(T value) : m_value{value} {}
    static constexpr auto limit = constant<static_cast<std::size_t>(T::count)>{};
    constexpr auto collapse() const noexcept {
        return static_cast<std::size_t>(m_value);
    }
    T m_value;
};
template<typename T1, typename T2>
    requires (limited<T1> && limited<T2>)
struct independent {
    static constexpr auto limit = constant<T1::limit() * T2::limit()>{};
    constexpr auto collapse() const noexcept
        -> std::size_t { 
        return m_value1.limit() * m_value2.collapse() + m_value1.collapse(); 
    }

    T1 m_value1;
    T2 m_value2;
};
template<typename T1, typename T2>
independent(T1, T2) -> independent<wrap_or_t<T1>, wrap_or_t<T2>>;

template<typename T1, typename T2>
constexpr auto operator*(T1 option1, T2 option2) {
    return independent{option1, option2};
}

}
```

## Dependent options
If I had to deal only with independent options I wouldn't have fallen victim to 
primal urge to overengineer a simple problem. And if I did, there were certainly
easier ways to do so. 

The issue is with what I call dependent options. It is easier to show an example
to illustrate what they are and how they can be frustrating

Say we have 2 different manufacturers of christmas lights:
```cpp
enum class manufacturer {
    grinch,
    santa, 

    count,
};
```

Each of them customize different aspects. For example santa can produce christmas lights
in different shapes, but grinch can produce only one. On the other hand grinch can make 
christmas lights in different sizes, while santa's have a fixed size.
```cpp
enum class size {
    small,
    big,
    
    count,
};
enum class shape {
    star,
    strawberry,
    sphere,

    count,
};
```

Creating a mapping in this case is a bit tedious:
```cpp
constexpr auto index(manufacturer manufacturer, size size, shape shape) {
    switch(manufacturer) {
        case manufacturer::grinch: {
            return static_cast<std::size_t>(size);
        }
        case manufacturer::santa: {
            return enum_limit<size> + static_cast<std::size_t>(shape);
        }
    }
}

// 0 - {grinch, small}, 1 - {grinch, big}
// 2 - {santa, star}, 3 - {santa, strawberry}, 4 - {santa, sphere}
```
And if the amount of options grows it will _really_ get out of hand. I mean 
imagine possible different combinations of dependent and independent options.

That's why I chose expression templates for it, they are just the right tool
to express all of those complex combinations.

### Game plan mk. 2 

So once again we will do it from top to bottom.
Out endgoal:

```cpp
constexpr auto index(manufacturer manufacturer, size size, shape shape) {
    return manufacturer % (size, shape);
}
```
`(size, shape)` declares an option `bundle`, where first element of the bundle
corresponds to the first enum member of corresponding dependent option.

Next is `operator%` which looks like this:
```cpp
template<typename T, typename... Ts>
constexpr auto operator%(T option, bundle<Ts...> option_bundle) {
    return dependent{option, option_bundle};
}
```

`bundle` class has one simple responsibility and it is to wrap `Ts...` in 
limiter.

```cpp
template<typename... Ts>
    requires (limited<Ts> and ...)
struct bundle {
    constexpr bundle(Ts... elems) : m_elems{elems...} {}
    std::tuple<Ts...> m_elems;
};
template<typename... Ts>
bundle(Ts...) -> bundle<wrap_or_t<Ts>...>;
```

`dependent` class is a little bit nastier, but the idea is similar to 
`independent`

```cpp
template<typename T1, typename T2>
struct dependant {
    // valid in c++23 as of CWG 2518
    static_assert(false, "No bueno");
};
template<typename T, typename... Ts>
    requires (limited<T>)
struct dependant<T, bundle<Ts...>> {
    static_assert(T::limit() == sizeof...(Ts),
        "Bundle must have a match for every option value");
    static constexpr auto limit = constant<(Ts::limit() + ...)>{};

    constexpr auto collapse() const noexcept {
         // ???
    }
    T m_value;
    bundle<Ts...> m_bundle;
};
template<typename T1, typename T2>
dependant(T1, T2) -> dependant<wrap_or_t<T1>, T2>;
```
The idea behind `.collapse()` function is actually simpler then in `independent`'s case.

If `m_value` is the first enum member then we just collapse the first element in the bundle.  
If `m_value` is the second enum member then we add offset in the form of limit of the first
element in the bundle and then collapse the second element in the bundle. 
And so on and so forth.

Even if the idea is simple, implementation in todays c++ is nasty. But if P1306 goes through,
collapse function would look like this:

```cpp
template<typename T, typename... Ts>
    requires (limited<T>)
struct dependant<T, bundle<Ts...>> {
    // ...
    constexpr auto collapse() const noexcept {
        auto index  = std::size_t{0};
        auto result = std::size_t{0};
        // template for allows us to iterate(kinda) through 
        // the elements of the tuple
        template for (auto elem : m_bundle) {
            if (m_value.collapse() > index) {
                sum += elem.limit();
            } else {
                sum += elem.collapse();
            }
        }
    }
    T m_value;
    bundle<Ts...> m_bundle;
};
```
Unfortunately we can't do that now. **BUT**

> Nothing is impossible in c++ as long as you don't care how ugly
> your code ends up looking.  
-- some wise dude from stackoverflow

I also like to add this:

> Nothing is impossible in c++ as long as you don't care how ugly
> your code ends up looking and how long it takes to compile.

So lo and behold. The ugly:
```cpp
template<typename T, typename... Ts>
    requires (limited<T>)
struct dependant<T, bundle<Ts...>> {
    // ...
    constexpr auto collapse() const noexcept {
        auto result = std::size_t{0};

        [&]<std::size_t... Is>(std::index_sequence<Is...>) {
            auto collapser = [&](auto element, std::size_t index) {
                if (m_value.collapse() > index) {
                    result += element.limit();
                    return true;
                } else {
                    result += element.collapse();
                    return false;
                }
            };
            // using short-circuiting property of the and ... fold to 
            // get early returns.
            (collapser(std::get<Is>(m_bundle.m_elems), Is) and ...);
        }(std::make_index_sequence<T::limit()>{});

        return result;
    }
    T m_value;
    bundle<Ts...> m_bundle;
};
```
That's about it. Well ther is that cool `(size, shap)` syntax 
but that's just `operator,` overload.
```cpp
template<typename T1, typename T2>
constexpr auto operator,(T1 option1, T2 option2) {
    return bundle{option1, option2};
}
template<typename T, typename... Ts>
constexpr auto operator,(bundle<Ts...> option_bundle, T option) {
    return [&]<std::size_t... Is>(std::index_sequence<Is...>){
        return bundle{std::get<Is>(option_bundle.m_elems)..., option};
    }(std::index_sequence_for<Ts...>{});
}
template<typename T, typename... Ts>
constexpr auto operator,(T option, bundle<Ts...> option_bundle) = delete;
```

## It is actually useful?!
We are done. Now we can compose arbitrary[^1] complex expressions. For example this one
comes from my actual project:

```
write_back * direction * indexing * (data_size % (
    immediate_operand % (shift, dummy), 
    immediate_operand * signedndness, 
    immediate_operand * signedndness,
))
```
Imagine doing this by hand! It would take like 10 minutes! My heart can't handle that.

Jokes aside. There is an actual reason for all of this. 

See, it is pretty easy to write functions that maps options to indices, but what about 
the opposite. Having index can we find options? 

It is indeed possible, and I need it. It is easy to do manually in case of simple expressions, 
for example, to find independent options from index we just need to solve:
```
index = option1.limit() * option2 + option1
where option1 and option2 are natural numbers.

Solution is just 
option1 = index % option1.limit()
option2 = idnex / option1.limit()
```
The case for dependent options is also simple but a bit more verbose.
But what about expressions like in my project? That would be a nightmare to figure out 
reversal algorithm. 

And here comes my approach. Since our big and scary expression is a combination
of smaller and simpler ones, we can recursively call a static method `decompose()`
until we find one of those simple expressions and extract options from it.
We just need to define this method for a few classes and we get decomposing for free!

```cpp
// This class is the most basic part of out expression 
// it contains just the options value.
template<typename T> 
struct limiter {
    static constexpr auto decompose(std::size_t composed) 
        -> std::tuple<T> {
        return {static_cast<T>(composed)};
    }
    // rest of the class
};

template<typename T1, typename T2>
struct independent {
    static constexpr auto decompose(std::size_t composed) {
        auto lhs = T1::decompose(composed % T1::limit());
        auto rhs = T2::decompose(composed / T1::limit());
        return std::tuple_cat(lhs, rhs);
    }
    // rest of the class
};

template<typename T, typename... Ts>
struct dependent {
    static constexpr auto decompose(std::size_t composed) {
        return /*oops*/ 
}
```
Oops. Even though principle behind this kind of decomposing of dependent options
is simple, there is one imortant question which needs an answer. 

What is the return type? 
 
To highlight the issue let's return to the santa and grinch example.
```
0 - {grinch, small}, 1 - {grinch, big}
2 - {santa, star}, 3 - {santa, strawberry}, 4 - {santa, sphere}
if we get for example index = 1, we can easily determine that options are
std::tuple{manufacturer::grinch, size::big}
if we get index 3 we also can easily determine that options are
std::tuple{manufacturer::santa, shape::strawberry}
```
Those two are different types and we can't return 2 different types from the 
same function. So we need to choose some common type, which will be able to represent
both cases.
So how can we choose a proper return type for all of the cases? 

One way is to keep track of all options involved while we construct an expression. And when 
we need to decompose something, we only set the relevant options from the list of options.

Small note: I pulled in boost.mp11 because, my god, it is so annoying to reinvent the wheel.
```cpp
using namespace boost::mp11;
// This class is the most basic part of our expression 
// it contains just option's value.
template<typename T> 
struct limiter {
    using options = std::tuple<T>;
    static constexpr auto decompose(std::size_t composed) 
        -> options {
        return {static_cast<T>(composed)};
    }
    // rest of the class
};
template<typename T1, typename T2>
struct independent {
    // mp_append concatenates multiple tuples into one big tuple
    using options = mp_append<typename T1::options, typename T2::options>;
    static constexpr auto decompose(std::size_t composed) 
        -> options {
        auto lhs = T1::decompose(composed % T1::limit());
        auto rhs = T2::decompose(composed / T1::limit());
        return std::tuple_cat(lhs, rhs);
    }
    // rest of the class
};

template<typename T, typename... Ts>
struct dependent<T, bundle<Ts...>> {
    // concatenates tuples into one tuple with unique elements
    using options = mp_set_union<typename T::options, typename Ts::options...>;
    static constexpr auto decompose(std::size_t composed) 
        -> options {
        // default initialize option holder
        auto result = options{};
        auto index  = std::size_t{};
        auto decomposer = [&]<typename U> -> bool {
            // searches for the value of T
            if (U::limit() <= composed) {
                composed -= U::limit();
                ++index;
                return true;
            } else {
                // when we found a proper value of T 
                // set the relevant options in option holder
                auto options_that_matter = U::decompose(composed);
                std::get<0>(result) = static_cast<mp_at_c<options, 0>>(index);

                [&]<typename... Us>(std::tuple<Us...> significant_options) {
                    ((std::get<Us>(result) = std::get<Us>(significant_options)), ...);
                }(options_that_matter);

                return false;
            }
        };

        (decomposer.template operator()<Ts>() and ...);
        return result;
    }
    // rest of the class
};
```
And now testing it out:
```cpp
constexpr auto expression_for(option1 o1, option2 o2, option3 o3) {
    using namespace option_collapsing;
    return o1 % (o2 * o3, o3);
}
constexpr auto test(option1 o1, option2 o2, option3 o3) {
    auto result = expression_for(o1, o2, o3);
    return decltype(result)::decompose(result.collapse());
}
static_assert(
    test(option1::off, option2::on, option3::on) == 
    std::tuple{option1::off, option2::on, option3::on}
);

```
I wouldn't call it free. But it does work for whatever expression I need. And I need a lot of those.

## Encore 
From here on out will be code which I wouldn't use, but it addresses one crucial issue with this 
approach.

### The issue
Let's look at before:
```cpp
constexpr auto index(manufacturer manufacturer, size size, shape shape) {
    switch(manufacturer) {
        case manufacturer::grinch: {
            return static_cast<std::size_t>(size);
        }
        case manufacturer::santa: {
            return enum_limit<size> + static_cast<std::size_t>(shape);
        }
    }
}
```
And after:
```cpp
constexpr auto index(manufacturer manufacturer, size size, shape shape) {
    return manufacturer % (size, shape);
}
```
Oh, and if you forgot. Here are enums:
```cpp
enum class manufacturer {
    santa,
    grinch, 

    count,
};
enum class size {
    small,
    big,
    
    count,
};
enum class shape {
    star,
    strawberry,
    sphere,

    count,
};
```
Do you see the issue? If not, then that's exactly the issue.
`santa` and `grinch` are swapped in here compared to before.
And while our 'before' is fine, because it doesn't care about the 
order of enum members, our 'after' is completely broken right now, 
because it relies heavily on underlying values of enum members.

Order of enum members shouldn't silently break our code. 
To fix this we need to somehow match dependent options to corresponding
enum member. One way is to bundle up a proper enum value with right
option.
```cpp
constexpr auto index(manufacturer manufacturer, size size, shape shape) {
    return manufacturer % (
        match{manufacturer::grinch, size},
        match{manufacturer::santa, shape}
    );
}
```
...

It is a good start, but there is still a room for an error.
For one, nothing will prevent you from writing:
```
manufacturer % (
        match{manufacturer::grinch, size},
        match{manufacturer::grinch, shape}
    );

```
It would be nice if we could check everything at compile time. 
But to do so, we need to somehow have knowledge of every signle match.
### Lifting into a template parameter

We need to lift our `manufacturer::grinch` into the template parameter of the
`match` class and then it will be propagated into the `dependent` class where
we can check whatever we want.

So we want to write something like this:
```cpp
template<auto EnumValue, typename T>
struct match {
    constexpr match(T option)
        : m_value{option} {}
    T m_value;
};
// and use like this
match<manufacturer::grinch>{size};
```
There is just one small issue. You can't do that in c++. You can't have
some parameters deduced and some specified at the same time. It is either
all deduced or none. You can do so in functions tho, and you can try to dance with that:
```cpp
template<auto EnumValue, typename T>
constexpr auto indirect(T option) {
    return match<EnumValue, T>{option};
}
// actually does work
indirect<manufacturer::grinch>(size);
```
There is one more approach which goes all in on ctad:
```cpp
template<typename EnumConstant, typename T>
struct match {
    static constexpr auto enum_value = EnumConstant{};
    constexpr match(EnumConstant, T);
};

// and call like this
match{constant<manufacturer::grinch>{}, size};
// if you forgot 
template<auto Val>
using constant = std::integral_constant<decltype(Val), Val>;
```
We can overload one more operator to make it more concise.
```cpp
template<typename EnumConstant, typename T>
constexpr operator|(EnumConstant e, T option) {
    return match{e, option};
}
```
Now we can do this:
```cpp
manufacturer % (
    constant<manufacturer::grinch>{}| size,
    constant<manufacturer::santa> {}| shape
);
```
I prefer this version.
### Checking
There isn't much difference compared to before in terms of implementation.
So the only 






---
[^1]: Not really arbitrary. For example you can't do `option % (option, other_option)`.
    Like, if an option appears on the left side of `%` or `*` it can't be used anywhere on the right.
    Which isn't really limiting because this kind of stuff doesn't make sense to begin with.
