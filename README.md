# Personal Blog




## Theme Source

- [GH Repo][chirpy]
- [Gem][gem]
- [Tips and Best Practices][chipy-page]
- [Upgrading][upgrade]




## Building / Testing Locally


#### On MacOS

Check ruby version with `ruby -v` and if it returns `(ruby 2.6.10p210)`:

1. Install Ruby package manager (I chose `frum`)
    - install a new ruby version with `$ frum install`
    - its important that we don't `brew install` a newer ruby version so it doesn't clash with the system version
2. Make sure that `frum` is evaluated in the shell environment before you run any of the below commands
    - `$ eval "$(frum init)"`



### Installing Dependencies
```terminal
$ bundle
```
> only run bundle before running the local server for the first time



### Running Local Server

**For fully regenerative testing**:
```terminal
$ bundle exec jekyll s
```
> alternatively create a script that will verify ruby version and run the above command -> `run.sh`


**For incremental testing**:
```terminal
$ bundle exec jekyll s --incremental
```
> there is some weirdness sometimes with incremental updating, if something is not updating, run the "fully regenerative" command to "update" everything



[chirpy]: https://github.com/cotes2020/jekyll-theme-chirpy/
[gem]: https://rubygems.org/gems/jekyll-theme-chirpy
[chipy-page]: https://chirpy.cotes.page/
[upgrade]: https://github.com/cotes2020/jekyll-theme-chirpy/wiki/Upgrade-Guide#upgrade-from-starter
[use-template]: https://github.com/cotes2020/chirpy-starter/generate
[mit]: https://github.com/cotes2020/chirpy-starter/blob/master/LICENSE

