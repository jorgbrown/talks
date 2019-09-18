name: title-layout
layout: true
class: center, middle, title

---
name: basic-layout
layout: true
class: left, top

---
name: title
template: title-layout

# Reducing Template Compilation Overhead
.footnote[Jorg Brown, <jorg.brown@gmail.com>, [@jorgbrown](https://twitter.com/jorgbrown)]

???
Intro:
- I'm Jorg Brown
- I work with Chandler Carruth on Google's C++ PL platform
- Obsessed with performance and optimization.

I've wanted to have given a talk at CppCon for some time, and finally have one.

---
name: Background
class: middle, center

![First Guess](templates_memory_bender_meme.jpg)

???

---
name: Background
class: left, middle

# Reducing Template Overhead
## First, some background
1. Hard to measure when things are going well
1. BubbleSort is better than you think.
1. Compile-time calculation can be surprising

???
Well-designed O(n) template code is in the noise, as far as overhead goes.
So is O(n^2) overhead, typically.
BubbleSort isn't just OK for low N; it's OPTIMAL up to N=3.

---
### It's hard to measure template overhead...

* Typically the overhead of starting up a compiler and reading in the standard
  headers dwarfs most template instantiations, so measuring total allocations is
  noisy
   * For clang, use -Xclang -print-stats to get fine detail; it even works in
     Compiler Explorer.
   * Dump all types: compile with "-g", then "dwarfdump -debug-types
     -debug-pubtypes -debug-gnu-pubtypes foo.o"

---
### Compile-time basics: functions

```cpp
int Fibonacci(int n) {
  return (n <= 1) ? n : Fibonacci(n - 1) + Fibonacci(n - 2);
}

int main() {
  int fib = Fibonacci(15);
  return fib;
}
```

???
Inefficient at runtime, but compiles quickly.

---
### Compile-time basics: functions

```cpp
*constexpr
int Fibonacci(int n) {
  return (n <= 1) ? n : Fibonacci(n - 1) + Fibonacci(n - 2);
}

int main() {
  int fib = Fibonacci(15);
  return fib;
}
```

---
### Compile-time basics: functions

```cpp
constexpr
int Fibonacci(int n) {
  return (n <= 1) ? n : Fibonacci(n - 1) + Fibonacci(n - 2);
}

int main() {
  `constexpr` int fib = Fibonacci(15);
  return fib;
}
```

---
### Compile-time basics: functions

```cpp
constexpr
int Fibonacci(int n) {
  return (n <= 1) ? n : Fibonacci(n - 1) + Fibonacci(n - 2);
}

int main() {
  // Fails to compile with clang due to "exceeding step limit"
  // Compiles with MSVC but takes 10 seconds; 28 will fail.
  constexpr int fib = Fibonacci(`27`);
  return fib;
}
```

---
### Compile-time basics: templates

```cpp
template<int n>
struct Fibonacci {
  static constexpr int value = n <= 1 ? n : Fibonacci<n - 1>::value +
                                            Fibonacci<n - 2>::value;
};

int main() {
  constexpr int fib = Fibonacci<46>::value;
  return fib;
}
```

???
Note: Specializations for <0> and <1> were necessary, because lazy evaluation for
? : doesn't block template instantiation.

---
### Compile-time basics: templates

```cpp
template<int n>
struct Fibonacci {
  static constexpr int value = n <= 1 ? n : Fibonacci<n - 1>::value +
                                            Fibonacci<n - 2>::value;
};

*template<> struct Fibonacci<0> { static constexpr int value = 0; };
*template<> struct Fibonacci<1> { static constexpr int value = 1; };

int main() {
  constexpr int fib = Fibonacci<46>::value;
  return fib;
}
```

???
Note: Specializations for <0> and <1> were necessary, because lazy evluation for
? : doesn't block template instantiation.

---
### Compile-time basics: templates

```cpp
template<int n>
struct Fibonacci {
  static constexpr int value = [] {
*   if constexpr(n > 1) {
*     return Fibonacci<n - 1>::value + Fibonacci<n - 2>::value;
*   }
    return n;
  }();
};

int main() {
  constexpr int fib = Fibonacci<46>::value;
  return fib;
}
```

???
Note: Specializations for <0> and <1> were necessary, because lazy evluation for
? : doesn't block template instantiation.

---
#### This one weird trick...

```cpp
template<int N>
constexpr auto Fibonacci() {
  if constexpr (N > 1) {
    return std::integral_constant<int, Fibonacci<N-1>() + Fibonacci<N-2>()>{};
  } else {
    return std::integral_constant<int, N>{};
  }
}

int main() {
  constexpr int fib = Fibonacci<46>();
  return fib;
}
```

???
By putting the calculation into a template instantiation, we force the compiler
to do the calculation at compile-time, even in debug mode.

By making the return value part of the returned type, we make it possible to
avoid calling the routine at all.

The function's signature gets smaller because we're returning auto without the
"->"

---
## The overhead of function signatures

```cpp
std::string add1(std::string a, std::string b);



template<typename T>
T add2(T a, T b);



template<typename T>
auto add3(T a, T b) { return a + b; }
```

---
## The overhead of function signatures

```cpp
std::string add1(std::string a, std::string b);

*add1(std::string, std::string)

template<typename T>
T add2(T a, T b);

*std::string add2<std::string>(std::string, std::string)

template<typename T>
auto add3(T a, T b) { return a + b; }

*auto add3<std::string>(std::string, std::string)
```

---
## The overhead of function signatures

```cpp
template<typename T>
T process(T a);

*std::string process<std::string>(std::string)

template<typename T>
typename std::enable_if_t<!std::is_same<T, void>::value, T>
process(T a);

```

---
## The overhead of function signatures

```cpp
template<typename T>
T process(T a);

*std::string process<std::string>(std::string)

template<typename T>
typename std::enable_if_t<!std::is_same<T, void>::value, T>
process(T a);

*std::enable_if<!std::is_same<std::string, void>::value, std::string>::type
*            process<std::string>(std::string)

```

---
## The overhead of function signatures

```cpp
template<typename T>
typename std::enable_if_t<!std::is_same<T, void>::value, T>
process(T a);

*std::enable_if<!std::is_same<std::string, void>::value, std::string>::type
*            process<std::string>(std::string)

template<typename T>
auto process(T a) {
  static_assert(!std::is_same<T, void>::value);
  // ...
}

*auto process<std::string>(std::string)
```

---
name: lessons
template: basic-layout

### Lessons so far

1. 'inline' functions aren't necessarily inlined.
   * 'inline' is only guaranteed to affect linkage.
2. 'constexpr' functions are usually not memo-ized, and not necessarily inlined,
   even if they always produce the same result.
   * 'constexpr' only guarantees that the result can be used in situations where
     only a constant is allowed.
3. 'template' instantiations are guaranteed to be memo-ized
   * ... but their functions are not necessarily inlined.
4. 'if constexpr' and 'auto' return types go well with templated functions.
5. Results may vary by compiler.

---
name: custom tuple
template: basic-layout

## Case study: A custom tuple type.

#### Or "How I learned to write an O(N-cubed) template"

```cpp
template<typename... T> class MyTuple;
```

---
name: custom tuple
template: basic-layout

#### Case study: A custom tuple type.

```cpp
template<typename... T> class MyTuple;

template<typename T>
class MyTuple<T> {
  T first;
  template<size_t N, typename TN> friend struct Getter;
};

template<typename T, typename... More>
class MyTuple<T, More...> {
  T first;
  MyTuple<More...> more;

  template<size_t N, typename TN> friend struct Getter;
};
```
---
name: custom tuple
template: basic-layout

#### Case study: A custom tuple type.

```cpp
template<size_t N, typename TN> struct Getter {
  template<typename T0, typename... MoreN>
  static TN &get(MyTuple<T0, MoreN...> &tup) {
    return Getter<N - 1, TN>::get(tup.more);
  }
};

template<typename T0> struct Getter<0, T0> {
  template<typename... Ts>
  static T0 &get(MyTuple<Ts...> &tup) { return tup.first; }
};

template<size_t N, typename... Ts>
auto get(MyTuple<Ts...> &tup)
                -> typename std::tuple_element<N, std::tuple<Ts...>>::type& {
  return Getter<N, typename std::tuple_element<N, std::tuple<Ts...>>::type>::get(tup);
}
```

???
GodBolt: https://godbolt.org/z/75Qe2Z

---
name: custom tuple
template: basic-layout

#### Results: custom tuple type is O(n^3) in codegen

(As measured by clang -fno-inline and objdump --disassemble)

```cpp
using T = MyTuple<bool, char, short, int, double>;

bool tryit4T() {
  T tup;
  get<0>(tup) = true;
  get<1>(tup) = 1;
  get<2>(tup) = 1;
  get<3>(tup) = 1;
  get<4>(tup) = 1.0;
  return get<0>(tup);
}
```

???
GodBolt: https://godbolt.org/z/75Qe2Z

---
name: custom tuple
template: basic-layout

(As seen at https://godbolt.org/z/o2uJWt)

<pre style='font-size:12px'>
tuple_element&lt;0ul, std::tuple&lt;bool, char, short, int, double> >::type& get&lt;0ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
tuple_element&lt;1ul, std::tuple&lt;bool, char, short, int, double> >::type& get&lt;1ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
tuple_element&lt;2ul, std::tuple&lt;bool, char, short, int, double> >::type& get&lt;2ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
tuple_element&lt;3ul, std::tuple&lt;bool, char, short, int, double> >::type& get&lt;3ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
tuple_element&lt;4ul, std::tuple&lt;bool, char, short, int, double> >::type& get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
bool& Getter&lt;0ul, bool>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
char& Getter&lt;1ul, char>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
char& Getter&lt;0ul, char>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
short& Getter&lt;2ul, short>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
short& Getter&lt;1ul, short>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
short& Getter&lt;0ul, short>::get&lt;short, int, double>(MyTuple&lt;short, int, double>&)
int& Getter&lt;3ul, int>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
int& Getter&lt;2ul, int>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
int& Getter&lt;1ul, int>::get&lt;short, int, double>(MyTuple&lt;short, int, double>&)
int& Getter&lt;0ul, int>::get&lt;int, double>(MyTuple&lt;int, double>&)
double& Getter&lt;4ul, double>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
double& Getter&lt;3ul, double>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
double& Getter&lt;2ul, double>::get&lt;short, int, double>(MyTuple&lt;short, int, double>&)
double& Getter&lt;1ul, double>::get&lt;int, double>(MyTuple&lt;int, double>&)
double& Getter&lt;0ul, double>::get&lt;double>(MyTuple<double>&)
</pre>

---
name: custom tuple
template: basic-layout

Just this one line:
```cpp
get<4>(tup) = 1.0;
```
Calls:
<pre style='font-size:20px'>
 tuple_element&lt;4ul, std::tuple&lt;bool, char, short, int, double> >::type&
     get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&);

 // Which calls these, in order:
 double& Getter&lt;4ul, double>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
 double& Getter&lt;3ul, double>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
 double& Getter&lt;2ul, double>::get&lt;short, int, double>(MyTuple&lt;short, int, double>&)
 double& Getter&lt;1ul, double>::get&lt;int, double>(MyTuple&lt;int, double>&)
 double& Getter&lt;0ul, double>::get&lt;double>(MyTuple<double>&)
</pre>

---
name: custom tuple
template: basic-layout

1. Change the functions to return auto& to determine the type.

```cpp
template<size_t N, typename... Ts> auto get(MyTuple<Ts...> &tup)
                -> typename std::tuple_element<N, std::tuple<Ts...>>::type& {
  return Getter<N, typename std::tuple_element<N, std::tuple<Ts...>>::type>::get(tup);
```

Becomes this:
```cpp
template<size_t N, typename... Ts> auto& get(MyTuple<Ts...> &tup) {
  return Getter<N>::get(tup);
```
<hr />
<pre style='font-size:20px'>
 tuple_element&lt;4ul, std::tuple&lt;bool, char, short, int, double> >::type&
     get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&);
</pre>
Becomes:
<pre style='font-size:20px'>
 auto& get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&);
</pre>

---
name: custom tuple
template: basic-layout

Still O(N^2) to get a single element however...
```cpp
get<4>(tup) = 1.0;
```
Calls:
<pre style='font-size:20px'>
 auto& get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&);

 // Which calls these, in order:
 auto& Getter&lt;4ul>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
 auto& Getter&lt;3ul>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
 auto& Getter&lt;2ul>::get&lt;short, int, double>(MyTuple&lt;short, int, double>&)
 auto& Getter&lt;1ul>::get&lt;int, double>(MyTuple&lt;int, double>&)
 auto& Getter&lt;0ul>::get&lt;double>(MyTuple<double>&)
</pre>

---
name: custom tuple
template: basic-layout

2 . Use function overrides to go directly to an inherited function of your choice

```cpp
template<typename... T> class MyTuple {
  // This definition is only used for MyTuple<>, the base instance.
  // Real uses are specialized below.
};

template<typename T, typename... More>
class MyTuple<T, More...> : MyTuple<More...> {
  T first;

 public:
  using MyTuple<More...>::get;
  auto& get(std::integral_constant<size_t, sizeof...(More)>) { return first; }
};

template<size_t N, typename... Ts>
auto& get(MyTuple<Ts...> &tup) {
  return tup.get(std::integral_constant<size_t, sizeof...(Ts) - 1 - N>{});
}
```

---
name: custom tuple
template: basic-layout

One tweak for the base class though:

```cpp
template<typename... T> class MyTuple {
*public:
* void get(...) = delete;
};

template<typename T, typename... More>
class MyTuple<T, More...> : MyTuple<More...> {
  T first;

 public:
  using MyTuple<More...>::get;
  auto& get(std::integral_constant<size_t, sizeof...(More)>) { return first; }
};

template<size_t N, typename... Ts>
auto& get(MyTuple<Ts...> &tup) {
  return tup.get(std::integral_constant<size_t, sizeof...(Ts) - 1 - N>{});
}
```

---

And now...
```cpp
get<4>(tup) = 1.0;
```
Calls:
<pre style='font-size:20px'>
 auto& get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&);

 // Which calls:
 MyTuple&lt;double>::get(std::integral_constant&lt;unsigned long, 0ul>)
</pre>


---
name: custom tuple
template: basic-layout

#### Still O(N^2) to instantiate MyTuple.

MyTuple&lt;A, B, C> inherits from
MyTuple&lt;B, C> which inherits from MyTuple&lt;C>.

In order to do MyTuple in O(N), we need a way to instantiate each individual type that doesn't involve the others.  So let's try inheriting from Leaf&lt;A> and Leaf&lt;B> and Leaf&lt;C> etc.

???
Next slide.

---
name: custom tuple
template: basic-layout

So let's try inheriting from Leaf&lt;A> and Leaf&lt;B> and Leaf&lt;C> etc.

```cpp
template <typename T> struct MyTupleLeaf {
  T value_;
  constexpr MyTupleLeaf(Arg&&... arg) : value_(std::forward<Arg>(arg)...) {}
};

template<typename... T> class MyTuple : MyTupleLeaf<T>... {
public:
  template<typename... Args>
  explicit constexpr MyTuple(Args&&... args)
    : MyTupleLeaf<T>(std::forward<Args>(args))... {}

 template<size_t I>
 auto& get() {  /* ?? */ }
};

MyTuple<char, short, int> tuple{'c', 1, 2};
```

???

Mention the constructor API and the fact that future slides own't have it.

But how does get() work?  More importantly, doesn't this have problems if a T is mentioned
twice?

---
name: custom tuple
template: basic-layout

So let's bake a selector value into the types we derive from...

```cpp
template <`std::size_t I,` typename T>
struct MyTupleLeaf {
  T value_;
};

template<`size_t... Index,` typename... T> class MyTuple : MyTupleLeaf<`Index,` T>... {
public:
 template<size_t I>
 auto& get() {  /* ?? */ }
};

MyTuple<`0, 1, 2,` char, short, int> tuple;
```

???

But you can only have one variadic in a template specialization, so we'll need to encapsulate them in a type.
---
name: custom tuple
template: basic-layout

Encapsulate the indices into a type.

```cpp
template <std::size_t I, typename T>
struct MyTupleLeaf {
  T value_;
};

*template<typename... T> class MyTuple;

template<size_t... Index, typename... T>
class MyTuple<`std::index_sequence<Index...>,` T...> : MyTupleLeaf<Index, T>... {
public:
 template<size_t I>
 auto& get() {  /* ?? */ }
};

MyTuple<`std::index_sequence<0, 1, 2>,` char, short, int> tuple;
```

???
And now we can implement get!
---
name: custom tuple
template: basic-layout

Now, implement get()

```cpp
template <std::size_t I, typename T>
struct MyTupleLeaf {
  T value_;
* T& get(std::integral_constant<size_t, I>) { return value_; }
};

template<typename... T> class MyTuple;

template<size_t... Index, typename... T>
class MyTuple<std::index_sequence<Index...>, T...> : MyTupleLeaf<Index, T>... {
* using MyTupleLeaf<Index, T>::get...;
public:
  template<size_t I>
* auto& get() { return get(std::integral_constant<size_t, I>{}); }
};

MyTuple<std::index_sequence<0, 1, 2>, char, short, int> tuple;
*short& get1 = tuple.get<1>();
```
???
But what if you don't have C++17?  The technique is applicable elsewhere, so...

---
name: hi there
template: basic-layout

Make MyTuple generate the index sequence!

```cpp
template <std::size_t I, typename T>
struct MyTupleLeaf {
  T value_;
  T& get(std::integral_constant<size_t, I>) { return value_; }
};

template<typename... T> class MyTuple;
template<size_t... Index, typename... T>
class MyTuple<std::index_sequence<Index...>, T...> : MyTupleLeaf<Index, T>... {
  using MyTupleLeaf<Index, T>::get...;
public:
  template<size_t I>
  auto& get() { return get(std::integral_constant<size_t, I>{}); }
};

*template<typename... T>
*class MyTuple : MyTuple<std::make_index_sequence<sizeof...(T)>, T...> {
*};
*MyTuple<char, short, int> tuple;
```
???
But what if you want a tuple with a first type that is an index sequence?

---
Make MyTuple a using statement, to separate it from its implementation.

```cpp
template <std::size_t I, typename T>
struct MyTupleLeaf {
  T value_;
  T& get(std::integral_constant<size_t, I>) { return value_; }
};

template<typename... T> class MyTuple`Impl`;
template<size_t... Index, typename... T>
class MyTuple`Impl`<std::index_sequence<Index...>, T...> : MyTupleLeaf<Index, T>... {
  using MyTupleLeaf<Index, T>::get...;
public:
  template<size_t I>
  auto& get() { return get(std::integral_constant<size_t, I>{}); }
};

template<typename... T>
 `using` MyTuple = MyTuple`Impl`<std::make_index_sequence<sizeof...(T)>, T...>;

MyTuple<char, short, int> tuple;
```

???
But what about C++11?
---
name: custom tuple in C++11
template: basic-layout

But before C++17, you can't use variadic using statements.  What about C++11?

```cpp
template<typename... T> class MyTupleImpl;
template<size_t... Index, typename... T>
class MyTupleImpl<std::index_sequence<Index...>, T...> : MyTupleLeaf<Index, T>... {
* using MyTupleLeaf<Index, T>::get...;
public:
  template<size_t I>
  auto& get() { return get(std::integral_constant<size_t, I>{}); }
};
```

---
name: custom tuple in C++11
template: basic-layout

In C++11, we could do it with an extra template parameter...

```cpp
template<typename... T> class MyTupleImpl;
template<size_t... Index, typename... T>
class MyTupleImpl<std::index_sequence<Index...>, T...> : MyTupleLeaf<Index, T>... {
  using MyTupleLeaf<Index, T>::get...;
public:
//template<size_t I>
//auto& get() { return get(std::integral_constant<size_t, I>{}); }
  template<size_t I`, typename TType`>
  auto& get() {
    `MyTupleLeaf<I, TType> &leaf = *this;`
    return `leaf.`get(std::integral_constant<size_t, I>{});
  }
};

MyTuple<std::index_sequence<0, 1, 2>, char, short, int> tuple;
//short& get1 = tuple.get<1>();
short& get1 = tuple.get<1, short>();

```

---
name: custom tuple in C++11
template: basic-layout

Better yet, if we pass the tuple as a parameter, we can deduce the type:

```cpp
template<typename... T> class MyTupleImpl;
template<size_t... Index, typename... T>
class MyTupleImpl<std::index_sequence<Index...>, T...> : MyTupleLeaf<Index, T>... {
  using MyTupleLeaf<Index, T>::get...;
public:
//template<size_t I>
//auto& get() { return get(std::integral_constant<size_t, I>{}); }
* template<size_t I, typename TType>
* auto& get_leaf(MyTupleLeaf<I, TType> *leaf) { return *leaf; }
  template<size_t I>
  auto& get() {
*   auto &leaf = get_leaf<I>(this);
    return leaf.get(std::integral_constant<size_t, I>{});
  }
};

MyTuple<std::index_sequence<0, 1, 2>, char, short, int> tuple;
short& get1 = tuple.get<1>();

```

---
name: custom tuple
template: basic-layout

LAST SLIDE
LAST SLIDE
LAST SLIDE
LAST SLIDE
LAST SLIDE


Reference: http://talesofcpp.fusionfenix.com/post-22/true-story-efficient-packing

Reference2: https://ldionne.com/2015/11/29/efficient-parameter-pack-indexing/

Older code: https://godbolt.org/z/5RZkb5
Newest code: https://godbolt.org/z/vgGT2I

---
name: conclusion
template: title-layout

## Hopefully this at least serves a basis for understanding C++ templates!

---
name: questions
template: title-layout

# Questions?

???

