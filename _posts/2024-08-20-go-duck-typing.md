---
layout: post
title: "Duck Typing (horror) in Go"
date: 2024-08-20 20:19:12
author: C. Campo
categories: go 
---

If you say Go supports ["duck typing"][wiki] online, you'll probably get some
pushback. The Go community often points out that Go doesn't support duck typing
because it's statically typed, and duck typing is a feature of dynamically typed
languages. Rather, Go has "structural typing" where interfaces are still
implicitly implemented, but still checked at compile time.

Here's an example of structural typing in Go:

{% highlight go %}
package main

import "fmt"

type Duck interface {
	Swim()
	Quack()
}

type Mallard struct{}

func (Mallard) Swim() { fmt.Println("mallard swimming") }

func (Mallard) Quack() { fmt.Println("mallard quacking") }

type Goose struct{}

func (Goose) Swim() { fmt.Println("goose swimming") }

func (Goose) Quack() { fmt.Println("goose quacking") }

func quack(ducks ...Duck) {
	for _, duck := range ducks {
		duck.Swim()
		duck.Quack()
	}
}

func main() {
	quack(Mallard{}, Goose{})
}
{% endhighlight %}

Give a [try](https://go.dev/play/p/fYrgjjEgxXi).

Both `Mallard` and `Goose` implicitly implement the `Duck` interface, so they
can be assumed to be a `Duck` at compile time, and in this example, in the
`quack` function. A `Goose` swims like a `Duck` and quacks like a `Duck`, so it
must be a `Duck`!

So how isn't this duck typing? Well, it's a bit of a pedantic argument, but
duck typing is typically associated with dynamically typed languages, where the
types are determined at runtime rather than compile time like here.

From [Wikipedia](https://en.wikipedia.org/wiki/Duck_typing#Structural_type_systems):

> Duck typing is similar to, but distinct from, structural typing. Structural
> typing is a _static typing system_ that determines type compatibility and
> equivalence by a type's structure, whereas duck typing is _dynamic_ and
> determines type compatibility by only that part of a type's structure that is
> accessed during _runtime_.

With that definition in hand, can we do duck typing in Go? Well, when using the
language as intended, **no**. But if you're willing to hack around the type
system a bit, you can still write this horror show:

{% highlight go %}
package main

import "fmt"

type Duck interface {
	Swim()
	Quack()
}

type Mallard struct{}

func (Mallard) Swim() { fmt.Println("mallard swimming") }

func (Mallard) Quack() { fmt.Println("mallard quacking") }

type Dog struct{}

func (Dog) Swim() { fmt.Println("dog swimming") }

func (Dog) Bark() { fmt.Println("woof") }

func quack(ducks ...any) {
	for _, duck := range ducks {
		duck.(interface { Swim() }).Swim()
		duck.(interface { Quack() }).Quack()
	}
}

func main() {
	quack(Mallard{}, Dog{})
}
{% endhighlight %}

Go ahead... [run it](https://go.dev/play/p/FNg_xqIfbwy)!

I would say that Go can duck type with the best of em!

PS: please don't write real code like this! I'm just having some in this post!
It's really a bad idea. Don't do it. This example is intended to panic!

My point is just to show that while it is often said that Go doesn't support
duck typing, you can write something that looks awfully a lot like duck typing
if you (mis)use some of the more "dynamic" features of the language like `any`
and type assertions and hack around the type system. It doesn't even require
reflection, as one might expect, such as in Java<sup>[1][java_reflect],
[2][java_proxy]</sup>. It is essentially the Go analogue of the
[Wikipedia duck typing example in Python](https://en.wikipedia.org/wiki/Duck_typing#Example).
And if the code looks and behaves like duck typing, then it must duck typing ;).

Please note that there are _very valid_ use cases for `any` and type assertions,
but duck typing is not one of them!

Of course, the "correct" implementation of the `quack` function would be to
accept `Duck` instead of `any` and ditch the type assertions, which will result
a _compile_ time error. This is the first example in this post. 

Even if for whatever reason you were stuck with using `any`, you could still use
the [two-valued type assertion][assert] (or a [type switch][ts]) to check if the
type implements `Duck`, which of course is still a runtime check, but at least
it avoids the panic:

{% highlight go %}
func quack(ducks ...any) {
	for _, duck := range ducks {
		if d, ok := duck.(Duck); ok {
			d.Swim()
			d.Quack()
		} else {
			fmt.Println("not a duck")
		}
	}
}
{% endhighlight %}

See [here](https://go.dev/play/p/uK8fmJi560W) for the output.

Still, always prefer using the type system and the compiler to your advantage
whenever possible!

Thanks for reading!

[wiki]: https://github.com/ccampo133/go-docker-alpine-remote-debug

[java_reflect]: https://stackoverflow.com/a/1079799

[java_proxy]: https://thinking-in-code.blogspot.com/2008/11/duck-typing-in-java-using-dynamic.html

[assert]: https://go.dev/tour/methods/15

[ts]: https://go.dev/tour/methods/16
