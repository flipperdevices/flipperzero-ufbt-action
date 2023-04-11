## GitHub Action for [ufbt, micro Flipper Build Tool](https://pypi.org/project/ufbt/)

This action brings all features of [ufbt](https://pypi.org/project/ufbt/) to GitHub Workflows. It is used to build applications for [Flipper Zero](https://flipperzero.one/). Building applications with `ufbt` is much faster than using `fbt` with [full firmware sources](https://github.com/flipperdevices/flipperzero-firmware), and it allows you to build applications for different versions of the firmware, including unofficial ones.

This action and has 3 modes of operation:

* **application builder**: builds .fap (Flipper Application Package) file for the application in specified directory and returns a list of built files;
* **linter**: runs linter (`clang-format`) on application's sources;
* **ufbt setup**: makes `ufbt` available in the environment, without running any tasks. This mode is used to set up `ufbt` command and SDK for other steps in the workflow. 

You can use [all SDK update modes](https://github.com/flipperdevices/flipperzero-ufbt/blob/dev/README.md#managing-the-sdk) supported by `ufbt`, including downloads from unofficial sources. See [SDK source options](#sdk-source-options) for action inputs used to specify SDK origin.

> This action caches toolchain used for `ufbt` operations, so it can be reused without being re-downloaded in subsequent runs of workflows in your repository.

## Usage 

To build your application with this action, you must create a workflow file in your repository. Place a file named `.github/workflows/build.yml` with one of the examples below, and, optionally, tune it to your needs.

### Basic example: build and lint

This example will build your application for bleeding edge version, `dev`, of the official firmware, and upload generated binaries to GitHub artifacts. It will also run linter on your sources and fail the workflow if there are any formatting errors.

```yaml
name: "FAP: Build and lint"
on: [push, pull_request]
jobs:
  ufbt-build-action:
    runs-on: ubuntu-latest
    name: 'ufbt: Build for Dev branch'
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Build with ufbt
        uses: hedger/flipperzero-ufbt-action@dev
        id: build-app
        with:
          # Set to 'release' to build for latest published release version
          sdk-channel: dev
      - name: Upload app artifacts
        uses: actions/upload-artifact@v3
        with:
          name: fap-${{ steps.build-app.outputs.suffix }}
          path: ${{ steps.build-app.outputs.fap-artifacts }}
      # You can remove this step if you don't want to check source code formatting
      - name: Lint sources
        uses: hedger/flipperzero-ufbt-action@dev
        with:
          # skip SDK setup, we already did it in previous step
          skip-setup: true
          task: lint
```

### Advanced example: build for multiple SDK sources

This example will build your application for 3 different SDK sources: `dev` and `release` channels of official firmware, and for an SDK from an unofficial source. It will upload generated binaries to GitHub artifacts.

```yaml
name: "FAP: Build for multiple SDK sources"
on: [push, pull_request]
jobs:
  ufbt-build-action:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - name: dev channel
            sdk-channel: dev
          - name: release channel
            sdk-channel: release
          - name: Unofficial firmware
            # example URL, replace with a valid one
            # you can also use other modes for specifying SDK sources
            sdk-index-url: https://up.unofficialflip.com/directory.json
            sdk-channel: dev
    name: 'ufbt: Build for ${{ matrix.name }}'
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Build with ufbt
        uses: hedger/flipperzero-ufbt-action@dev
        id: build-app
        with:
          sdk-channel: ${{ matrix.sdk-channel }}
          sdk-index-url: ${{ matrix.sdk-index-url }}
      - name: Upload app artifacts
        uses: actions/upload-artifact@v3
        with:
          name: fap-${{ steps.build-app.outputs.suffix }}
          path: ${{ steps.build-app.outputs.fap-artifacts }}
```

## Inputs

#### `task`

**Optional** Task to run. Can be `setup`, `build` or `lint`. Default is `"build"`.

#### `app-dir`

**Optional** Path to application's source code. Default is `"."` - the root of the repository.

#### `skip-setup`

**Optional** If set to `true`, skips SDK setup. Useful when running multiple action multiple times. Default is `false`.

#### SDK source options

Table below describes options for SDK update. See ufbt documentation on available [SDK update modes](https://github.com/flipperdevices/flipperzero-ufbt/blob/dev/README.md#managing-the-sdk) for details. All these inputs are **optional**.

| Input name      | `ufbt update` argument | Description |
| ---             | ---                    | --- |
| `sdk-channel`   | `--channel`            | Release channel to use. Can be `dev`, `rc` or `release` |
| `sdk-branch`    | `--branch`             | Branch to use |
| `sdk-index-url` | `--index-url`          | Index URL to use in branch and channel modes |
| `sdk-file`      | `--file`               | Path to a local SDK file |
| `sdk-url`       | `--url`                | Fixed URL to a remote SDK file |
| `sdk-hw-target` | `--hw-target`          | Hardware target to use |

## Outputs

#### `fap-artifacts`

A list of built `.fap` and `.fal` files. Can be used as input for `upload-artifact` action. Only available in `build` mode.

#### `fap-dir`

A directory with built artifacts and `.elf` files with debugging information. Only available in `build` mode.

#### `api-version`

API version used to build the application. Only available in `build` mode.

#### `suffix`

A suffix to use in artifact names, includes hardware target, API and SDK versions. Can be used for naming artifacts uploaded with `upload-artifact` action. Only available in `build` mode.

#### `lint-messages`

A list of linter messages, in case of any formatting errors. Only available in `lint` mode.

#### `ufbt-status`

A string, JSON object with ufbt status. Contains `ufbt status --json` output. Useful for extracting information about ufbt's directories, configured modes, etc. Useful in combination with [fromJSON() function](https://docs.github.com/en/actions/learn-github-actions/expressions#fromjson) to extract information from the output. For example, `${{ fromJSON(steps.<build>.ufbt-status).toolchain_dir }}`. Available in all modes. 

#### `toolchain-version`

A version of the toolchain used to build the application. Defined by current SDK. Available in all modes.

## Acknowledgements

First version of this action was created by [Oleksii Kutuzov](https://github.com/oleksiikutuzov) and is available [here](https://github.com/oleksiikutuzov/flipperzero-ufbt-action). However, it no longer works with the latest version of `ufbt`. This version is a complete rewrite, with problem matcher borrowed from original verison.