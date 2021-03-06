# Updating our version of Ruby

Before you do anything, check with Benoit Daloze for clearance to upgrade.

The workflow below will allow you to see and reapply the modifications that we
have to MRI source code while updating.

You can re-run these instructions at any time to compare against unmodified
MRI files.

## Create reference branches

For both the current version of Ruby you're using, and the new version, create
reference branches that include unmodified MRI sources.

Check out the version of Ruby you want to create the branch for in `../ruby`.

Then create the reference branch in the TruffleRuby repository

```bash
git checkout -b vNN
tool/import-mri-files.sh
git commit -am 'vNN'
```

You can then compare between these two branches and yours. For example to see
what changes you made on top of the old version, what's changed between the
old version and the new version, and so on. Keep them around while you do the
update.

## Update MRI with modifications

Install the target MRI version using the command `ruby-install ruby 2.6.6`.

In your working branch you can import MRI files again, and you can re-apply
old patches using the old reference branch.

```bash
tool/import-mri-files.sh
git revert vNN
```

You'll usually get some conflicts to work out.

## Update config_*.h files

Configuration files must be regenerated from ruby for Linux and macOS
and copied into `lib/cext/include/truffleruby`. In the MRI repository
do the following:

```
graalvm_clang=$(jt ruby -e 'puts RbConfig::CONFIG["CC"]')

autoconf
CC=$graalvm_clang ./configure
```

The output of configure should report that it has created or updated a
config.h file. For example

```
.ext/include/x86_64-linux/ruby/config.h updated
```

You will need to copy that file to
`lib/cext/include/truffleruby/config_linux.h` or
`lib/cext/include/truffleruby/config_darwin.h`.

After that you should clean your MRI source repository with:

```bash
git clean -Xdf
```

## Update libraries from third-party repos

Look in `../ruby/ext/json` to see the version of `flori/json` being used, and
then copy the original source of `flori/json` into `lib/json`.

## Updating default and bundled gems

You need a clean install (e.g., no extra gems installed) of MRI for this.

```
export TRUFFLERUBY=$(pwd)
rm -rf lib/gems/gems
rm -rf lib/gems/specifications

cd clean-install-of/ruby-n.n.n
cp -R lib/ruby/gems/*.0/gems $TRUFFLERUBY/lib/gems
cp -R lib/ruby/gems/*.0/specifications $TRUFFLERUBY/lib/gems

cd $TRUFFLERUBY
ruby tool/patch-default-gemspecs.rb
```

## Updating bundled gems

If the gem installs any executables like `rake` in `bin` ensure that the
shebang has a format as follows:

```bash
#!/usr/bin/env bash
#
# This file was generated by RubyGems.
# The above lines match the format expected by rubygems/installer.rb check_executable_overwrite
# bash section ignored by the Ruby interpreter

# get the absolute path of the executable and resolve symlinks
SELF_PATH=$(cd "$(dirname "$0")" && pwd -P)/$(basename "$0")
while [ -h "$SELF_PATH" ]; do
  # 1) cd to directory of the symlink
  # 2) cd to the directory of where the symlink points
  # 3) get the pwd
  # 4) append the basename
  DIR=$(dirname "$SELF_PATH")
  SYM=$(readlink "$SELF_PATH")
  SELF_PATH=$(cd "$DIR" && cd "$(dirname "$SYM")" && pwd)/$(basename "$SYM")
done
exec "$(dirname $SELF_PATH)/ruby" "$SELF_PATH" "$@"

#!ruby
# ^ marks start of Ruby interpretation

# ... the content of the executable
```

## Make other changes

In a separate commit, update all of these:

* Update `.ruby-version`, `TruffleRuby.LANGUAGE_VERSION/LANGUAGE_REVISION` and `versions.json`
* Copy and paste `-h` and `--help` output to `RubyLauncher`
* Copy and paste the TruffleRuby `--help` output to `doc/user/options.md`
* Update `doc/user/compatibility.md`
* Update `doc/legal/legal.md`
* Update `doc/contributor/stdlib.md`
* Update method lists - see `spec/truffle/methods_spec.rb`
* Run `jt test gems default-bundled-gems`
* Update `ci.jsonnet` to use the corresponding MRI version for benchmarking
* Grep for the old version with `git grep -F x.y.z`
* If `id.def` or `id.h` has changed, `jt build core-symbols` and check for correctness.
