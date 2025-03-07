## How do I avoid running builders on unnecessary inputs?

Slow builds are often the result of builders that run on all Dart files in your
package, and analyze them. In this case you can speed up your builds by telling
those builders exactly which files they should run on.

You can do this in your `build.yaml` file, by configuring the `generate_for`
option of the builder:

```
targets:
  $default:
    builders:
      # Typically the builder key is just the package name, run
      # `dart run build_runner doctor` to check your config.
      <builder-key>:
        generate_for:
          # Example glob for only the Dart files under `lib/models`
          - lib/models/*.dart
```

## Where are the generated files?

There are 3 places you might see files getting generated. On _every_ build files
might go to _cache_ (`.dart_tool/build/generated/*`), and _source_ (the working
tree of your package). These are determined by the `build_to` configuration of
the Builders you are using. When a Builder specifies `build_to: source` you will
see them next to your other files but should not edit them by hand, and they are
generally meant to be published with your package. If you see a warning about
"conflicting outputs" they refer to _source_ outputs, because when the build
tool is starting with no prior information it can't tell if it might be
overwriting some file you wrote by hand.

Separately the `--output` option can specify a directory to create a _merged_
view of all the files in the build system. This copies _hand written source_
files, along with _source_ and _cache_ outputs and puts them all together in the
same directory structure.

## How can I debug my release mode web app (dart2js)?

By default, the `dart2js` compiler is only enabled in `--release` mode, which
does not include source maps or the original `.dart` files. If you need to debug
an error which only happens in `dart2js`, you will want to change your debug
mode compiler to `dart2js`. You can either do this using the `--define` command
line option:

```text
--define "build_web_compilers:entrypoint=compiler=dart2js"
```

Or by editing your `build.yaml` file:

```yaml
targets:
  $default:
    builders:
      build_web_compilers:entrypoint:
        options:
          compiler: dart2js
```

## How can I build with multiple configurations?

The build system supports two types of builds, "dev" and "release". By default
with `build_runner` the "dev" version is built by regardless of the command
used, build in release mode by passing the `--release` flag. With `webdev` the
default mode for the `serve` command is dev, and the default mode for the
`build` command is release. The `build` command can use dev mode with the
`--no-release` flag.

Options can be configured per mode, and they are
[merged by key](#how-is-the-configuration-for-a-builder-resolved) with the
defaults provided by the builder and global overrides. The `options` field
defines configuration used in all modes, and the `dev` and `release` fields
defines the overrides to those defaults for the specific mode chosen. Builders
can define their own defaults by mode which is overridden by user config. For
example `build_web_compilers` defines options that use `dartdevc` compiler in
dev mode, and `dart2js` in release mode.

The following configuration builds with `dart2js` always, passes `--no-minify`
in dev mode, and passed `-O3` in release mode:

```yaml
targets:
  $default:
    builders:
      build_web_compilers:entrypoint:
        options:
          compiler: dart2js
        dev_options:
          dart2js_args:
          - --no-minify
        release_options:
          dart2js_args:
          - -O3
```

If you need other configurations in addition to dev and release, you can define
multiple `build.yaml` files. For instance if you have a `build.debug.yaml` file
you can build with `--config debug` and this file will be used instead of the
default `build.yaml`. The dev and release flavors still apply. `dart run
build_runner serve --config debug` will use the `dev_options` in
`build.debug.yaml`, while `dart run build_runner build --config debug --release`
will use the `release_options` in `build.debug.yaml`.

Only one build flavor can be built at a time. It is not possible to have
multiple targets defined which set different builder options for the same set of
sources. Builds will overwrite generated files in the build cache, so flipping
between build configurations may be less performant than building the same build
configuration repeatedly.

## How is the configuration for a builder resolved?

Builders are constructed with a map of options which is resolved from the
builder specified defaults and user overrides. The configuration is specific to
a `target` and [build mode](#how-can-i-build-with-multiple-configurations). The
configuration is "merged" one by one, where the higher precedence configuration
overrides values by String key. The order of precedence from lowest to highest
is:

-   Builder defaults without a mode.
-   Builder defaults by mode.
-   Target configuration without a mode.
-   Target configuration by mode.
-   Global options without a mode.
-   Global options by mode.
-   Options specified on the command line.

For example:

```yaml
builders:
  some_builder:
    # Some required fields omitted
    defaults:
      options:
        some_option: "Priority 0"
      release_options:
        some_option: "Priority 1"
      dev_options:
        some_option: "Priority 1"
targets:
  $default:
    builders:
      some_package:some_builder:
        options:
          some_option: "Priority 2"
        release_options:
          some_option: "Priority 3"
        dev_options:
          some_option: "Priority 3"

global_options:
  some_package:some_builder:
    options:
      some_option: "Priority 4"
    release_options:
      some_option: "Priority 5"
    dev_options:
      some_option: "Priority 5"
```

And when running the build:

```
dart run build_runner build --define=some_package:some_builder=some_option="Priority 6"
```

## How can I include additional sources in my build?

The `build_runner` package defaults the included source files to directories
derived from the
[package layout conventions](https://dart.dev/tools/pub/package-layout).

If you have additional files which you would like to be included as part of the
build, you will need a custom `build.yaml` file. You will want to modify the
`sources` field on the `$default` target:

```yaml
targets:
  $default:
    sources:
      - my_custom_sources/**
      - lib/**
      - web/**
      # Note that it is important to include these in the default target.
      - pubspec.*
      - $package$
```

## Why do Builders need unique outputs?

`build_runner` relies on determining a static build graph before starting a
build - it needs to know every file that may be written and which Builder would
write it. If multiple Builders are configured to (possibly) output the same file
you can:

-   Add a `generate_for` configuration for one or both Builders so they do not
    both operate on the same primary input.
-   Disable one of the Builders if it is unneeded.
-   Contact the author of the Builder and ask that a more unique output
    extension is chosen.
-   Contact the author of the Builder and ask that a more unique input extension
    is chose, for example only generating for files that end in
    `_something.dart` rather than all files that end in `.dart`.

## How can I use my own development server to serve generated files?

There are 2 options for using a different server during development:

1.  Run `build_runner serve web:<port>` and proxy the requests to it from your
    other server. This has the benefit of delaying requests while a build is
    ongoing so you don't get an inconsistent set of assets.

2.  Run `build_runner watch --output web:build` and use the created `build/`
    directory to serve files from. This will include a `build/packages`
    directory that has these files in it.

## How can I fix `AssetNotFoundException`s for swap files?

Some editors create swap files during saves, and while build_runner uses some
heuristics to try and ignore these, it isn't perfect and we can't hardcode
knowledge about all editors.

One option is to disable this feature in your editor. Another option is you can
explicitly ignore files with a given extension by configuring the `exclude`
option for your targets `sources` in `build.yaml`:

```yaml
targets:
  $default:
    sources:
      exclude:
        # Example that excludes intellij's swap files
        - **/*___jb_tmp___
```

## Why are some logs "(cached)"?

`build_runner` will only run actions that have changes in their inputs. When an
action fails, and a subsequent build has exactly the same inputs for that action
it will not be rerun - the previous error messages, however, will get reprinted
to avoid confusion if a build fails with no printed errors. To force the action
to run again make an edit to any file that is an input to that action, or throw
away all cached values with `dart run build_runner clean` before starting the
next build.

## How can I resolve "Skipped compiling" warnings?

These generally come up in the context of a multi-platform package (generally
due to a mixture of vm and web tests), and look something like this:

```text
[WARNING] build_vm_compilers:entrypoint on example|test/my_test.dart:

Skipping compiling example|test/my_test.dart for the vm because some of its
transitive libraries have sdk dependencies that not supported on this platform:

example|test/imports_dart_html.dart
```

While we are smart enough not to attempt to compile your web tests for the vm,
it is slow for us to figure that out so we print this warning to encourage you
to set up the proper configuration so we will never even attempt to compile
these tests.

You can set up this configuration in your `build.yaml` file, using the
`generate_for` option on the builder. It helps a lot if you separate your web
and vm tests into separate directories, but you don't have to.

For example, your `build.yaml` might look like this:

```yaml
targets:
  $default:
    builders:
      build_web_compilers:entrypoint:
        generate_for:
        - test/multiplatform/**_test.dart
        - test/web/**_test.dart
        - web/**.dart
      build_vm_compilers:entrypoint:
        generate_for:
        - test/multiplatform/**_test.dart
        - test/vm/**_test.dart
        - bin/**.dart
```

## Why can't I see a file I know exists?

A file may not be served or be present in the output of a build because:

-   You may be looking for it in the wrong place. For example if a server for
    the `web/` directory is running on port `8080` then the file at
    `web/index.html` will be loaded from `localhost:8080/index.html`.
-   It may have be excluded from the build entirely because it isn't present as
    in the `sources` for any `target` in `build.yaml`. Only assets that are
    present in the build (as either a source or a generated output from a
    source) can be served.
-   It may have been removed by a `PostProcessBuilder`. For example in release
    modes, by default, the `build_web_compilers` package enables a
    `dart_source_cleanup` builder that removes all `.dart` source files.

## Configuring the number of compiler processes

Some builders run multiple compiler processes in order to speed up compilation.

The amount of parallelism per task can be configured using the environment
variable `BUILD_MAX_WORKERS_PER_TASK`.

The "tasks" in this case refer to different types of compilation (ie: different
compilers). There are least 3 different compilers which may be a part of any
given build - `kernel`, `dartdevc`, and `dart2js`.

## How can I reduce the amount of memory used by the build process?

By default most files in a typical build are cached in memory, but this can
cause problems in memory constrained environments (such as CI systems).

You can pass the `--low-resources-mode` to disable this file caching.

We may add future optimizations for this mode as well, with the general
principle being making the tradeoff of worse build times for less resource
usage on the machine.

See also [Configuring the number of compiler processes](#configuring-the-number-of-compiler-processes).
