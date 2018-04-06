# sopstool

[![Build Status](https://travis-ci.org/Ibotta/sopstool.svg?branch=master)](https://travis-ci.org/Ibotta/sopstool) [![Maintainability](https://api.codeclimate.com/v1/badges/addf39da73692548e1e3/maintainability)](https://codeclimate.com/github/Ibotta/sopstool/maintainability) [![Test Coverage](https://api.codeclimate.com/v1/badges/addf39da73692548e1e3/test_coverage)](https://codeclimate.com/github/Ibotta/sopstool/test_coverage)

sopstool is a multi-file wrapper around [sops](https://github.com/mozilla/sops). It uses the sops binary to encrypt and decrypt files, and piggybacks off the .sops.yaml configuration file.

sopstool provides functionality to manage multiple secret files at once, and even use as an entrypoint to decrypt at startup, for container images.  Much of this behavior is inspired by the great [blackbox project](https://github.com/StackExchange/blackbox).

## Installation

The quickest install uses a shell script hosted in this repo. This script will install the latest sops (if the command does not exist) and sopstool to `./bin` by default.

```sh
curl https://raw.githubusercontent.com/Ibotta/sopstool/master/install.sh | bash
```

* Override the sops version with the environment variable `SOPS_VERSION`
* Override the sopstool version with the environment variable `SOPSTOOL_VERSION`
* Override the binary install location with the first shell argument
  * remember, you may need `sudo` or root access if you are installing to `/usr/*`

Example with overrides:

```sh
curl https://raw.githubusercontent.com/Ibotta/sopstool/master/install.sh | SOPS_VERSION=3.0.0 SOPSTOOL_VERSION=0.0.1 bash /usr/local/bin
```

### Homebrew

Ibotta maintains a tap for their opensource projects, which includes sopstool. This will install sops as a requirement

```sh
brew install Ibotta/public/sopstool
```

### Installing sops manually

Since sopstool requires sops, install it first. You can install it by hand [from a github release](https://github.com/mozilla/sops/releases), or

#### installing the sops binary with our script installer

The install script above uses a separate script to download sops

```sh
curl https://raw.githubusercontent.com/Ibotta/sopstool/master/sopsinstall.sh | bash
```

* Override the tag with the first shell argument (defaults to latest)
* Override the binary install location with the -b flag (defaults to `/.bin`)

(This script was generated by [godownloader](https://github.com/goreleaser/godownloader))

#### installing sops using go (master branch)

```sh
go get -u go.mozilla.org/sops/cmd/sops
```

### Installing sopstool manually

Download the latest version for your platform from our [RELEASES](https://github.com/Ibotta/sopstool/releases) or follow one of the below to install the latest

Following the lead of [sops](https://github.com/mozilla/sops), we only build 64bit binaries.

### installing the sopstool binary using our script installer

The install script above uses a separate script to download sopstool

```sh
curl https://raw.githubusercontent.com/Ibotta/sopstool/master/sopstoolinstall.sh | bash
```

* Override the tag with the first shell argument (defaults to latest)
* Override the binary install location with the -b flag (defaults to `/.bin`)

(This script was generated by [godownloader](https://github.com/goreleaser/godownloader))

### installing sopstool using go (master branch)

```sh
go get -u github.com/Ibotta/sopstool
```

## Usage

This is a package that builds a single binary (per architecture) for wrapping [sops](https://github.com/mozilla/sops) with multi-file capabilities.

for more details, use the built-in documentation on commands:

```sh
sopstool -h
```

to get the shell completion helpers:

```sh
#!/usr/bin/env bash
sopstool completion
```

```sh
#!/usr/bin/env zsh
sopstool completion --sh zsh
```

## Configuration

1. use a [`.sops.yaml`](https://github.com/mozilla/sops#using-sops-yaml-conf-to-select-kms-pgp-for-new-files) file
    * this will be at the root of your project. this file is used to both configure keys as well as hold the list of files managed.
    * it needs to specify at least one KMS key accessible by your environment

        ```yaml
        creation_rules:
          - kms: arn:aws:kms:REGION:ACCOUNT:key/KEY_ID
        ```

    * it can specify more complex cases of patterns vs keys too (see link)

## How-To

1. Create a [KMS Key](https://aws.amazon.com/kms/).
1. Follow along the [Configuration Steps](https://github.com/Ibotta/sopstool/tree/master/#configuration), and place the `.sops.yaml` file at the root directory where your scripts will run.
    * All files added to SOPS are relative, or in child directories to the `.sops.yaml` configuration file.
1. Create a file to encrypt(any extension other than `.yaml` if you wish to do the **ENTIRE** file), or create a yaml file with `key: value` pairs(and make sure it's extension is `.yaml`). Sops will encrypt the values, but not it's keys.
    * You can read more about [SOPS Here](https://github.com/mozilla/sops).
1. At this point, `sopstool` is ready and you can now `sopstool add filename`. You'll notice it will create a `filename.sops.extension`. This is your newly encrypted file.
    * When your files are properly encyrepted, you can run `sopstool clean` to remove the original plain text secret files.
1. Now, you can interact via the command line in various ways.
    * **Editing an encrypted file** - `sopstool edit filename.sops.extension`. You can also use your original filename too! `sopstool edit filename.extension`
    * **Listing all encrypted files** - `sopstool list`
    * **Removing encrypted file** - `sopstool remove filename.extension`
    * **Display the contents of encrypted file** - `sopstool cat filename.extension`

### Walkthrough

In this walkthrough, we will go through the steps required to get a secure yaml configuration file running.

1. Configure your `.sops.yaml`

    ```yaml
    # .sops.yaml
    creation_rules:
      - kms: arn:aws:kms:REGION:ACCOUNT:key/KEY_ID
    ```

1. Create a secrets yaml configuration file

    ```yaml
    # credentials.yaml
    database.password: supersecretdb
    database.user: supersecretpassword
    redshift:
      user: my.user.name
      password: my.password
    ```

1. Encrypt the newly created file

    ```sh
    sopstool add credentials.yaml
    ```

1. Create a sample script

    ```python
    # myscript.py
    import yaml
    with open('credentials.yaml', 'r') as f:
        credentials = yaml.load(f)

    print credentials["database.user"]
    print credentials["database.password"]
    print credentials["redshift"]["user"]
    print credentials["redshift"]["password"]
    ```

1. Here is what your folder structure would look like to this point(after deleting the unencrypted credentials.yaml file)

    ```text
    my-project/
    ├── .sops.yaml
    ├── credentials.sops.yaml
    └── myscript.py
    ```

1. Accessing credentials

    The flow should be as follows: unencrypt credentials -> run script -> destroy credentials. You can use the `sopstool entrypoint` to achieve this.

    ```sh
    sopstool entrypoint python myscript.py
    ```

## Contributing

Bug reports and pull requests are welcome at <https://github.com/Ibotta/sopstool>

### docs

Generate markdown docs for the commands via

```sh
sopstool docs
```
