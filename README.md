
# PKI

**Warning: use at your own risk.** I am neither a PKI nor OpenSSL
expert. I've never run a proper CA. I'm just a guy on the internet
with a script that `curl`s your private keys up to my server. (Not
really, but if you were going to use this for anything important you
were planning on auditing everything so you can be certain, right?
This _is_ root-of-trust we're talking about, here.)

With that said, the `pki` script in this repo makes it easy and fun to
create and manage your own personal PKI infrastructure. If you have
Yubikeys and the right software, you can even store your signing keys
securely.

The basic idea and some of the implementation details come from
https://github.com/ryankurte/pki. The `pki` script creates two root
authorities, A and B, whose certificates are signed by each other. The
root keys can be stored on Yubikey devices. One of the roots is used
to sign an intermediate CA certificate (whose key can also be stored
on a Yubikey), which is presumably issued to an individual and is used
to sign "leaf" certificates. The roots are able to revoke
certificates, though this ability is not (yet?) built in to the `pki`
script.

The `pki` script has a subcommand-style UI. Run `pki -h` for
documentation:

```
usage: pki [-h] [-c FILE] [-d DIR] [--dry] [--pkcs11-so PATH]
           [--pkcs11-module PATH]
           {root,intermediate,leaf,ykload,ykpin} ...

A program for managing personal PKI infrastructure.

optional arguments:
  -h, --help            show this help message and exit
  -c FILE, --config FILE
                        path to the site configuration file (default:
                        /path/to/clone/site.conf)
  -d DIR, --directory DIR
                        directory where everything lives (default:
                        /path/to/clone/files)
  --dry                 dry run – do not actually run commands, change
                        files, etc
  --pkcs11-so PATH      specify the path to the pkcs11 openssl engine
                        (default: from config)
  --pkcs11-module PATH  specify the path to the pkcs11 plugin (default: from
                        config)

subcommands:
  Convenience commands for PKI infrastructure.

  {root,intermediate,leaf,ykload,ykpin}
    root                Generate the cross-signed PKI roots.
    intermediate        Generate the intermediate CA for day-to-day signing.
    leaf                Generate a certificate signed by an intermediate CA.
    ykload              Load a cert/key pair on to a Yubikey.
    ykpin               Change the Yubikey signing PIN.
```

You can run `pki <subcommand> -h` to get help for a particular
subcommand.

`pki` aims to be flexible when it comes to configuration. Most things
can be configured using command-line arguments _or_ settings in an
ini-style config file. When both are specified, values from
command-line arguments override settings from config files.

The default config file is in this repo, at `defaults.conf`. The
"preferred" way to configure `pki` is to copy this file elsewhere,
then use the `-c/--config` to point `pki` to your site-specific
configuration. (It's also super useful to change the value of
`directory` inside your copied config file – see _Installation_
below.)

Note that you can override `country`, `state-or-province`, `locality`,
`organization`, and `organizational-unit` within the `[root]`,
`[intermediate]`, and `[leaf]` sections. If overridden, the specified
values will apply when running the respective subcommands. (Just to be
clear: for example, when determining the country code for an
intermediate CSR, `pki` looks at `--country`, then at
`conf.intermediate.country`, then at `conf.pki.country`. If you don't
specify any of those, it ends up using the value from
`conf.pki.country` in `defaults.conf` – i.e. `US`.)


## Installation

Here's one way to set things up:

```
cd
mkdir vendor
cd vendor
git clone https://github.com/decafjoe/pki
cd
mkdir pki
cp vendor/pki/defaults.conf pki/site.conf
# This next line could go in a .bashrc (or equivalent)....
alias pki="$HOME/vendor/pki -c$HOME/pki/site.conf"
# Edit ~/pki/site.conf as appropriate. Add the following to the [pki] section:
# directory = ~/pki
```


## Examples

Note that these are untested. As with everything else in this repo,
use at your own risk.


### No Yubikey

One-time setup:

```
pki root
pki intermediate jdoe 'John Doe'
```

Generating certificates:

```
pki leaf --sign-with jdoe test example.com
# Subject alternate names are supported
pki leaf --sign-with jdoe test example.com example.org random.wut
```


### With Yubikeys

One-time setup:

```
pki root
# Insert first yubikey
pki ykpin
# Default pin is 123456, set it to something, er, better...
pki ykload --root b
# Remove first yubikey, insert second yubikey
pki ykpin
pki ykload --root a
# Note that --yubikey below means "sign with root cert on yubikey", _not_
# "the new intermediate CA will be a yubikey based certificate"
pki intermediate --yubikey jdoe 'John Doe'
# Remove the second yubikey, insert John's yubikey
pki ykpin
pki ykload --intermediate jdoe
shred -n35 ~/pki/*.key
```

Generating certificates:

```
# Note that --yubikey below means "sign with intermediate cert on yubikey"
# and does _not_ imply the new certificate will at all be associated with
# the yubikey.
pki leaf --yubikey example example.com
# Subject alternate names are supported
pki leaf --yubikey example example.com example.org random.wut
```
