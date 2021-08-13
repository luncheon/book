## Advanced Types

The Rust type system has some features that we’ve mentioned in this book but
haven’t yet discussed. We’ll start by discussing newtypes in general as we
examine why newtypes are useful as types. Then we’ll move on to type aliases, a
feature similar to newtypes but with slightly different semantics. We’ll also
discuss the `!` type and dynamically sized types.

> Rust の型システムには、すでに本書で触れたものの、まだ説明していない機能があります。まず、一般的な newtypes について説明し、 newtypes が型として有用である理由を見ていきます。それから、 newtypes と似ていますが、やや異なる意味を持つ type aliases に移ります。また、 `!` 型と動的にサイズ変更される型についても説明します。

### Using the Newtype Pattern for Type Safety and Abstraction

> Note: This section assumes you’ve read the earlier section [“Using the
> Newtype Pattern to Implement External Traits on External
> Types.”][using-the-newtype-pattern]<!-- ignore -->
>
> 注意: このセクションでは、前のセクション [「newtype パターンを使って外部の型に外部のトレイトを実装する」][using-the-newtype-pattern] を読んでいることを前提としています。

The newtype pattern is useful for tasks beyond those we’ve discussed so far,
including statically enforcing that values are never confused and indicating
the units of a value. You saw an example of using newtypes to indicate units in
Listing 19-15: recall that the `Millimeters` and `Meters` structs wrapped `u32`
values in a newtype. If we wrote a function with a parameter of type
`Millimeters`, we couldn’t compile a program that accidentally tried to call
that function with a value of type `Meters` or a plain `u32`.

> newtype パターンは、これまで説明してきたこと以外にも、値が決して混同されないことを静的に保証したり、値の単位を示すのにも役立ちます。リスト19-15では、単位を示すために newtypes を使用する例を見ました。 `Millimeters` と `Meters` 構造体が `u32 `の値を newtype でラップしたことを思い出してください。 `Millimeters` 型のパラメータを持つ関数を書いた場合、誤って `Meters` 型や単なる `u32` 型の値でその関数を呼ぼうとしたプログラムはコンパイルできません。

Another use of the newtype pattern is in abstracting away some implementation
details of a type: the new type can expose a public API that is different from
the API of the private inner type if we used the new type directly to restrict
the available functionality, for example.

> newtype パターンのもうひとつの使い方は、型の実装の詳細を抽象化することです。例えば、利用可能な機能を制限するために new type を使えば、プライベートな内部の型の API とは異なる新しい型の API を外部公開できます。

Newtypes can also hide internal implementation. For example, we could provide a
`People` type to wrap a `HashMap<i32, String>` that stores a person’s ID
associated with their name. Code using `People` would only interact with the
public API we provide, such as a method to add a name string to the `People`
collection; that code wouldn’t need to know that we assign an `i32` ID to names
internally. The newtype pattern is a lightweight way to achieve encapsulation
to hide implementation details, which we discussed in the [“Encapsulation that
Hides Implementation
Details”][encapsulation-that-hides-implementation-details]<!-- ignore -->
section of Chapter 17.

> newtypes は内部の実装を隠すこともできます。例えば、 ID と名前を関連付けて格納する `HashMap<i32, String>` をラップした `People` 型を提供できます。 `People` を使用するコードは、名前の文字列を `People` コレクションに追加するメソッドなどの公開 API でのみやりとりすることになります。そのコードは、名前に対して `i32` の ID を内部的に割り当てていることを知る必要はありません。 newtype パターンは、実装の詳細を隠すためのカプセル化を実現する軽量な方法で、第17章の [「実装の詳細を隠すカプセル化」][encapsulation-that-hides-implementation-details] のセクションで説明しました。

### Creating Type Synonyms with Type Aliases

Along with the newtype pattern, Rust provides the ability to declare a *type
alias* to give an existing type another name. For this we use the `type`
keyword. For example, we can create the alias `Kilometers` to `i32` like so:

> newtype パターンに加えて、 Rust では既存の型に別の名前を与える *type alias* を宣言する機能があります。これには `type` キーワードを使います。例えば、次のように `i32` へのエイリアス `Kilometers` を作ることができます。

```rust
{{#rustdoc_include ../listings/ch19-advanced-features/no-listing-04-kilometers-alias/src/main.rs:here}}
```

Now, the alias `Kilometers` is a *synonym* for `i32`; unlike the `Millimeters`
and `Meters` types we created in Listing 19-15, `Kilometers` is not a separate,
new type. Values that have the type `Kilometers` will be treated the same as
values of type `i32`:

> これで、エイリアス `Kilometers` は `i32` の *シノニム* になります。リスト19-15で作成した `Millimeters` や `Meters` の型とは異なり、 `Kilometers` は独立した新しい型ではありません。 `Kilometers` 型の値は、 `i32` 型の値と同じように扱われます。

```rust
{{#rustdoc_include ../listings/ch19-advanced-features/no-listing-04-kilometers-alias/src/main.rs:there}}
```

Because `Kilometers` and `i32` are the same type, we can add values of both
types and we can pass `Kilometers` values to functions that take `i32`
parameters. However, using this method, we don’t get the type checking benefits
that we get from the newtype pattern discussed earlier.

> `Kilometers` と `i32` は同じ型なので、双方の型の値を加算でき、 `i32` のパラメータを取る関数に `Kilometers` の値を渡すこともできます。ただし、この方法では、前述の newtype パターンで得られる型チェックのメリットは得られません。

The main use case for type synonyms is to reduce repetition. For example, we
might have a lengthy type like this:

> 型シノニムの主なユースケースとしては、繰り返しを減らすことが挙げられます。例えば、次のような長い型があるとします。

```rust,ignore
Box<dyn Fn() + Send + 'static>
```

Writing this lengthy type in function signatures and as type annotations all
over the code can be tiresome and error prone. Imagine having a project full of
code like that in Listing 19-24.

> このような長い型を関数のシグネチャやコード全体の型アノテーションで書くのは、面倒な上にエラーが起こりやすいものです。リスト19-24のようなコードでいっぱいのプロジェクトを想像してみてください。

```rust
{{#rustdoc_include ../listings/ch19-advanced-features/listing-19-24/src/main.rs:here}}
```

<span class="caption">Listing 19-24: Using a long type in many places</span>

A type alias makes this code more manageable by reducing the repetition. In
Listing 19-25, we’ve introduced an alias named `Thunk` for the verbose type and
can replace all uses of the type with the shorter alias `Thunk`.

> 型エイリアスは繰り返しを減らし、コードをより管理しやすくします。リスト19-25では、冗長な型に対して `Thunk` という名前のエイリアスを導入し、この型のすべての使用箇所をより短いエイリアス `Thunk` で置き換えています。

```rust
{{#rustdoc_include ../listings/ch19-advanced-features/listing-19-25/src/main.rs:here}}
```

<span class="caption">Listing 19-25: Introducing a type alias `Thunk` to reduce
repetition</span>

This code is much easier to read and write! Choosing a meaningful name for a
type alias can help communicate your intent as well (*thunk* is a word for code
to be evaluated at a later time, so it’s an appropriate name for a closure that
gets stored).

> このコードは読みやすく、書きやすくなりました。型エイリアスに意味のある名前を選ぶことは、意図を伝えるのにも役立ちます（*thunk* は、あとで評価されるコードを表す言葉なので、保存されるクロージャの名前として適切です）。

Type aliases are also commonly used with the `Result<T, E>` type for reducing
repetition. Consider the `std::io` module in the standard library. I/O
operations often return a `Result<T, E>` to handle situations when operations
fail to work. This library has a `std::io::Error` struct that represents all
possible I/O errors. Many of the functions in `std::io` will be returning
`Result<T, E>` where the `E` is `std::io::Error`, such as these functions in
the `Write` trait:

> 型エイリアスは、繰り返しを減らすために、 `Result<T, E>` 型でもよく使われます。標準ライブラリの `std::io` モジュールを考えてみましょう。 I/O 操作では、失敗時のハンドリングのために、しばしば `Result<T, E>` を返します。このライブラリには、起こりうる全ての I/O エラーを表現するための `std::io::Error` 構造体があります。 `std::io` の関数の多くが返す `Result<T, E>` の `E` は `std::io::Error` です。例えば `Write` トレイトのこれらの関数がそうです。

```rust,noplayground
{{#rustdoc_include ../listings/ch19-advanced-features/no-listing-05-write-trait/src/lib.rs}}
```

The `Result<..., Error>` is repeated a lot. As such, `std::io` has this type
alias declaration:

> `Result<..., Error>` は何度も繰り返されます。このため、 `std::io` にはこのような型エイリアス宣言があります。

```rust,noplayground
{{#rustdoc_include ../listings/ch19-advanced-features/no-listing-06-result-alias/src/lib.rs:here}}
```

Because this declaration is in the `std::io` module, we can use the fully
qualified alias `std::io::Result<T>`—that is, a `Result<T, E>` with the `E`
filled in as `std::io::Error`. The `Write` trait function signatures end up
looking like this:

> この宣言は `std::io` モジュールの中にあるので、私たちは完全修飾のエイリアス `std::io::Result<T>` を使うことができます（`Result<T, E>` の `E` に `std::io::Error` を当てはめた型として）。 `Write` トレイトの関数シグネチャは次のようになります。

```rust,noplayground
{{#rustdoc_include ../listings/ch19-advanced-features/no-listing-06-result-alias/src/lib.rs:there}}
```

The type alias helps in two ways: it makes code easier to write *and* it gives
us a consistent interface across all of `std::io`. Because it’s an alias, it’s
just another `Result<T, E>`, which means we can use any methods that work on
`Result<T, E>` with it, as well as special syntax like the `?` operator.

> 型エイリアスには 2 つの利点があります: コードを書きやすくすること *and* `std::io` 全体で一貫したインターフェイスを提供することです。これはエイリアスであり、単なるもうひとつの `Result<T, E>` なので、 `Result<T, E>` で動作するすべてのメソッドや `?` 演算子のような特殊な構文を使うことができます。

### The Never Type that Never Returns

Rust has a special type named `!` that’s known in type theory lingo as the
*empty type* because it has no values. We prefer to call it the *never type*
because it stands in the place of the return type when a function will never
return. Here is an example:

```rust,noplayground
{{#rustdoc_include ../listings/ch19-advanced-features/no-listing-07-never-type/src/lib.rs:here}}
```

This code is read as “the function `bar` returns never.” Functions that return
never are called *diverging functions*. We can’t create values of the type `!`
so `bar` can never possibly return.

But what use is a type you can never create values for? Recall the code from
Listing 2-5; we’ve reproduced part of it here in Listing 19-26.

```rust,ignore
{{#rustdoc_include ../listings/ch02-guessing-game-tutorial/listing-02-05/src/main.rs:ch19}}
```

<span class="caption">Listing 19-26: A `match` with an arm that ends in
`continue`</span>

At the time, we skipped over some details in this code. In Chapter 6 in [“The
`match` Control Flow Operator”][the-match-control-flow-operator]<!-- ignore
--> section, we discussed that `match` arms must all return the same type. So,
for example, the following code doesn’t work:

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch19-advanced-features/no-listing-08-match-arms-different-types/src/main.rs:here}}
```

The type of `guess` in this code would have to be an integer *and* a string,
and Rust requires that `guess` have only one type. So what does `continue`
return? How were we allowed to return a `u32` from one arm and have another arm
that ends with `continue` in Listing 19-26?

As you might have guessed, `continue` has a `!` value. That is, when Rust
computes the type of `guess`, it looks at both match arms, the former with a
value of `u32` and the latter with a `!` value. Because `!` can never have a
value, Rust decides that the type of `guess` is `u32`.

The formal way of describing this behavior is that expressions of type `!` can
be coerced into any other type. We’re allowed to end this `match` arm with
`continue` because `continue` doesn’t return a value; instead, it moves control
back to the top of the loop, so in the `Err` case, we never assign a value to
`guess`.

The never type is useful with the `panic!` macro as well. Remember the `unwrap`
function that we call on `Option<T>` values to produce a value or panic? Here
is its definition:

```rust,ignore
{{#rustdoc_include ../listings/ch19-advanced-features/no-listing-09-unwrap-definition/src/lib.rs:here}}
```

In this code, the same thing happens as in the `match` in Listing 19-26: Rust
sees that `val` has the type `T` and `panic!` has the type `!`, so the result
of the overall `match` expression is `T`. This code works because `panic!`
doesn’t produce a value; it ends the program. In the `None` case, we won’t be
returning a value from `unwrap`, so this code is valid.

One final expression that has the type `!` is a `loop`:

```rust,ignore
{{#rustdoc_include ../listings/ch19-advanced-features/no-listing-10-loop-returns-never/src/main.rs:here}}
```

Here, the loop never ends, so `!` is the value of the expression. However, this
wouldn’t be true if we included a `break`, because the loop would terminate
when it got to the `break`.

### Dynamically Sized Types and the `Sized` Trait

Due to Rust’s need to know certain details, such as how much space to allocate
for a value of a particular type, there is a corner of its type system that can
be confusing: the concept of *dynamically sized types*. Sometimes referred to
as *DSTs* or *unsized types*, these types let us write code using values whose
size we can know only at runtime.

Let’s dig into the details of a dynamically sized type called `str`, which
we’ve been using throughout the book. That’s right, not `&str`, but `str` on
its own, is a DST. We can’t know how long the string is until runtime, meaning
we can’t create a variable of type `str`, nor can we take an argument of type
`str`. Consider the following code, which does not work:

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch19-advanced-features/no-listing-11-cant-create-str/src/main.rs:here}}
```

Rust needs to know how much memory to allocate for any value of a particular
type, and all values of a type must use the same amount of memory. If Rust
allowed us to write this code, these two `str` values would need to take up the
same amount of space. But they have different lengths: `s1` needs 12 bytes of
storage and `s2` needs 15. This is why it’s not possible to create a variable
holding a dynamically sized type.

So what do we do? In this case, you already know the answer: we make the types
of `s1` and `s2` a `&str` rather than a `str`. Recall that in the [“String
Slices”][string-slices]<!-- ignore --> section of Chapter 4, we said the slice
data structure stores the starting position and the length of the slice.

So although a `&T` is a single value that stores the memory address of where
the `T` is located, a `&str` is *two* values: the address of the `str` and its
length. As such, we can know the size of a `&str` value at compile time: it’s
twice the length of a `usize`. That is, we always know the size of a `&str`, no
matter how long the string it refers to is. In general, this is the way in
which dynamically sized types are used in Rust: they have an extra bit of
metadata that stores the size of the dynamic information. The golden rule of
dynamically sized types is that we must always put values of dynamically sized
types behind a pointer of some kind.

We can combine `str` with all kinds of pointers: for example, `Box<str>` or
`Rc<str>`. In fact, you’ve seen this before but with a different dynamically
sized type: traits. Every trait is a dynamically sized type we can refer to by
using the name of the trait. In Chapter 17 in the [“Using Trait Objects That
Allow for Values of Different
Types”][using-trait-objects-that-allow-for-values-of-different-types]<!--
ignore --> section, we mentioned that to use traits as trait objects, we must
put them behind a pointer, such as `&dyn Trait` or `Box<dyn Trait>` (`Rc<dyn
Trait>` would work too).

To work with DSTs, Rust has a particular trait called the `Sized` trait to
determine whether or not a type’s size is known at compile time. This trait is
automatically implemented for everything whose size is known at compile time.
In addition, Rust implicitly adds a bound on `Sized` to every generic function.
That is, a generic function definition like this:

```rust,ignore
{{#rustdoc_include ../listings/ch19-advanced-features/no-listing-12-generic-fn-definition/src/lib.rs}}
```

is actually treated as though we had written this:

```rust,ignore
{{#rustdoc_include ../listings/ch19-advanced-features/no-listing-13-generic-implicit-sized-bound/src/lib.rs}}
```

By default, generic functions will work only on types that have a known size at
compile time. However, you can use the following special syntax to relax this
restriction:

```rust,ignore
{{#rustdoc_include ../listings/ch19-advanced-features/no-listing-14-generic-maybe-sized/src/lib.rs}}
```

A trait bound on `?Sized` means “`T` may or may not be `Sized`” and this
notation overrides the default that generic types must have a known size at
compile time. The `?Trait` syntax with this meaning is only available for
`Sized`, not any other traits.

Also note that we switched the type of the `t` parameter from `T` to `&T`.
Because the type might not be `Sized`, we need to use it behind some kind of
pointer. In this case, we’ve chosen a reference.

Next, we’ll talk about functions and closures!

[encapsulation-that-hides-implementation-details]:
ch17-01-what-is-oo.html#encapsulation-that-hides-implementation-details
[string-slices]: ch04-03-slices.html#string-slices
[the-match-control-flow-operator]:
ch06-02-match.html#the-match-control-flow-operator
[using-trait-objects-that-allow-for-values-of-different-types]:
ch17-02-trait-objects.html#using-trait-objects-that-allow-for-values-of-different-types
[using-the-newtype-pattern]: ch19-03-advanced-traits.html#using-the-newtype-pattern-to-implement-external-traits-on-external-types
