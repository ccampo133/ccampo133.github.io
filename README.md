# ccampo.me

My website, hosted on GitHub pages. Heavily based on the [Centrarium][centrarium]
theme. Check out that page for more info on how to use said theme.

[centrarium]: https://github.com/bencentra/centrarium

## Development

See: https://github.com/envygeeks/jekyll-docker/blob/master/README.md

```sh
JEKYLL_VERSION=3.8
docker run -it --rm \
  --volume="$PWD:/srv/jekyll:Z" \
  --volume="$PWD/vendor/bundle:/usr/local/bundle:Z" \
  --publish "4000:4000" \
  jekyll/jekyll:$JEKYLL_VERSION \
  jekyll serve --drafts
```
