# Databox Vagrant
> A complete Databox environment inside a Vagrant machine.

A Vagrant machine that sets up a complete Databox environment including:
- The [Databox Container Manager](https://github.com/me-box/databox-container-manager)
- The [Databox Application Server](https://github.com/me-box/databox-app-server)
- A [Docker registry](https://docs.docker.com/registry/)

Once the machine has booted, the container manager is available at ``http://IP/``
and the application server at ``http://IP/app-server/`` where IP is the IP address
of the Vagrant machine (see Installation section below).

## Installation

Copy ``vagrant-config.rb.example`` to ``vagrant-config.rb`` and edit in a text
editor.
The configuration values available are:
- *IP*: Sets the static IP for the Vagrant box.
- *memory*: Sets the memory, in megabytes, for the box. 2048 is the recommended value.
- *databox_application_server | recaptcha | publicKey*: reCaptcha public key.
- *databox_application_server | recaptcha | privateKey*: reCaptcha private key.

Then just run ``vagrant up``.

## Meta

Distributed under the MIT license. See ``LICENSE`` for more information.

[https://github.com/me-box/databox-vagrant](https://github.com/me-box/databox-vagrant)