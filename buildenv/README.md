== Build Environment

This directory contains docker configuration for building the environment
in which frugen/libfru builds can be tested

Example:
```
OST=ubuntu
OSV=noble
docker --build-arg ostype=$OST --build-arg osver=$OSV -t frugen-buildenv:$OST-$OSV .
```

Supported OS types:

  - ubuntu
  - fedora

The above example builds an image based on Ubuntu Noble Numbat.
