# Fun with tests in U-Boot

**Last update**: 2026-05-31

## Introduction

U-Boot includes a pretty complete test suite, with some tests also executable
in a "sandbox" environment, in which U-Boot is run as a regular application on
the host system.

Some notes on the usage of this test suite follow.

## Environment setup

The simplest way to setup a proper environemnt for U-Boot testing is creating a
Python environment containing all the required dependencies:

```sh
python3 -m venv .test-venv
source .test-venv/bin/activate
pip install pytest pytest-xdist coverage fattools filelock setuptools pycryptodomex pyelftools yamllint
```

Additional dependencies can be installed through the system package manager:

```sh
dnf install erofs-utils gdisk squashfs-tools qemu-img vboot-utils
```

## Sandbox tests

Sandbox tests, which only require pytest and a bunch of other dependencies,
can be run with a simple:

```sh
./test/py/test.py --bd sandbox --build --basetemp=$(pwd)/tmpdir -v
```

> The `--basetemp=$(pwd)/tmpdir` instruction is not strictly required, but can
  help whenever the space reserved for `/tmp` is limited (e.g., when the test
  fail with `Disk quota exceeded`).
