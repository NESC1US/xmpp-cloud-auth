# Installation instructions

## Adding `xcauth` to an existing Nextcloud and an existing XMPP server
This document gives a step-by-step setup if you already have Nextcloud and an XMPP server set up and just want to add *XMPP Cloud Authentication*.

If you are in a hurry, you might use the follow simpler documents instead. If they do not answer all your questions, just come back to this document:
-* [Quick setup for an existing *ejabberd* installation](./QuickInstallEjabberd.md)
-* [Quick setup for an existing *Prosody* installation](./QuickInstallProsody.md)

## Setting things up from the beginning
If, instead, you would like to set up a federated cloud server for storage and communication more or less from scratch, there are more detailed setups in our wiki. We especially recommend:
* [Step-by-step tutorial-style setup of a new server](https://github.com/jsxc/xmpp-cloud-auth/wiki/raspberry-pi-en) (using *ejabberd*; achieves 100% score on [Conversations XMPP Compliance test](https://compliance.conversations.im))
* [Setup of a new server for the experienced admin or separate servers](https://github.com/jsxc/xmpp-cloud-auth/wiki) (the description for *ejabberd* achieves the same 100%)

## Download software
### Option 1, simple: Use the Debian package

Currently, only nightlies are supported, which you get by
```
sudo -s
echo deb https://dl.jsxc.org stable main > /etc/apt/sources.list.d/jsxc.list
wget -qO - https://dl.jsxc.org/archive.key | apt-key add -
apt update
apt install xcauth
```

### Option 2, manual: Use the source, Luke!
Download [the latest release](https://github.com/jsxc/xmpp-cloud-auth/releases)
and put it to your desired location (e.g. `/opt/xmpp-cloud-auth`) or clone this
repository to remain on the leading edge:

```
cd /opt
sudo git clone https://github.com/jsxc/xmpp-cloud-auth
```

Install Python3 and all required libraries. On Ubuntu 18.04, this is:
```
sudo apt install python3 python3-requests python3-configargparse python3-bcrypt python3-bsddb3
sudo apt install socket
```

Create the `xcauth` user and directories, install programs:
```
make install
```

### Developers (in addition to Option 2)
…might want to additionally install
```
sudo apt install python3-nose python3-rednose python3-nose-cov socket
```
They are required to run the tests.

## XMPP Server Configuration

:warning: The API secret must not fall into the wrong hands!
Anyone knowing it can authenticate as any user to the XMPP server
(and create arbitrary new users on the XMPP server).

1. Copy `xcauth.conf` to `/etc` as root and restrict the access rights
   (e.g., `chown xcauth:xcauth /etc/xcauth.conf; chmod 660 /etc/xcauth.conf`)
1. Modify `/etc/xcauth.conf` according to your environment. The values for
   API URL and API SECRET can be found in your Nextcloud/ownCloud JSXC admin page.
1. Adapt your ejabberd/prosody configuration to use this authentication script:

### ejabberd
Adjust your configuration as described in the [admin manual](https://docs.ejabberd.im/admin/configuration/#external-script).
```yaml
nano /etc/ejabberd/ejabberd.yml

auth_method: external
extauth_program: "/usr/bin/socket 127.0.0.1 23662"
auth_use_cache: false
```
and then enable xcauth in [systemd](../doc/Systemd.md)
```
sudo systemctl enable xcejabberd.socket
sudo systemctl start xcauth.service
```
:warning: On Ubuntu, `ejabberd` will come with an **apparmor** profile which will block the external authentication script.
See also the related issue [ejabberd#1598](https://github.com/processone/ejabberd/issues/1598).

:warning: This starts `xcauth.sh`, as some **ejabberd** installations will lead to shared library conflicts,
preventing HTTPS access from within Python. The shell wrapper prevents this conflict.
([ejabberd#1756](https://github.com/processone/ejabberd/issues/1756))
additional information on services, sockets and systemd is [here](../doc/Systemd.md)

### Prosody
Install *lua-lpty* (not necessary when using the [*socket mode*](#socket-interface):
```
apt install lua-lpty
```

Add the following to your config:
```
authentication = "external"
external_auth_command = "/opt/xmpp-cloud-auth/xcauth.py"
```
:warning: The Prosody `mod_auth_external.lua` only accepts a command name, no parameters
([xmpp-cloud-auth#2](https://github.com/jsxc/xmpp-cloud-auth/issues/2), [Prosody#841](https://prosody.im/issues/issue/841)).
All parameters must therefore be set in the configuration file.

:warning: Use the `mod_auth_external.lua` in this repository.
This fixes a bug with treating an echo of the request as the answer
([xmpp-cloud-auth#21](https://github.com/jsxc/xmpp-cloud-auth/issues/21), [Prosody#855](https://prosody.im/issues/issue/855)).

Even then, several users report problems with `lua-lpty`, [such as
processes not dying and still occupying sockets](https://github.com/jsxc/xmpp-cloud-auth/issues/63). Please look at the [socket mode](#socket-interface) for an alternative.

## Configuration as Mail Server Backend

`xcauth` can also be used to provide
- user authentication to mail servers using the *saslauthd* protocol and
- verification of user existence using the *postfix* protocol.

Administrators of small SOHO systems can thus use Nextcloud as their prime
authentication source for
- file storage/sharing (Nextcloud),
- instant messaging (XMPP), and
- email (tested with Cyrus and Postfix).

### *saslauthd* authentication

In an attempt to move toward Nextcloud as the main authentication source,
`-t saslauthd` mode is supported, which allows to run services
which can authenticate against Cyrus *saslauthd* to authenticate against
JSXC and Nextcloud. It has been successfully tested against *Postfix*
and *Cyrus IMAP*. More information can be found in
[doc/Systemd.md (*saslauthd* mode)](../doc/Systemd.md#saslauthd-mode-authentication).
The protocol is described in [doc/Protocol.md](./Protocol.md#saslauthd).

### *postfix* existence tests

When using virtual mailboxes (i.e., mailboxes in multiple domains, nut just
using virtual addresses), *Postfix* needs a way to check for the existence
of that mailbox. A *Postfix* `tcp_table` compatible interface has been
implemented using the `-t postfix` mode, so an `xcauth` instance
started e.g. by *systemd* can be used to provide the mailbox existence
information, as explained in
[doc/Systemd.md (*postfix* mode)](../doc/Systemd.md#postfix-mode-existence-check).
Please note, that aliases or virtual users still need to be configured
using the standard *postfix* mechanisms.

The protocol is described in [doc/Protocol.md](./Protocol.md#postfix).

## Options
```
$ ./xcauth.py --help
usage: xcauth.py [-h] [--config-file CONFIG_FILE] [--db DB]
                 [--cache-storage {none,memory,db}] [--domain-db DOMAIN_DB]
                 [--auth-test USER DOMAIN PASSWORD]
                 [--isuser-test USER DOMAIN] [--roster-test USER DOMAIN]
                 [--update-roster] --url URL --secret SECRET [--log LOG]
                 [--debug] [--interactive]
                 [--type {generic,prosody,ejabberd,saslauthd,postfix}]
                 [--timeout TIMEOUT] [--cache-db CACHE_DB]
                 [--cache-query-ttl CACHE_QUERY_TTL]
                 [--cache-verification-ttl CACHE_VERIFICATION_TTL]
                 [--cache-unreachable-ttl CACHE_UNREACHABLE_TTL]
                 [--cache-bcrypt-rounds CACHE_BCRYPT_ROUNDS]
                 [--ejabberdctl PATH] [--shared-roster-db SHARED_ROSTER_DB]
                 [--version]

XMPP server authentication against JSXC>=3.2.0 on Nextcloud. See
https://jsxc.org or https://github.com/jsxc/xmpp-cloud-auth. Args that start
with '--' (eg. --db) can also be set in a config file (/etc/xcauth.conf or
specified via --config-file). Config file syntax allows: key=value, flag=true,
stuff=[a,b,c] (for details, see syntax at https://goo.gl/R74nmi). If an arg is
specified in more than one place, then commandline values override config file
values which override defaults.

optional arguments:
  -h, --help            show this help message and exit
  --config-file CONFIG_FILE, -c CONFIG_FILE
                        config file path
  --db DB               Path to the SQLite state database
  --cache-storage {none,memory,db}
                        How to cache authentication information
  --domain-db DOMAIN_DB, -b DOMAIN_DB
                        persistent domain database; manipulated with xcdbm.py.
                        DEPRECATED, will only be used for migration purposes.
  --auth-test USER DOMAIN PASSWORD, -A USER DOMAIN PASSWORD
                        single, one-shot query of the user, domain, and
                        password triple
  --isuser-test USER DOMAIN, -I USER DOMAIN
                        single, one-shot query of the user and domain tuple
  --roster-test USER DOMAIN, -R USER DOMAIN
                        single, one-shot query of the user's shared roster
  --update-roster, -T   DEPRECATED. Automatically activated when --ejabberdctl
                        is set
  --url URL, -u URL     base URL
  --secret SECRET, -s SECRET
                        secure api token
  --log LOG, -l LOG     log directory (default: /var/log/xcauth)
  --debug, -d           enable debug mode
  --interactive, -i     log to stderr
  --type {generic,prosody,ejabberd,saslauthd,postfix}, -t {generic,prosody,ejabberd,saslauthd,postfix}
                        XMPP server/query protocol type (prosody≘generic);
                        implies reading requests from stdin. See
                        doc/Installation.md and doc/Systemd.md for more
                        information and overrides.
  --timeout TIMEOUT     Timeout for connection setup, request processing
  --cache-db CACHE_DB   Database path for the user cache. DEPRECATED, only for
                        conversion purposes
  --cache-query-ttl CACHE_QUERY_TTL
                        Maximum time between queries
  --cache-verification-ttl CACHE_VERIFICATION_TTL
                        Maximum time between backend verifications
  --cache-unreachable-ttl CACHE_UNREACHABLE_TTL
                        Maximum cache time when backend is unreachable
                        (overrides the other TTLs)
  --cache-bcrypt-rounds CACHE_BCRYPT_ROUNDS
                        Encrypt passwords with 2^ROUNDS before storing (i.e.,
                        every increment of ROUNDS results in twice the
                        computation time, both for us and an attacker). First
                        value is for persistent, second value for in-memory
                        cache, if both values are present.
  --ejabberdctl PATH    Enables shared roster updates on authentication; use
                        ejabberdctl command at PATH to modify them
  --shared-roster-db SHARED_ROSTER_DB
                        Which groups a user has been added to (to ensure
                        proper deletion). DEPRECATED, only for conversion
                        purposes.
  --version             show program's version number and exit

-I, -R, and -A take precedence over -t. One of them is required. -I, -R, and
-A imply -i and -d. Signals: SIGHUP reopens the error log, SIGUSR1 dumps the
thread list to the error log.
```

Note that `-t generic` is identical to `-t prosody`. This is just to indicate
that new applications should pick the line-based protocol instead of the `ejabberd`
length-prefixed protocol. (*Prosody* `mod_auth_external.lua` calls the protocol
`generic` as well.)

If only a single (API secret, API url) tuple is defined (the one in the configuration file or on the command line), then this one will be used for all requests.
If additional per-domain-configuration entries are given (via the `-p` option), then if the domain equals one in this per-domain configuration, the parameters
there will take precedence over the global, fallback tuple. You generally will only need this if you operate a single XMPP server providing service
to multiple cloud instances.

For information about the caching system, see [Cache.md](Cache.md)

## Commands
When using `xmpp-cloud-auth.py` in `-t` mode (reading commands from stdin), the following commands are recognized:

* `auth:<USER>:<DOMAIN>:<PASSWORD>`: Is this the PASSWORD for the given USER (in the given DOMAIN)?
* `isuser:<USER>:<DOMAIN>`: Does this USER exist (in the given DOMAIN)?
* `roster:<USER>:<DOMAIN>`: Return the shared roster information. Nonstandard, only useful with `-t generic`
* `quit` and `exit`: Terminate (for interactive commands, especially over a socket connection; nonstandard)
* EOF: Terminate


## Troubleshooting
In case you need some additional debugging, you can try and run `xcauth.py` from the command line with the usual options and then add `-A jane.doe example.com p4ssw0rd` to test the connection to the ownCloud/Nextcloud server.

If Conversations cannot connect and complains about "Downgrade attack", see the following issue:
[No (obvious?) way to accept SASL downgrade (Conversations#2498)](https://github.com/siacs/Conversations/issues/2498).
Current workaround: Delete the account in Conversations and then add it again.

### Socket interface

If you see unreliable behavior with *Prosody*, you might want to try the experimental socket interface.
When using the `mod_auth_external.lua` bundled here (together with `pseudolpty.lua`), you can use
the `external_auth_command = "@localhost:23663";` option to talk over a socket to a process not spawned
by *Prosody* on port 23663. [doc/Systemd.md](../doc/Systemd.md) explains how to automatically start
such a process using *systemd*.

### *ejabberd* shared roster support

In the configuration file, set
1. the `ejabberdctl` configuration variable to the path to the `ejabberdctl` binary, and
1. `shared-roster-db` to a path where *xmpp-cloud-auth* can create its shared roster database (e.g. `/var/lib/xcauth/shared-roster.db`).
Then, on every future login, *xmpp-cloud-auth* will query JSXC for the list of Nextcloud groups the user is in, and create a shared
roster group for each of those groups. The grouping will not be visible in JSXC, as the
[JSXC user interface does not yet display the groupings (pull requests welcome!)](https://github.com/jsxc/jsxc/issues/77); JSXC users will just see the members without the group structure.
XMPP clients supporting roster groups will also display the groups accordingly.

Shared rosters provide implicit mutual access to the presence information within the roster groups.

For *Prosody*, there is a [module available to manage shared rosters](https://github.com/jsxc/prosody-cloud-roster).

## How does it work?
Your XMPP server sends the authentication data in a [special format](https://www.ejabberd.im/files/doc/dev.html#htoc9) on the standard input to the authentication script, length-prefixed (`-t ejabberd`) for *ejabberd*, newline-terminated (`-t prosody` aka `-t generic`) for *Prosody* (and maybe others). The script will first try to verify the given password as time-limited token and if this fails, it will send a HTTP request to your cloud installation to verify this data. To protect your Nextcloud/Owncloud against different attacks, every request has a signature similar to the  [github webhook signature]( https://developer.github.com/webhooks/securing/).

More information can be found in [Protocol.md](Protocol.md).
