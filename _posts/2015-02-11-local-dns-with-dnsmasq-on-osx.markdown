---
layout: post
title: "Local DNS with dnsmasq on OS X"
---

It's useful to be able to test web applications during development with hostnames that are more realistic than `localhost`, such as `subdomain.my-web-app.dev`. Using the `dnsmasq` package from [MacPorts](https://trac.macports.org/browser/trunk/dports/net/dnsmasq/Portfile) and OS X's support for manual nameserver assignment for specific domains, setting up resolution for local testing domains is pretty straightforward.

***"I'm in a hurry!":*** use the script from [this Gist](https://gist.github.com/oko/c05420c39efa3204d2b7) to set up resolution for `.dev` and `.test` domain names that point to `127.0.0.1`.

## Prerequisites

* Installed version of [MacPorts](https://www.macports.org/install.php)
* Installed Mac OS X command line tools (`make`, `gcc`, etc.). If not installed, the following will install them on Yosemite:

    ```
    $ xcode-select --install
    ```

## Installation and Configuration

1.  Install and load the `dnsmasq` port from MacPorts:

        $ sudo port -v install dnsmasq
        $ sudo port -v load dnsmasq

    This will install `dnsmasq` and create a `launchd` service file in `/Library/LaunchDaemons/`. As soon as the service file is created `dnsmasq` will be loaded by `launchd`.

2.  Create the `/etc/resolver` directory for domain-specific nameserver settings:

        $ sudo mkdir /etc/resolver

    The `/etc/resolver` directory contains files in `resolv.conf` format that specify name resolution settings for specific domain names. For example, a file `/etc/resolver/exampledomain` containing the line `nameserver 10.10.10.10` would resolve queries to `*.exampledomain` using the DNS server at `10.10.10.10`, while resolving queries for all other domains via the system-configured nameservers. 

3.  Create a resolver file for each domain you'd like to resolve locally:

        $ echo -n "nameserver 127.0.0.1" | sudo tee /etc/resolver/dev

    Replace `dev` with any other custom TLD you'd like to resolve. Don't specify a real TLD here unless you never plan on visiting a domain from that TLD.

4.  Create a `dnsmasq` rule for each domain you added in step 3:

        $ echo "address=/dev/127.0.0.1" | sudo tee -a /opt/local/etc/dnsmasq.conf

    Repeat the above for each domain you added in step 3.

5.  Reload `dnsmasq`:

        $ sudo kill -9 $(pgrep dnsmasq)

    This will kill the current `dnsmasq` instance, which will be immediately relaunched by `launchd` with the new configuration.

6.  Verify the configuration:

        $ dig + short @localhost domain.dev

    This should return `127.0.0.1` on standard output. Verify resolution using OS X's system configuration utility:

        $ scutil --dns

    This should return an entry similar to the following for each custom domain:

        resolver #2
          domain   : dev
          nameserver[0] : 127.0.0.1
          flags    : Request A records, Request AAAA records
          reach    : Reachable,Local Address

You should now be able to start up a local development server and browse to it at `http://localhost.dev:PORT`. Run `python -m SimpleHTTPServer 8080` in your home directory and you should be able to see its contents at [`localhost.dev:8080`](http://localhost.dev:8080)
