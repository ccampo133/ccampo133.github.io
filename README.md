# ccampo.me

My website, hosted on GitHub pages. Heavily based on the [Centrarium][centrarium]
theme. Check out that page for more info on how to use said theme.

[centrarium]: https://github.com/bencentra/centrarium

## Development

Use RVM: 

* https://rvm.io/rvm/install

Install Ruby 2.X:

```sh
rvm install ruby-2.7.5

rvm use 2.7.5
```

Then install dependencies:

```sh
bundle install
```

Run locally (with draft posts):

```sh
bundle exec jekyll serve --drafts
```
