---
layout: post
title:  "Duck Typing (horror) in Go"
date:   2024-08-20 20:19:12
author: C. Campo
categories: go 
---

If you say Go supports ["duck typing"][wiki] online, you'll probably get some
pushback. The Go community likes point out that Go can't support duck typing
because it's statically typed, and duck typing is a feature of dynamically typed
languages. Rather, Go has "structural typing" where interfaces are still 
implicitly implemented, but still checked at compile time...

...and yet I can still write this horror show:

{% highlight go %}
package main

import "fmt"

type Duck interface {
	Quack()
}

type Mallard struct{}

func (m Mallard) Quack() { fmt.Println("quack") }

type Dog struct{}

func (d Dog) Bark() { fmt.Println("woof") }

func main() {
	quack(Mallard{}, Dog{})
}

func quack(ducks ...any) {
	for _, duck := range ducks {
		duck.(Duck).Quack()
	}
}
{% endhighlight %}

Go ahead... [run it](https://go.dev/play/p/hi8vEivq29H)! 

Go can duck type with the best of em!

PS: please don't write real code like this.

[wiki]: https://github.com/ccampo133/go-docker-alpine-remote-debug
