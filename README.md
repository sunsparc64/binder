<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2014, Joyent, Inc.
-->

# binder

This repository is part of the Joyent SmartDataCenter project (SDC).  For
contribution guidelines, issues, and general documentation, visit the main
[SDC](http://github.com/joyent/sdc) project page.

This repo contains 'binder', which is a DNS server implemented on top of
ZooKeeper.  Hosts use [registrar](http://github.com/joyent/registrar) to
register themselves into DNS.

See docs/index.md for more information.

# Repository

    boot/           Configuration scripts on zone setup.
    deps/           Git submodules (node et al).
    docs/           Project docs (restdown)
    lib/            Source files.
    node_modules/   Node.js deps, populated at build time.
    sapi_manifests/ SAPI manifests for zone configuration.
    smf/manifests   SMF manifests
    test/           Test suite (using nodeunit)
    tools/          Miscellaneous dev/upgrade/deployment tools and data.
    Makefile
    package.json    npm module info (holds the project version)
    README.md

# Development

To run the binder server:

    git clone git@github.com:joyent/binder.git
    cd binder
    git submodule update --init
    make all
    . ./env.sh
    ZK_HOST=<ZK IP address> node main.js 2>&1 | bunyan

To update the docs, edit "docs/index.md" and run `make docs`
to update "docs/index.html".

Before commiting/pushing run `make prepush` and, if possible, get a code
review.

# Testing

    ZK_HOST=<ZK IP address> make test
