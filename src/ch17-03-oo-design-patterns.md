## Implementing an Object-Oriented Design Pattern

The *state pattern* is an object-oriented design pattern. The crux of the
pattern is that a value has some internal state, which is represented by a set
of *state objects*, and the value’s behavior changes based on the internal
state. The state objects share functionality: in Rust, of course, we use
structs and traits rather than objects and inheritance. Each state object is
responsible for its own behavior and for governing when it should change into
another state. The value that holds a state object knows nothing about the
different behavior of the states or when to transition between states.

Using the state pattern means when the business requirements of the program
change, we won’t need to change the code of the value holding the state or the
code that uses the value. We’ll only need to update the code inside one of the
state objects to change its rules or perhaps add more state objects. Let’s look
at an example of the state design pattern and how to use it in Rust.

We’ll implement a blog post workflow in an incremental way. The blog’s final
functionality will look like this:

1. A blog post starts as an empty draft.
2. When the draft is done, a review of the post is requested.
3. When the post is approved, it gets published.
4. Only published blog posts return content to print, so unapproved posts can’t
   accidentally be published.

Any other changes attempted on a post should have no effect. For example, if we
try to approve a draft blog post before we’ve requested a review, the post
should remain an unpublished draft.

Listing 17-11 shows this workflow in code form: this is an example usage of the
API we’ll implement in a library crate named `blog`. This won’t compile yet
because we haven’t implemented the `blog` crate yet.

<span class="filename">Filename: src/main.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch17-oop/listing-17-11/src/main.rs:all}}
```

<span class="caption">Listing 17-11: Code that demonstrates the desired
behavior we want our `blog` crate to have</span>

We want to allow the user to create a new draft blog post with `Post::new`.
Then we want to allow text to be added to the blog post while it’s in the draft
state. If we try to get the post’s content immediately, before approval,
nothing should happen because the post is still a draft. We’ve added
`assert_eq!` in the code for demonstration purposes. An excellent unit test for
this would be to assert that a draft blog post returns an empty string from the
`content` method, but we’re not going to write tests for this example.

Next, we want to enable a request for a review of the post, and we want
`content` to return an empty string while waiting for the review. When the post
receives approval, it should get published, meaning the text of the post will
be returned when `content` is called.

Notice that the only type we’re interacting with from the crate is the `Post`
type. This type will use the state pattern and will hold a value that will be
one of three state objects representing the various states a post can be
in—draft, waiting for review, or published. Changing from one state to another
will be managed internally within the `Post` type. The states change in
response to the methods called by our library’s users on the `Post` instance,
but they don’t have to manage the state changes directly. Also, users can’t
make a mistake with the states, like publishing a post before it’s reviewed.

### Defining `Post` and Creating a New Instance in the Draft State

Let’s get started on the implementation of the library! We know we need a
public `Post` struct that holds some content, so we’ll start with the
definition of the struct and an associated public `new` function to create an
instance of `Post`, as shown in Listing 17-12. We’ll also make a private
`State` trait. Then `Post` will hold a trait object of `Box<dyn State>`
inside an `Option<T>` in a private field named `state`. You’ll see why the
`Option<T>` is necessary in a bit.

<span class="filename">Filename: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch17-oop/listing-17-12/src/lib.rs}}
```

<span class="caption">Listing 17-12: Definition of a `Post` struct and a `new`
function that creates a new `Post` instance, a `State` trait, and a `Draft`
struct</span>

The `State` trait defines the behavior shared by different post states, and the
`Draft`, `PendingReview`, and `Published` states will all implement the `State`
trait. For now, the trait doesn’t have any methods, and we’ll start by defining
just the `Draft` state because that is the state we want a post to start in.

When we create a new `Post`, we set its `state` field to a `Some` value that
holds a `Box`. This `Box` points to a new instance of the `Draft` struct. This
ensures whenever we create a new instance of `Post`, it will start out as a
draft. Because the `state` field of `Post` is private, there is no way to
create a `Post` in any other state! In the `Post::new` function, we set the
`content` field to a new, empty `String`.

### Storing the Text of the Post Content

Listing 17-11 showed that we want to be able to call a method named
`add_text` and pass it a `&str` that is then added to the text content of the
blog post. We implement this as a method rather than exposing the `content`
field as `pub`. This means we can implement a method later that will control
how the `content` field’s data is read. The `add_text` method is pretty
straightforward, so let’s add the implementation in Listing 17-13 to the `impl
Post` block:

<span class="filename">Filename: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch17-oop/listing-17-13/src/lib.rs:here}}
```

<span class="caption">Listing 17-13: Implementing the `add_text` method to add
text to a post’s `content`</span>

The `add_text` method takes a mutable reference to `self`, because we’re
changing the `Post` instance that we’re calling `add_text` on. We then call
`push_str` on the `String` in `content` and pass the `text` argument to add to
the saved `content`. This behavior doesn’t depend on the state the post is in,
so it’s not part of the state pattern. The `add_text` method doesn’t interact
with the `state` field at all, but it is part of the behavior we want to
support.

### Ensuring the Content of a Draft Post Is Empty

Even after we’ve called `add_text` and added some content to our post, we still
want the `content` method to return an empty string slice because the post is
still in the draft state, as shown on line 7 of Listing 17-11. For now, let’s
implement the `content` method with the simplest thing that will fulfill this
requirement: always returning an empty string slice. We’ll change this later
once we implement the ability to change a post’s state so it can be published.
So far, posts can only be in the draft state, so the post content should always
be empty. Listing 17-14 shows this placeholder implementation:

<span class="filename">Filename: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch17-oop/listing-17-14/src/lib.rs:here}}
```

<span class="caption">Listing 17-14: Adding a placeholder implementation for
the `content` method on `Post` that always returns an empty string slice</span>

With this added `content` method, everything in Listing 17-11 up to line 7
works as intended.

### Requesting a Review of the Post Changes Its State

Next, we need to add functionality to request a review of a post, which should
change its state from `Draft` to `PendingReview`. Listing 17-15 shows this code:

<span class="filename">Filename: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch17-oop/listing-17-15/src/lib.rs:here}}
```

<span class="caption">Listing 17-15: Implementing `request_review` methods on
`Post` and the `State` trait</span>

We give `Post` a public method named `request_review` that will take a mutable
reference to `self`. Then we call an internal `request_review` method on the
current state of `Post`, and this second `request_review` method consumes the
current state and returns a new state.

We’ve added the `request_review` method to the `State` trait; all types that
implement the trait will now need to implement the `request_review` method.
Note that rather than having `self`, `&self`, or `&mut self` as the first
parameter of the method, we have `self: Box<Self>`. This syntax means the
method is only valid when called on a `Box` holding the type. This syntax takes
ownership of `Box<Self>`, invalidating the old state so the state value of the
`Post` can transform into a new state.

To consume the old state, the `request_review` method needs to take ownership
of the state value. This is where the `Option` in the `state` field of `Post`
comes in: we call the `take` method to take the `Some` value out of the `state`
field and leave a `None` in its place, because Rust doesn’t let us have
unpopulated fields in structs. This lets us move the `state` value out of
`Post` rather than borrowing it. Then we’ll set the post’s `state` value to the
result of this operation.

We need to set `state` to `None` temporarily rather than setting it directly
with code like `self.state = self.state.request_review();` to get ownership of
the `state` value. This ensures `Post` can’t use the old `state` value after
we’ve transformed it into a new state.

The `request_review` method on `Draft` needs to return a new, boxed instance of
a new `PendingReview` struct, which represents the state when a post is waiting
for a review. The `PendingReview` struct also implements the `request_review`
method but doesn’t do any transformations. Rather, it returns itself, because
when we request a review on a post already in the `PendingReview` state, it
should stay in the `PendingReview` state.

Now we can start seeing the advantages of the state pattern: the
`request_review` method on `Post` is the same no matter its `state` value. Each
state is responsible for its own rules.

We’ll leave the `content` method on `Post` as is, returning an empty string
slice. We can now have a `Post` in the `PendingReview` state as well as in the
`Draft` state, but we want the same behavior in the `PendingReview` state.
Listing 17-11 now works up to line 10!

### Adding the `approve` Method that Changes the Behavior of `content`

The `approve` method will be similar to the `request_review` method: it will
set `state` to the value that the current state says it should have when that
state is approved, as shown in Listing 17-16:

> `approve` メソッドは `request_review` メソッドと似ています: 承認されたときに持つべき値を `state` に設定します。 → リスト17-16

<span class="filename">Filename: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch17-oop/listing-17-16/src/lib.rs:here}}
```

<span class="caption">Listing 17-16: Implementing the `approve` method on
`Post` and the `State` trait</span>

We add the `approve` method to the `State` trait and add a new struct that
implements `State`, the `Published` state.

> `State` トレイトに `approve` メソッドを追加し、 `State` を実装した新しい構造体 `Published` を追加しています。

Similar to `request_review`, if we call the `approve` method on a `Draft`, it
will have no effect because it will return `self`. When we call `approve` on
`PendingReview`, it returns a new, boxed instance of the `Published` struct.
The `Published` struct implements the `State` trait, and for both the
`request_review` method and the `approve` method, it returns itself, because
the post should stay in the `Published` state in those cases.

> `request_review` と同様に、 `Draft` に対して `approve` メソッドを呼び出しても `self` を返すだけで何の効果もありません。 `PendingReview` に対して `approve` を呼び出すと、 `Published` 構造体のボックス化された新しいインスタンスが返されます。 `Published` 構造体は `State` トレイトを実装しており、 `request_review` メソッドと `approve` メソッドの両方で `self` を返します。

Now we need to update the `content` method on `Post`: if the state is
`Published`, we want to return the value in the post’s `content` field;
otherwise, we want to return an empty string slice, as shown in Listing 17-17:

> 今度は `Post` の `content` メソッドを更新する必要があります: 状態が `Published` であれば post の `content` フィールドの値を、そうでなければ空の文字列スライスを返したいです。 → リスト17-17

<span class="filename">Filename: src/lib.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch17-oop/listing-17-17/src/lib.rs:here}}
```

<span class="caption">Listing 17-17: Updating the `content` method on `Post` to
delegate to a `content` method on `State`</span>

Because the goal is to keep all these rules inside the structs that implement
`State`, we call a `content` method on the value in `state` and pass the post
instance (that is, `self`) as an argument. Then we return the value that is
returned from using the `content` method on the `state` value.

> 目標は、これらのルールをすべて `State` を実装する構造体群の中にしまい込むことです。なので、 `state` の `content` メソッドを呼び出し、引数として post インスタンス（すなわち `self`）を渡します。そして、 `state` の `content` メソッドで返ってきた値を返します。

We call the `as_ref` method on the `Option` because we want a reference to the
value inside the `Option` rather than ownership of the value. Because `state`
is an `Option<Box<dyn State>>`, when we call `as_ref`, an `Option<&Box<dyn State>>` is
returned. If we didn’t call `as_ref`, we would get an error because we can’t
move `state` out of the borrowed `&self` of the function parameter.

> `Option` の中の値の所有権ではなく参照がほしいので、 `Option` の `as_ref` メソッドを呼び出します（訳注: [`as_ref: &Option<T> -> Option<&T>`](https://doc.rust-lang.org/std/option/enum.Option.html#method.as_ref)）。 `state` は `Option<Box<dyn State>>` なので、 `as_ref` を呼び出すと `Option<&Box<dyn State>>` が返ってきます。借用した引数 `&self` では `state` をムーブできないので、 `as_ref` を呼ばなければエラーになるでしょう。

We then call the `unwrap` method, which we know will never panic, because we
know the methods on `Post` ensure that `state` will always contain a `Some`
value when those methods are done. This is one of the cases we talked about in
the [“Cases In Which You Have More Information Than the
Compiler”][more-info-than-rustc]<!-- ignore --> section of Chapter 9 when we
know that a `None` value is never possible, even though the compiler isn’t able
to understand that.

> それから `unwrap` メソッドを呼び出しますが、これは決してパニックにならないことがわかっています。なぜなら、 `Post` のメソッドが呼ばれるときには、 `state` が常に `Some` 値を持つことが保証されることを私たちは知っているからです。これは、第9章の「[コンパイラよりもあなたの方が多くの情報を持っている場合][more-info-than-rustc]<!-- ignore -->」で説明した、 `None` になり得ないことが私たちには分かってもコンパイラには分からないケースの一例です。（訳注: だとすれば `Post` の `request_review` と `approve` の実装も `self.state.take().unwrap()` で良いのでは？）

At this point, when we call `content` on the `&Box<dyn State>`, deref coercion will
take effect on the `&` and the `Box` so the `content` method will ultimately be
called on the type that implements the `State` trait. That means we need to add
`content` to the `State` trait definition, and that is where we’ll put the
logic for what content to return depending on which state we have, as shown in
Listing 17-18:

> この時点で、 `&Box<dyn State>` の `content` を呼び出すと、参照外し型強制 (deref coercion) が `&` と `Box` に働くので、最終的には `State` トレイトを実装した型に対して `content` メソッドが呼び出されることになります。つまり、 `State` トレイトの定義に `content` を追加する必要があり、状態に応じてどのコンテンツを返すかというロジックをここに置くことになります。 → リスト17-18

<span class="filename">Filename: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch17-oop/listing-17-18/src/lib.rs:here}}
```

<span class="caption">Listing 17-18: Adding the `content` method to the `State`
trait</span>

We add a default implementation for the `content` method that returns an empty
string slice. That means we don’t need to implement `content` on the `Draft`
and `PendingReview` structs. The `Published` struct will override the `content`
method and return the value in `post.content`.

> 空の文字列スライスを返す `content` メソッドのデフォルト実装を追加しています。 `Draft` 構造体と `PendingReview` 構造体に `content` を実装する必要はありません。 `Published` 構造体は `content` メソッドをオーバーライドし、 `post.content` の値を返します。

Note that we need lifetime annotations on this method, as we discussed in
Chapter 10. We’re taking a reference to a `post` as an argument and returning a
reference to part of that `post`, so the lifetime of the returned reference is
related to the lifetime of the `post` argument.

> 第10章で説明したように、このメソッドにはライフタイムアノテーションが必要であることに注意してください。引数として `post` への参照を取り、その `post` のフィールドの参照を返しているので、返される参照の寿命は引数 `post` の寿命に関連付けられます。

And we’re done—all of Listing 17-11 now works! We’ve implemented the state
pattern with the rules of the blog post workflow. The logic related to the
rules lives in the state objects rather than being scattered throughout `Post`.

> これでリスト17-11のすべてが動作するようになりました! ブログ記事のワークフローのルールで State パターンを実装しました。ルールに関連するロジックは `Post` 全体に散らばっているのではなく、状態オブジェクトの中にあります。

### Trade-offs of the State Pattern

We’ve shown that Rust is capable of implementing the object-oriented state
pattern to encapsulate the different kinds of behavior a post should have in
each state. The methods on `Post` know nothing about the various behaviors. The
way we organized the code, we have to look in only one place to know the
different ways a published post can behave: the implementation of the `State`
trait on the `Published` struct.

> Rust でオブジェクト指向の State パターン（状態ごとの post が持つべき各種動作のカプセル化）が実装できることを示しました。 `Post` のメソッドは各種動作について何も知りません。このようにコードを整理すると、公開済み記事のさまざまな動作を知るためには、たった 1 つの場所、すなわち `Published` 構造体の `State` トレイトの実装を見ればよいことになります。

If we were to create an alternative implementation that didn’t use the state
pattern, we might instead use `match` expressions in the methods on `Post` or
even in the `main` code that checks the state of the post and changes behavior
in those places. That would mean we would have to look in several places to
understand all the implications of a post being in the published state! This
would only increase the more states we added: each of those `match` expressions
would need another arm.

> State パターンを使わない別の実装を作るなら、 post の状態をチェックして動作を変える場所（`Post` の各メソッド or even `main` コード）で代わりに `match` 式を使うかもしれません。これでは、あちこち調べないと、記事が公開状態であることの意味を完全に理解できません！ これは状態を追加するほど増えていきます: 各 `match` 式に別のアームが必要になります。

With the state pattern, the `Post` methods and the places we use `Post` don’t
need `match` expressions, and to add a new state, we would only need to add a
new struct and implement the trait methods on that one struct.

> State パターンでは、 `Post` のメソッドや `Post` を使用する場所に `match` 式は必要ありません。新しい状態を追加するには、新しい構造体を追加して、その構造体にトレイトのメソッドを実装するだけでいいのです。

The implementation using the state pattern is easy to extend to add more
functionality. To see the simplicity of maintaining code that uses the state
pattern, try a few of these suggestions:

* Add a `reject` method that changes the post’s state from `PendingReview` back
  to `Draft`.
* Require two calls to `approve` before the state can be changed to `Published`.
* Allow users to add text content only when a post is in the `Draft` state.
  Hint: have the state object responsible for what might change about the
  content but not responsible for modifying the `Post`.

> State パターンを用いた実装は、機能を追加するための拡張が容易です。 State パターンを用いたコードのメンテナンスのシンプルさを確認するために、これらの提案をいくつか試してみてください:
>
> * post の状態を `PendingReview` から `Draft` に戻す `reject` メソッドを追加する。
> * 状態を `Published` に変更する前に `approve` の呼び出しを 2 回必要にする。
> * post が `Draft` 状態の時のみ、ユーザーがテキストコンテンツを追加できるようにする。  
    ヒント: 状態オブジェクトは、コンテンツの変更に責任を持ちますが、 `Post` の変更には責任を持ちません。

One downside of the state pattern is that, because the states implement the
transitions between states, some of the states are coupled to each other. If we
add another state between `PendingReview` and `Published`, such as `Scheduled`,
we would have to change the code in `PendingReview` to transition to
`Scheduled` instead. It would be less work if `PendingReview` didn’t need to
change with the addition of a new state, but that would mean switching to
another design pattern.

> State パターンの欠点の 1 つは、状態が状態間の遷移を実装するため、いくつかの状態が互いに結合していることです。もし、 `PendingReview` と `Published` の間に `Scheduled` のような別の状態を追加したら、 `PendingReview` のコードを変更して、代わりに `Scheduled` に遷移させる必要があります。新しい状態を追加しても `PendingReview` を変更せずに済むなら作業は少ないのですが、それだと別のデザインパターンになってしまいます。

Another downside is that we’ve duplicated some logic. To eliminate some of the
duplication, we might try to make default implementations for the
`request_review` and `approve` methods on the `State` trait that return `self`;
however, this would violate object safety, because the trait doesn’t know what
the concrete `self` will be exactly. We want to be able to use `State` as a
trait object, so we need its methods to be object safe.

> もう一つの欠点は、いくつかのロジックを重複させてしまったことです。重複をなくすために、 `State` トレイトの `request_review` と `approve` メソッドに `self` を返すデフォルト実装を作ろうとするかもしれません; しかし，これではオブジェクト安全性に反することになります。なぜなら、実体の `self` が何になるのか、トレイトは正確にはわからないからです。 `State` をトレイトオブジェクト（訳注: `dyn State`）として使えるようにしたいので、そのメソッドはオブジェクトセーフにする必要があります（訳注: オブジェクト安全性の詳細は [The Rust Reference 6.11. Traits # Object Safety](https://doc.rust-lang.org/reference/items/traits.html#object-safety) 参照）。

Other duplication includes the similar implementations of the `request_review`
and `approve` methods on `Post`. Both methods delegate to the implementation of
the same method on the value in the `state` field of `Option` and set the new
value of the `state` field to the result. If we had a lot of methods on `Post`
that followed this pattern, we might consider defining a macro to eliminate the
repetition (see the [“Macros”][macros]<!-- ignore --> section in Chapter 19).

> 他の重複としては、 `Post` の `request_review` と `approve` のメソッドの実装が似ていることが挙げられます。どちらのメソッドも `Option` の `state` フィールドの値を同じメソッドの実装に委譲して、 `state` フィールドの新しい値を結果に設定します。このパターンに沿った `Post` 上のメソッドがたくさんあるなら，マクロを定義して繰り返しをなくすことも考えられます（Chapter 19 「[Macros](ch19-06-macros.html#macros)」セクション参照）。

By implementing the state pattern exactly as it’s defined for object-oriented
languages, we’re not taking as full advantage of Rust’s strengths as we could.
Let’s look at some changes we can make to the `blog` crate that can make
invalid states and transitions into compile time errors.

> オブジェクト指向言語のために定義された State パターンを愚直に実装することで、 Rust の強みを最大限に活用できていません。ここでは、無効な状態や遷移をコンパイル時のエラーにするために、 `blog` クレートに加えられる変更を見てみましょう。

#### Encoding States and Behavior as Types

We’ll show you how to rethink the state pattern to get a different set of
trade-offs. Rather than encapsulating the states and transitions completely so
outside code has no knowledge of them, we’ll encode the states into different
types. Consequently, Rust’s type checking system will prevent attempts to use
draft posts where only published posts are allowed by issuing a compiler error.

> ここでは、 State パターンを再考し、異なるトレードオフを実現する方法を紹介します。状態と遷移を完全にカプセル化して外部のコードがそれを知らないようにするのではなく、状態を異なる型にエンコードします。結果、公開済み記事のみ許可されているところで下書き記事を使おうとすると、 Rust の型チェックシステムがコンパイルエラーを起こして阻止します。

Let’s consider the first part of `main` in Listing 17-11:

<span class="filename">Filename: src/main.rs</span>

```rust,ignore
{{#rustdoc_include ../listings/ch17-oop/listing-17-11/src/main.rs:here}}
```

We still enable the creation of new posts in the draft state using `Post::new`
and the ability to add text to the post’s content. But instead of having a
`content` method on a draft post that returns an empty string, we’ll make it so
draft posts don’t have the `content` method at all. That way, if we try to get
a draft post’s content, we’ll get a compiler error telling us the method
doesn’t exist. As a result, it will be impossible for us to accidentally
display draft post content in production, because that code won’t even compile.
Listing 17-19 shows the definition of a `Post` struct and a `DraftPost` struct,
as well as methods on each:

> `Post::new` を使った下書き状態での新規記事の作成や、記事のコンテンツにテキストを追加する機能は引き続き有効です。しかし、空文字列を返す `content` メソッドを下書き記事に持たせる代わりに、下書き記事には `content` メソッドを持たせないようにします。そうすれば、下書き記事のコンテンツを取得しようとすると、メソッドが存在しないというコンパイラエラーが発生します。結果として、そのコードはコンパイルされないので、運用時に誤って下書き記事のコンテンツを表示することは不可能です。リスト17-19は、 `Post` 構造体と `DraftPost` 構造体の定義、およびそれぞれのメソッドを示しています:

<span class="filename">Filename: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch17-oop/listing-17-19/src/lib.rs}}
```

<span class="caption">Listing 17-19: A `Post` with a `content` method and a
`DraftPost` without a `content` method</span>

Both the `Post` and `DraftPost` structs have a private `content` field that
stores the blog post text. The structs no longer have the `state` field because
we’re moving the encoding of the state to the types of the structs. The `Post`
struct will represent a published post, and it has a `content` method that
returns the `content`.

> `Post` 構造体と `DraftPost` 構造体には、ブログ記事のテキストを格納するプライベートの `content` フィールドがあります。これらの構造体には、もはや `state` フィールドはありません。これは、状態のエンコーディングを構造体の型に移行するためです。 `Post` 構造体は公開済みの記事を表し、 `content` を返す `content` メソッドを持ちます。

We still have a `Post::new` function, but instead of returning an instance of
`Post`, it returns an instance of `DraftPost`. Because `content` is private
and there aren’t any functions that return `Post`, it’s not possible to create
an instance of `Post` right now.

> `Post::new` 関数はありますが、 `Post` のインスタンスを返すのではなく、 `DraftPost` のインスタンスを返します。 `content` は private であり、 `Post` を返す関数もないので、今のところ `Post` のインスタンスは作成できません。

The `DraftPost` struct has an `add_text` method, so we can add text to
`content` as before, but note that `DraftPost` does not have a `content` method
defined! So now the program ensures all posts start as draft posts, and draft
posts don’t have their content available for display. Any attempt to get around
these constraints will result in a compiler error.

> `DraftPost` 構造体には `add_text` メソッドがあるので、以前のように `content` にテキストを追加することができますが、 `DraftPost` には `content` メソッドが定義されていないことに注意してください！ つまり、このプログラムは、すべての記事が下書き記事として始まること、および下書き記事のコンテンツが画面に表示されないことを保証します。これらの制約を回避しようとすると、コンパイラエラーになります。

#### Implementing Transitions as Transformations into Different Types

So how do we get a published post? We want to enforce the rule that a draft
post has to be reviewed and approved before it can be published. A post in the
pending review state should still not display any content. Let’s implement
these constraints by adding another struct, `PendingReviewPost`, defining the
`request_review` method on `DraftPost` to return a `PendingReviewPost`, and
defining an `approve` method on `PendingReviewPost` to return a `Post`, as
shown in Listing 17-20:

> では、どうすれば公開済み記事を得られるでしょうか？ 公開可能になる前に、下書き記事はレビューと承認を受けなければならないというルールを強制したいです。レビュー待ち状態の記事は、まだコンテンツを表示してはいけません。これらの制約を実装してみましょう。
>
> - `PendingReviewPost` という別の構造体を追加する
> - `DraftPost` の `request_review` メソッドが `PendingReviewPost` を返すようにする
> - `PendingReviewPost` の `approve` メソッドが `Post` を返すようにする
>
> → リスト17-20

<span class="filename">Filename: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch17-oop/listing-17-20/src/lib.rs:here}}
```

<span class="caption">Listing 17-20: A `PendingReviewPost` that gets created by
calling `request_review` on `DraftPost` and an `approve` method that turns a
`PendingReviewPost` into a published `Post`</span>

The `request_review` and `approve` methods take ownership of `self`, thus
consuming the `DraftPost` and `PendingReviewPost` instances and transforming
them into a `PendingReviewPost` and a published `Post`, respectively. This way,
we won’t have any lingering `DraftPost` instances after we’ve called
`request_review` on them, and so forth. The `PendingReviewPost` struct doesn’t
have a `content` method defined on it, so attempting to read its content
results in a compiler error, as with `DraftPost`. Because the only way to get a
published `Post` instance that does have a `content` method defined is to call
the `approve` method on a `PendingReviewPost`, and the only way to get a
`PendingReviewPost` is to call the `request_review` method on a `DraftPost`,
we’ve now encoded the blog post workflow into the type system.

> `request_review` と `approve` メソッドは、 `self` の所有権を取得し、 `DraftPost` と `PendingReviewPost` のインスタンスを消費して、それぞれ `PendingReviewPost` と公開済みの `Post` に変換します。こうすることで、 `request_review` を呼び出した後に `DraftPost` インスタンスが残ってしまうことがなくなります。 `PendingReviewPost` 構造体には `content` メソッドが定義されていないので、その内容を読み取ろうとすると、 `DraftPost` と同様に、コンパイラエラーになります。 `content` メソッドが定義されている公開された `Post` インスタンスを取得する唯一の方法は、 `PendingReviewPost` の `approve` メソッドを呼び出すことであり、 `PendingReviewPost` を取得する唯一の方法は、 `DraftPost` の `request_review` メソッドを呼び出すことであるため、これでブログ記事のワークフローを型システムにエンコードすることができました。

But we also have to make some small changes to `main`. The `request_review` and
`approve` methods return new instances rather than modifying the struct they’re
called on, so we need to add more `let post =` shadowing assignments to save
the returned instances. We also can’t have the assertions about the draft and
pending review post’s contents be empty strings, nor do we need them: we can’t
compile code that tries to use the content of posts in those states any longer.
The updated code in `main` is shown in Listing 17-21:

> しかし、 `main` にもいくつかの小さな変更を加える必要があります。 `request_review` と `approve` メソッドは、呼び出された構造体を変更するのではなく、新しいインスタンスを返すので、返されたインスタンスを保存するために、 `let post =` のシャドーイング代入を追加する必要があります。また、下書きとレビュー待ちのコンテンツについて、空文字列によるアサーションはできませんし、その必要もありません: これらの状態の記事のコンテンツを使用しようとするコードは、もはやコンパイルできません。 `main` の更新されたコードをリスト17-21に示します:

<span class="filename">Filename: src/main.rs</span>

```rust,ignore
{{#rustdoc_include ../listings/ch17-oop/listing-17-21/src/main.rs}}
```

<span class="caption">Listing 17-21: Modifications to `main` to use the new
implementation of the blog post workflow</span>

The changes we needed to make to `main` to reassign `post` mean that this
implementation doesn’t quite follow the object-oriented state pattern anymore:
the transformations between the states are no longer encapsulated entirely
within the `Post` implementation. However, our gain is that invalid states are
now impossible because of the type system and the type checking that happens at
compile time! This ensures that certain bugs, such as display of the content of
an unpublished post, will be discovered before they make it to production.

> `post` の再代入のために `main` に加えた変更は、この実装がオブジェクト指向の State パターンに全く従っていないことを意味します: 状態間の変換 (transformations) は、もはや完全には `Post` の実装にカプセル化されていません。しかし、型システムとコンパイル時の型チェックにより、無効な状態が発生し得なくなったのは収穫です! これにより、未公開記事のコンテンツが表示されるなどの特定のバグが、製品化より前に確実に発見されます。

Try the tasks suggested for additional requirements that we mentioned at the
start of this section on the `blog` crate as it is after Listing 17-20 to see
what you think about the design of this version of the code. Note that some of
the tasks might be completed already in this design.

> このセクション冒頭の追加要件提案を、リスト17-20以降の `blog` クレートで試して、このバージョンのコード設計についてどう思うか確認してみてください。なお、いくつかのタスクはこの設計ですでに完了しているかもしれません。

We’ve seen that even though Rust is capable of implementing object-oriented
design patterns, other patterns, such as encoding state into the type system,
are also available in Rust. These patterns have different trade-offs. Although
you might be very familiar with object-oriented patterns, rethinking the
problem to take advantage of Rust’s features can provide benefits, such as
preventing some bugs at compile time. Object-oriented patterns won’t always be
the best solution in Rust due to certain features, like ownership, that
object-oriented languages don’t have.

> Rust はオブジェクト指向のデザインパターンを実装できるにもかかわらず、型システムに状態をエンコードするような他のパターンも Rust では利用できることを見てきました。これらのパターンには、異なるトレードオフがあります。オブジェクト指向のパターンには慣れているかもしれませんが、 Rust の機能を活用して問題を再考することで、コンパイル時のバグを防ぐことができるなどのメリットがあります。オブジェクト指向言語にはない所有権などの機能があるため、 Rust ではオブジェクト指向パターンが常に最良の解決策になるとは限りません。

## Summary

No matter whether or not you think Rust is an object-oriented language after
reading this chapter, you now know that you can use trait objects to get some
object-oriented features in Rust. Dynamic dispatch can give your code some
flexibility in exchange for a bit of runtime performance. You can use this
flexibility to implement object-oriented patterns that can help your code’s
maintainability. Rust also has other features, like ownership, that
object-oriented languages don’t have. An object-oriented pattern won’t always
be the best way to take advantage of Rust’s strengths, but is an available
option.

> この章を読んで、 Rust がオブジェクト指向言語だと思ったかどうかはさておき、 Rust ではトレイトオブジェクトを使っていくつかのオブジェクト指向の機能を得られることがわかったはずです。動的ディスパッチは、実行時のパフォーマンスを少し落とす代わりに、コードに柔軟性を与えることができます。この柔軟性を利用して、コードの保守性を高めるオブジェクト指向パターンを実装することができます。また、 Rust には所有権のような、オブジェクト指向言語にはない機能があります。オブジェクト指向パターンは、 Rust の強みを生かすための最良の方法とは限りませんが、選択肢のひとつです。（訳注: 所有権は機能というより、メモリ安全性の代償として課される制約と見るのが妥当でしょう。所有権を全く意識せずにコードが書けたらどれだけラクでしょうか。）

Next, we’ll look at patterns, which are another of Rust’s features that enable
lots of flexibility. We’ve looked at them briefly throughout the book but
haven’t seen their full capability yet. Let’s go!

> 次はパターン（マッチング）を見てみましょう。パターンは Rust の機能の 1 つで、柔軟性に富んでいます。この本ではパターンを簡単に紹介してきましたが、まだその完全な性能を見たわけではありません。 Let's go!

[more-info-than-rustc]: ch09-03-to-panic-or-not-to-panic.html#cases-in-which-you-have-more-information-than-the-compiler
[macros]: ch19-06-macros.html#macros
