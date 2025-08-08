# aarongodfrey.dev

A **Jekyll** a static site generator for my personal blog.

## Fresh Install

If you are doing a fresh install you will need to install ruby and ruby-dev.

```bash
$ sudo apt install ruby ruby-dev
```

Next install the `gitub-pages` and `bundler`.

````bash
$ sudo gem install github-pages bundler
```

Finally install.

```bash
$ bundle install
```

## Running Locally

```bash
$ bundle exec jekyll serve --incremental --drafts --watch --future
```

## Updating Gems and Theme

```bash
$ bundle update
```

## Scrubbing node-red exported flows

[Scrubber](https://zachowj.github.io/node-red-contrib-home-assistant-websocket/scrubber/)
