# Setting up this repo for UserClouds

I followed directions from https://github.com/slatedocs/slate/wiki/Using-Slate-Natively but found some missing gaps.

Mac OS has its own ruby version installed which you may not want to use, plus you'd have to `sudo` to make changes and install gems which is not recommended.

Instead we'll use `rbenv` to manage ruby versions (https://github.com/rbenv/rbenv). We'll also need `node`.

1. Run `brew install rbenv ruby-build node`

2. Set up `rbenv` in your `.bash_profile`:

```
# this sets up the "ruby shim" which allows for dynamically figuring out the right ruby version based on the presence of .ruby_version files
echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
source ~/.bash_profile
```

3. Now running `ruby` inside a repo should choose the right version, and we can follow the directions from the slate Repo by running:

```
gem update --system
gem install bundler
```

4. To run Slate in "interactive mode" (uses Middleman to run a webserver and compile your docs on the fly), run `bundle exec middleman server`.

5. To run Slate in "build" mode to emit static assets to the `build/` direcrory, run `bundle exec middleman build`.

6. You 