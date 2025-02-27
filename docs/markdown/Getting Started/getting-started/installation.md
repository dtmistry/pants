---
title: "Installing Pants"
slug: "installation"
hidden: false
createdAt: "2020-02-21T17:44:53.022Z"
updatedAt: "2022-07-12T00:00:00.000Z"
---

To install the `pants` binary you can use:

```
/bin/bash -c "$(curl -fsSL https://static.pantsbuild.org/setup/pantsup.sh)" 
```

This will install `pants` into `~/bin`, which must be on your PATH. The installer script will warn you if it is not.

Alternatively, on macOS you can also use homebrew to install `pants`:

```
brew install pantsbuild/tap/pants
```

`pants` is a launcher binary that delegates to the underlying version of Pants in each repo. This allows you to have multiple repos, each using an independent version of Pants.

- If you run `pants` in a repo that is already configured to use Pants, it will read the repo's Pants version from the `pants.toml` config file, install that version if necessary, and then run it.

- If you run `pants` in a repo that is not yet configured to use Pants, it will prompt you to set up a skeleton `pants.toml` that uses that latest stable version of Pants. You'll then need to edit that config file to add [initial configuration](doc:initial-configuration).

If you have difficulty installing Pants, see our [getting help](doc:getting-help) for community resources to help you resolve your issue.

> 👍 Upgrading Pants
> 
> The `pants` launcher binary will automatically install and use the Pants version specified in `pants.toml`, so upgrading Pants in a repo is as simple as editing `pants_version` in that file.
>
> To upgrade the `pants` launcher binary itself, run
> ```
> SCIE_BOOT=update pants
> ```

Running Pants from unreleased builds
------------------------------------

To use an unreleased build of Pants from the [pantsbuild/pants](https://github.com/pantsbuild/pants) main branch, locate the main branch SHA, set `PANTS_SHA=<SHA>` in the environment, and run `pants` as usual:

```
PANTS_SHA=8553e8cbc5a1d9da3f84dcfc5e7bf3139847fb5f pants --version
```

Running Pants from sources
--------------------------

See [here](doc:running-pants-from-sources) for instructions on how to run Pants directly from its [sources](https://github.com/pantsbuild/pants).

This is useful when making changes directly to Pants, to see how those changes impact your repo.


> 🚧 The old `./pants` script
>
> Before the creation of the `pants` launcher binary, the recommended way of installing Pants was to check a `./pants` launcher script into each repo. This script required an external Python interpreter, and was prone to errors and issues related to discovery and use of this interpreter. 
> 
> The `pants` launcher binary uses an embedded interpreter and does not rely on one being present on the system (although if your repo contains Python code then it naturally requires a Python interpreter).
> 
> We strongly recommend removing the `./pants` script from your repo and using the `pants` binary instead. You can keep a simple `./pants` script that delegates to `pants` to ease the transition. However, if you do need to continue to use the old installation method for some reason, it is described [here](doc:manual-installation). But please [let us know](doc:getting-help) so we can accommodate your use case in the launcher binary.

The `pants` binary's implementation
-----------------------------------

You don't need to know this to use `pants`, but it may be of interest:

The `pants` launcher binary is also known as [scie-pants](https://github.com/pantsbuild/scie-pants) (pronounced "ski pants"), It's implemented using [scie](https://github.com/a-scie/jump), a Self Contained Interpreted Executable launcher. scie is what allows `pants` to embed its own Python interpreter, instead of relying on a specific interpreter being available on your PATH.

In fact, instead of literally embedding an interpreter in the `pants` binary, which would inflate its size, the binary is "hollowed out": Instead of the interpreter itself it contains metadata on how to download a platform-specific [standalone interpreter executable](https://gregoryszorc.com/docs/python-build-standalone/main/). The scie mechanism then downloads that interpreter file on first use, and caches it for future use. So if you update the `pants` launcher binary, you don't have to re-download the interpreter.

See the links above for more details. We hope to soon add support in Pants for building scies out of your code, which will allow you to package and ship fully standalone Python binaries!
