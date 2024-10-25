# Subtensor Node

This directory contains the [Butane](https://coreos.github.io/butane/) config for the Inference Labs [subtensor node](https://docs.bittensor.com/subtensor-nodes/) running on [Fedora CoreOS](https://fedoraproject.org/coreos/).

Put your ssh public key(s) in `ssh-keys.pub`, Fedora CoreOS also supports downloading keys from links, for example: `https://github.com/username.keys`.
There should be one key or link per line.

Use `make ignition` to generate an ignition file for Fedora CoreOS.

To [provision on GCP](https://docs.fedoraproject.org/en-US/fedora-coreos/provisioning-gcp/), go under "Metadata" and create a key called "user-data" (w/o quotes), then paste the contents of the ignition file into the value field.
