---
title: "Secure GnuPG Setup with YubiKey Integration"
date: 2023-06-01 12:00:00 -0800
description: Using good GnuPG security and setup practices to create a hardened YubiKey security key
categories: [Security, Cryptography]
tags: [homelab, security, crypto, pki]
img_path: /assets/lib/2023-06-01-gpg-yubikey/
image:
  path: yubikey.jpeg
  alt: Yubikey 5C
---




## Overview

I wanted to find a way to provide the best level of 2FA possible for some of my more sensitive data and accounts. While services like SMS-based and token-based TOTP are valid in providing an extra layer of security, there is always a chance that that these channels could be compromised.

This method of using GnuPG to create and store private keys on a YubiKey is the most secure method to providing meaningful and reasonable authentication to sensitive services. Asymmetric-key cryptography is also a brilliant way to provide data encryption, integrity, and non-repudiation. For example, you could use your YubiKey for S/MIME in email or to sign git commits.

I also want to emphasize that having a security key doesn't mean you are safe from all attacks. It is important to still exercise good security practices, and to have revocation certificates on hand, in case you believe your security key has been compromised / stolen.




## Setup



### MacOS Setup

- recommend setup on air-gapped live environment, but MacOS homebrew for simplicity

Packages + dependencies to install:
```terminal
$ brew install gnupg yubikey-personalization hopenpgp-tools sqlite ykman pinentry-mac wget swig
```

Helps to set up the `.gnupg` directory and settings:
```terminal
$ gpg --card-status
gpg: directory '/Users/johnsmith/.gnupg' created
Reader ...........: Yubico YubiKey OTP FIDO CCID
[...]
```

Optionally, consider setting up the temp dir first and activating a venv before installing:
```terminal
$ pip install yubikey-manager
```


### Entropy

- quality of generated randomness, measured as entropy
- most operating systems use either PRNG (psuedo-random number generators) or HRNG (hardware random number generators)


#### Yubikey - Entropy

- YubiKey can gather additional entropy via the SmartCard interface

To seed the kernel's PRNG with additional 512 bytes retrieved from the YubiKey:
```terminal
$ echo "SCD RANDOM 512" | gpg-connect-agent | sudo tee /dev/random | hexdump -C
```


### Setting up Configuration


#### Temp Directory

> **Optional but Recommended**
>
> Create a temporary directory to be cleared at reboot:
> ```terminal
> $ export GNUPGHOME=$(mktemp -d -t gnupg_$(date +%Y%m%d%H%M)_XXX)
> ```
{: .prompt-info }


#### Harden Configuration

Create a hardened configuration in the temporary working directory:
```terminal
$ wget -O $GNUPGHOME/gpg.conf https://raw.githubusercontent.com/drduh/config/master/gpg.conf
```

This command should output this:
```terminal
$ grep -ve "^#" $GNUPGHOME/gpg.conf
personal-cipher-preferences AES256 AES192 AES
personal-digest-preferences SHA512 SHA384 SHA256
personal-compress-preferences ZLIB BZIP2 ZIP Uncompressed
default-preference-list SHA512 SHA384 SHA256 AES256 AES192 AES ZLIB BZIP2 ZIP Uncompressed
cert-digest-algo SHA512
s2k-digest-algo SHA512
s2k-cipher-algo AES256
charset utf-8
fixed-list-mode
no-comments
no-emit-version
keyid-format 0xlong
list-options show-uid-validity
verify-options show-uid-validity
with-fingerprint
require-cross-certification
no-symkey-cache
use-agent
throw-keyids
```

> **Air Gap**
>
> to harden security further it is **IMPORTANT** to **DISABLE** networking for the remainder of the setup
{: .prompt-danger }



### Master Key

- the master key will be used for certification only: to issue **sub-keys** that are used for *encryption, signing and authentication*.
- should be kept offline at ALL TIMES

Generate a strong passkey to be kept permanently (will be needed multiple times later):
```terminal
$ gpg --gen-random --armor 0 24
47YN/xx42/I4WRtDZkUimN7+Q3T6A6Qe
```

#### Generate

Generate a new key pair:
```terminal
$ gpg --expert --full-generate-key

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
   (9) ECC and ECC
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (13) Existing key
Your selection? 8

Possible actions for a RSA key: Sign Certify Encrypt Authenticate
Current allowed actions: Sign Certify Encrypt

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? E

Possible actions for a RSA key: Sign Certify Encrypt Authenticate
Current allowed actions: Sign Certify

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? S

Possible actions for a RSA key: Sign Certify Encrypt Authenticate
Current allowed actions: Certify

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? Q
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 0
Key does not expire at all
Is this correct? (y/N) y



GnuPG needs to construct a user ID to identify your key.

Real name: John Smith
Email address: john@smith.test
Comment: [Optional - leave blank]
You selected this USER-ID:
    "John Smith <john@smith.test>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o


        \\\ Enter in Master Key Passkey \\\


We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

gpg: /tmp.FLZC0xcM/trustdb.gpg: trustdb created
gpg: key 0xFF3E7D88647EBCDB marked as ultimately trusted
gpg: directory '/tmp.FLZC0xcM/openpgp-revocs.d' created
gpg: revocation certificate stored as '/tmp.FLZC0xcM/openpgp-revocs.d/011CE16BD45B27A55BA8776DFF3E7D88647EBCDB.rev'
public and secret key created and signed.

pub   rsa4096/0xFF3E7D88647EBCDB 2017-10-09 [C]
      Key fingerprint = 011C E16B D45B 27A5 5BA8  776D FF3E 7D88 647E BCDB
uid                              John Smith <john@smith.test>
```

#### Export Key ID

Export the key ID as an environment variable:
```terminal
$ export KEYID=0xFF3E7D88647EBCDB <change this to reflect your keyid>
```


### Sub-Keys

Edit the master key to add sub-keys

This will put you in "gpg edit mode":
```terminal
$ gpg --expert --edit-key $KEYID

Secret key is available.

sec  rsa4096/0xEA5DE91459B80592
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
[ultimate] (1). John Smith <john@smith.test>
```

#### Signing

Create a signing key by selecting `addkey` then `(4) RSA (sign only)`:
```terminal
gpg> addkey
Key is protected.

You need a passphrase to unlock the secret key for
user: "John Smith <john@smith.test>"
4096-bit RSA key, ID 0xFF3E7D88647EBCDB, created 2016-05-24

Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
Your selection? 4
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at Mon 10 Sep 2018 00:00:00 PM UTC
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: 2018-10-09       usage: S
[ultimate] (1). John Smith <john@smith.test>
```

#### Encrytion

Next, create an encryption key by selecting `(6) RSA (encrypt only)`:
```terminal
gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 6
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at Mon 10 Sep 2018 00:00:00 PM UTC
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: 2018-10-09       usage: S
ssb  rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: 2018-10-09       usage: E
[ultimate] (1). John Smith <john@smith.test>
```

#### Authentication

Finally, create an authentication key:

- GPG doesn't provide an authenticate-only key type, so select `(8) RSA (set your own capabilities)` and toggle the required capabilities until the only allowed action is `Authenticate`

```terminal
gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 8

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: --(Sign)-- --(Encrypt)--

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? S

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: --(Encrypt)--

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? E

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: --( )--

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? A

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: --(Authenticate)--

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? Q
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at Mon 10 Sep 2018 00:00:00 PM UTC
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: 2018-10-09       usage: S
ssb  rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: 2018-10-09       usage: E
ssb  rsa4096/0x3F29127E79649A3D
    created: 2017-10-09  expires: 2018-10-09       usage: A
[ultimate] (1). John Smith <john@smith.test>
```

Finish by saving the keys:
```terminal
gpg> save
```


### Verify

List the generated secret keys and verify the output:
```terminal
$ gpg -K
/tmp.FLZC0xcM/pubring.kbx
-------------------------------------------------------------------------
sec   rsa4096/0xFF3E7D88647EBCDB 2017-10-09 [C]
      Key fingerprint = 011C E16B D45B 27A5 5BA8  776D FF3E 7D88 647E BCDB
uid                            John Smith <john@smith.test>
ssb   rsa4096/0xBECFA3C1AE191D15 2017-10-09 [S] [expires: 2018-10-09]
ssb   rsa4096/0x5912A795E90DD2CF 2017-10-09 [E] [expires: 2018-10-09]
ssb   rsa4096/0x3F29127E79649A3D 2017-10-09 [AR] [expires: 2018-10-09]
```

- Add any additional identities or email addresses you wish to associate using the `adduid` command.

Verify with a OpenPGP key [best practice checker](https://riseup.net/en/security/message-security/openpgp/best-practices#openpgp-key-checks):
```terminal
$ gpg --export $KEYID | hokey lint
```

- The output will display any problems with your key in red text. If everything is green, your key passes each of the tests. If it is red, your key has failed one of the tests.

> Hokey may warn (orange text) about cross certification for the authentication key. GPG's [Signing Subkey Cross-Certification](https://gnupg.org/faq/subkey-cross-certify.html) documentation has more detail on cross certification, and gpg v2.2.1 notes "subkey does not sign and so does not need to be cross-certified". hokey may also indicate a problem (red text) with `Key expiration times: []` on the primary key.
{: .prompt-warning }



### Export Keys

Save a copy of your keys:
```terminal
$ gpg --armor --export-secret-keys $KEYID > $GNUPGHOME/mastersub.key

$ gpg --armor --export-secret-subkeys $KEYID > $GNUPGHOME/sub.key
```


### Revocation Certificate

- Although we will backup and store the master key in a safe place, it is best practice to never rule out the possibility of losing it or having the backup fail. Without the master key, it will be impossible to renew or rotate subkeys or generate a revocation certificate, the PGP identity will be useless.
- Even worse, we cannot advertise this fact in any way to those that are using our keys. It is reasonable to assume this *will* occur at some point and the only remaining way to deprecate orphaned keys is a revocation certificate.

To create the revocation certificate:
```terminal
$ gpg --output $GNUPGHOME/revoke.asc --gen-revoke $KEYID
```
> The `revoke.asc` certificate file should be stored (or printed) in a (secondary) place that allows retrieval in case the main backup fails.



### Backup

- I used a USB stick that I partitioned with APFS (Encrypted)
- called it KEY-BACKUP

1. `cp` the entire temp directory to a new directory `~/encryption`{: .filepath}
2. `rm` the venv/ directory
3. `cp or mv` to APFS volume (minus venv/)
4. `rm -rf` the fuck outta dat `~/encryption`{: .filepath} directory



### Public Key

Created a new volume on the usb stick APFS (not encrypted): named *Public-Key*

Exported public key file to that Public-Key volume:
```terminal
$ gpg --armor --export $KEYID | sudo tee /Volumes/Public-Key/gpg-$KEYID-$(date +%F).asc
```
> optionally you could upload to a pulic key server like MIT
{: .prompt-info }


### Configure Smartcard

Plug in a YubiKey and use GPG to configure it as a smartcard:
```terminal
$ gpg --card-edit

Reader ...........: Yubico Yubikey 4 OTP U2F CCID
Application ID ...: D2760001240102010006055532110000
Application type .: OpenPGP
Version ..........: 3.4
Manufacturer .....: Yubico
Serial number ....: 05553211
Name of cardholder: [not set]
Language prefs ...: [not set]
Salutation .......:
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: not forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 0 3
Signature counter : 0
KDF setting ......: off
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]
```

Enter administrative mode:
```terminal
gpg/card> admin
Admin commands are allowed
```

#### Enable KDF

Key Derived Function (KDF) enables YubiKey to store the hash of PIN, preventing the PIN from being passed as plain text. Note that this requires a relatively new version of GnuPG to work, and may not be compatible with other GPG clients (notably mobile clients). These incompatible clients will be unable to use the YubiKey GPG functions as the PIN will always be rejected.
```terminal
gpg/card> kdf-setup
```

> default admin pin: 12345678
{: .prompt-tip }


#### Change PIN

to update the GPG Pins on YubiKey:
```terminal
gpg/card> passwd
gpg: OpenPGP card no. D2760001240102010006055532110000 detected

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 3
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 1
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 4
Rest Code set.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? q
```

- set the Admin PIN, PIN, and Rest Code (located in an important folder)


#### Set Information

Some fields are optional:
```terminal
gpg/card> name
Cardholder's surname: Smith
Cardholder's given name: John

gpg/card> lang
Language preferences: en

gpg/card> login
Login data (account name): john@smith.test

gpg/card> list

Reader ...........: Yubico YubiKey OTP FIDO CCID
Application ID ...: D2760001240100000006234833810000
Application type .: OpenPGP
Version ..........: 3.4
Manufacturer .....: Yubico
Serial number ....: 23483381
Name of cardholder: John Smith
Language prefs ...: en
Salutation .......:
URL of public key : [not set]
Login data .......: john@smith.test
Signature PIN ....: not forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
KDF setting ......: on
UIF setting ......: Sign=off Decrypt=off Auth=off
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]

gpg/card> quit
```

> Optionally you can reset the number of retry attempts for the pin:
> ```terminal
> $ ykman openpgp access set-retries 5 5 5 -f -a YOUR_ADMIN_PIN
> ```
{: .prompt-tip }



### Transfer Keys

> Transferring keys to YubiKey using `keytocard` is a destructive, one-way operation only. Make sure you've made a backup before proceeding: `keytocard` converts the local, on-disk key into a stub, which means the on-disk copy is no longer usable to transfer to subsequent security key devices or mint additional keys.
{: .prompt-warning }

- Previous GPG versions required the `toggle` command before selecting keys. The currently selected key(s) are indicated with an `*`. When moving keys **only ONE KEY should be selected at a time**.

Enter into GPG Key Edit:

- the selected key will have an `*`

```terminal
$ gpg --edit-key $KEYID

Secret key is available.

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: 2018-10-09  usage: S
ssb  rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: 2018-10-09  usage: E
ssb  rsa4096/0x3F29127E79649A3D
    created: 2017-10-09  expires: 2018-10-09  usage: A
[ultimate] (1). John Smith <john@smith.test>
```

#### Signing

Select and transfer the signature key:

- you will be prompted for the Master Key Passphrase and Admin PIN

```terminal
gpg> key 1

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb* rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: 2018-10-09  usage: S
ssb  rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: 2018-10-09  usage: E
ssb  rsa4096/0x3F29127E79649A3D
    created: 2017-10-09  expires: 2018-10-09  usage: A
[ultimate] (1). John Smith <john@smith.test>

gpg> keytocard
Please select where to store the key:
   (1) Signature key
   (3) Authentication key
Your selection? 1

You need a passphrase to unlock the secret key for
user: "John Smith <john@smith.test>"
4096-bit RSA key, ID 0xBECFA3C1AE191D15, created 2016-05-24
```

#### Encryption

Type `key 1` again to de-select and `key 2` to select the next key:
```terminal
gpg> key 1

gpg> key 2

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: 2018-10-09  usage: S
ssb* rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: 2018-10-09  usage: E
ssb  rsa4096/0x3F29127E79649A3D
    created: 2017-10-09  expires: 2018-10-09  usage: A
[ultimate] (1). John Smith <john@smith.test>

gpg> keytocard
Please select where to store the key:
   (2) Encryption key
Your selection? 2

[...]
```

#### Authentication

Type `key 2` again to deselect and `key 3` to select the last key:
```terminal
gpg> key 2

gpg> key 3

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: 2018-10-09  usage: S
ssb  rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: 2018-10-09  usage: E
ssb* rsa4096/0x3F29127E79649A3D
    created: 2017-10-09  expires: 2018-10-09  usage: A
[ultimate] (1). John Smith <john@smith.test>

gpg> keytocard
Please select where to store the key:
   (3) Authentication key
Your selection? 3
```

Save and Quit:
```terminal
gpg> save
```


### Verify Card

Verify the sub-keys have been moved to YubiKey as indicated by `ssb>`:
```terminal
$ gpg -K
/tmp.FLZC0xcM/pubring.kbx
-------------------------------------------------------------------------
sec   rsa4096/0xFF3E7D88647EBCDB 2017-10-09 [C]
      Key fingerprint = 011C E16B D45B 27A5 5BA8  776D FF3E 7D88 647E BCDB
uid                            John Smith <john@smith.test>
ssb>  rsa4096/0xBECFA3C1AE191D15 2017-10-09 [S] [expires: 2018-10-09]
ssb>  rsa4096/0x5912A795E90DD2CF 2017-10-09 [E] [expires: 2018-10-09]
ssb>  rsa4096/0x3F29127E79649A3D 2017-10-09 [A] [expires: 2018-10-09]
```


### Multiple Hardware Keys

For instructions on configuring multiple hardware keys go to:

[Dr Duh => YubiKey => Multiple YubiKeys](https://github.com/drduh/YubiKey-Guide?tab=readme-ov-file#using-multiple-yubikeys)



### Cleanup

Before finishing the setup, ensure you have done the following:

-   Saved encryption, signing and authentication sub-keys to YubiKey (`gpg -K` should show `ssb>` for sub-keys).
-   Saved the YubiKey user and admin PINs which are different and were changed from default values.
-   Saved the password to the GPG master key in a secure, long-term location.
-   Saved a copy of the master key, sub-keys and revocation certificate on an encrypted volume, to be stored offline.
-   Saved the password to that encrypted volume in a secure, long-term location (separate from the device itself).
-   Saved a copy of the public key somewhere easily accessible later.

Now reboot or delete `$GNUPGHOME` and remove the secret keys from the GPG keyring:
```terminal
$ gpg --delete-secret-key $KEYID

$ sudo rm -rf $GNUPGHOME

$ unset GNUPGHOME
```

May need to uninstall and remove gpg system-wide directories:

- saw that the public and private keys as well as the key ring was installed on the main home directory

```terminal
$ brew uninstall gpg

$ rm -rf /usr/local/etc/gnupg /usr/local/etc/gnupg/scdaemon.conf

$ rm -rf /home/$USER/.gnupg

$ brew install gpg

$ gpg --card-status
```



## Usage



### Configure GPG

- this section will go over how to set up your public keys' configuration

Download [drduh/config/gpg.conf](https://github.com/drduh/config/blob/master/gpg.conf) (hardened gpg config):
```terminal
$ cd ~/.gnupg ; wget https://raw.githubusercontent.com/drduh/config/master/gpg.conf

$ chmod 600 gpg.conf
```

Import the public key file:
```terminal
$ gpg --import /Volumes/Public-Key/gpg-0x*.asc
gpg: /Users/johnsmith/.gnupg/trustdb.gpg: trustdb created
gpg: key 0x0AD6219B485B3FD1: public key "John Smith <john@smith.test>" imported
gpg: Total number processed: 1
gpg:               imported: 1
```

Or download from a public key server:
```terminal
$ gpg --recv $KEYID
```

Edit the master key to assign it ultimate trust by selecting `trust` and `5`:
```terminal
$ export KEYID=0x0AD6219B485B3FD1

$ gpg --edit-key $KEYID

gpg> trust
pub  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never       usage: C
                               trust: unknown       validity: unknown
sub  4096R/0xBECFA3C1AE191D15  created: 2017-10-09  expires: 2018-10-09  usage: S
sub  4096R/0x5912A795E90DD2CF  created: 2017-10-09  expires: 2018-10-09  usage: E
sub  4096R/0x3F29127E79649A3D  created: 2017-10-09  expires: 2018-10-09  usage: A
[ unknown] (1). John Smith <john@smith.test>

Please decide how far you trust this user to correctly verify other users' keys
(by looking at passports, checking fingerprints from different sources, etc.)

  1 = I don't know or won't say
  2 = I do NOT trust
  3 = I trust marginally
  4 = I trust fully
  5 = I trust ultimately
  m = back to the main menu

Your decision? 5
Do you really want to set this key to ultimate trust? (y/N) y

pub  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never       usage: C
                               trust: ultimate      validity: unknown
sub  4096R/0xBECFA3C1AE191D15  created: 2017-10-09  expires: 2018-10-09  usage: S
sub  4096R/0x5912A795E90DD2CF  created: 2017-10-09  expires: 2018-10-09  usage: E
sub  4096R/0x3F29127E79649A3D  created: 2017-10-09  expires: 2018-10-09  usage: A
[ unknown] (1). John Smith <john@smith.test>

gpg> quit
```

Remove and re-insert YubiKey and verify the status:
```terminal
$ gpg --card-status
Reader ...........: Yubico YubiKey OTP FIDO CCID 00 00
Application ID ...: D2760001240102010006055532110000
Version ..........: 3.4
Manufacturer .....: Yubico
Serial number ....: 05553211
Name of cardholder: John Smith
Language prefs ...: en
Sex ..............: unspecified
URL of public key : [not set]
Login data .......: john@smith.test
Signature PIN ....: not forced
Key attributes ...: rsa4096 rsa4096 rsa4096
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
KDF setting ......: on
Signature key ....: 07AA 7735 E502 C5EB E09E  B8B0 BECF A3C1 AE19 1D15
      created ....: 2016-05-24 23:22:01
Encryption key....: 6F26 6F46 845B BEB8 BDF3  7E9B 5912 A795 E90D D2CF
      created ....: 2016-05-24 23:29:03
Authentication key: 82BE 7837 6A3F 2E7B E556  5E35 3F29 127E 7964 9A3D
      created ....: 2016-05-24 23:36:40
General key info..: pub  4096R/0xBECFA3C1AE191D15 2016-05-24 John Smith <john@smith.test>
sec#  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never
ssb>  4096R/0xBECFA3C1AE191D15  created: 2017-10-09  expires: 2018-10-09
                      card-no: 0006 05553211
ssb>  4096R/0x5912A795E90DD2CF  created: 2017-10-09  expires: 2018-10-09
                      card-no: 0006 05553211
ssb>  4096R/0x3F29127E79649A3D  created: 2017-10-09  expires: 2018-10-09
                      card-no: 0006 05553211
```

- `sec#` indicates the master key is not available (as it should be stored encrypted offline).

> If you see `General key info..: [none]` in the output instead - go back and import the public key using the previous step.
{: .prompt-warning }



### Using Keys

- This section will go over how to sign and encrypt using your keys


#### Signing

Sign a message:
```terminal
$ echo "test message string" | gpg --armor --clearsign > signed.txt
```

> For some reason right now this only works if I first do:
> ```terminal
> gpg --clearsign --armor file.txt
> ```
>
> - *If not I get "Inappropriate ioctl for device"*
{: .prompt-warning }


Verify the signature:
```terminal
$ gpg --verify signed.txt
gpg: Signature made Wed 25 May 2016 00:00:00 AM UTC
gpg:                using RSA key 0xBECFA3C1AE191D15
gpg: Good signature from "John Smith <john@smith.test>" [ultimate]
Primary key fingerprint: 011C E16B D45B 27A5 5BA8  776D FF3E 7D88 647E BCDB
     Subkey fingerprint: 07AA 7735 E502 C5EB E09E  B8B0 BECF A3C1 AE19 1D15
```

#### Encryption/Decryption

Encrypt string to file:
```terminal
$ echo "test message string" | gpg --encrypt --armor --recipient $KEYID -o encrypted.txt
```

Decrypt generated encrypted file to string:
```terminal
$ gpg --decrypt --armor encrypted.txt
gpg: anonymous recipient; trying secret key 0x0000000000000000 ...
gpg: okay, we are the anonymous recipient.
gpg: encrypted with RSA key, ID 0x0000000000000000
test message string
```

> **Also created shell functions that automate this process**:
> ```bash
> function secret() {
>     output=$PWD/"${1}".$(date +%Y-%m-%d).enc
>     echo "Who do you want to send it to?"
>     gpg --list-keys --with-colons | awk -F: '/^uid/{print $10}'
>     echo -n "-> "
>     read recipient
>     gpg --sign --encrypt --armor --output ${output} -r ${recipient} "${1}" && echo "${1} -> ${output}"
>     [ -f ${output} ] && rm -f "${1}"
> }
>
> function reveal() {
>     output=$(echo "${1}" | rev | cut -c16- | rev)
>     gpg --decrypt --output ${output} "${1}" && echo "${1} -> ${output}"
>     [ -f ${output} ] && rm -f "${1}"
> }
> ```
>
> ```terminal
> $ secret <file.txt>
> <file.txt> -> $PWD/<file.txt>.timedate.enc
>
> $ reveal <file.txt>.timedate.enc
> gpg: encrypted with RSA key, ID 0x0000000000000000
> gpg: encrypted with RSA key, ID 0x0000000000000000
> gpg: anonymous recipient; trying secret key 0x0AD1233456789122 ...
> gpg: anonymous recipient; trying secret key 0x0AD1233456789123 ...
> gpg: anonymous recipient; trying secret key 0xCAC1233456789124 ...
> gpg: anonymous recipient; trying secret key 0xCAC1233456789125 ...
> gpg: anonymous recipient; trying secret key 0xDD71233456789126 ...
> gpg: okay, we are the anonymous recipient.
> <file.txt>.timedate.enc -> <file.txt>
> ```
{: .prompt-tip }




## Additional Notes

This setup closely follows Yubikey's Guide to GnuPG as well as drduh's [YubiKey Guide](https://github.com/drduh/YubiKey-Guide)




## Q&A
....................................


#### How do I view my keyring?

> To view your keyring in GnuPG, you can use the following commands:
>
> ```terminal
> gpg --list-public-keys
> ```
>
> - this will display information about your **public keys** (pub)
> - 'sub' refers to subkey
>
> ```terminal
> gpg --list-secret-keys
> ```
>
> - this will show information about your **private keys** (sec)
> - 'sec#' refers to private key not located on local device (#)
> - 'ssb>' refers to private subkey stored on Smartcard (>)
>
> The output will provide details about the keys in your keyring, including their validity, expiration dates, and other relevant information.
{: .prompt-tip }

....................................

#### So is the method to prove authentication and integrity (from the sender's perspective), by signing a message first and then encrypting it?

> Here's how the process works:
>
>	1. Signing: The sender uses their private key to create a digital signature of the message. The digital signature is a unique cryptographic representation of the message that is created using the sender's private key. This signature provides proof of authenticity and integrity.
>
>	2. Encryption: After signing the message, the sender encrypts the entire message (including the signature) using the recipient's public key. This ensures that only the recipient with the corresponding private key can decrypt and access the message.
>
> By signing the message first, the sender establishes the authenticity and integrity of the message. The recipient can then verify the signature using the sender's public key to ensure that the message hasn't been tampered with and originated from the expected sender. The recipient can also decrypt the encrypted message using their private key to access the original content.
>
> This two-step process of signing and encrypting provides a secure way to communicate messages while ensuring both authentication and confidentiality. It allows the recipient to verify the sender's identity and integrity of the message while keeping the content protected from unauthorized access during transmission.
{: .prompt-tip }

....................................

#### So I have 3 subkeys. An authentication, signing, and encryption key. Is my singular public key used for all of these keys?

> Yes, a singular public key is used for all of your subkeys. In OpenPGP (which GnuPG is based on), a key pair consists of a primary key and one or more subkeys. The primary key is responsible for key management and certification, while the subkeys are used for specific cryptographic operations like signing, encryption, and authentication.
>
> When you generate your key pair, the primary key is associated with your identity and is used for certification purposes. The subkeys, on the other hand, are generated alongside the primary key and are linked to it. Each subkey has its own specific purpose, such as signing, encryption, or authentication.
>
> Your public key, which is shared with others, contains all the necessary information about your primary key and subkeys. When someone wants to send you an encrypted message, they will use the encryption subkey from your public key. If they want to verify a signature you made, they will use the signing subkey. And if they want to authenticate your identity, they will use the authentication subkey.
>
> So, while you have separate subkeys for different purposes, they are all part of the same public key, which represents your identity and cryptographic capabilities.
{: .prompt-tip }

....................................


#### I am using a yubikey to store my private keys. Why are there private keys stored in the `~/.gnupg/private-keys-v1.d` directory

> The presence of private key files in the `~/.gnupg/private-keys-v1.d`{: .filepath} directory is a normal behavior in GnuPG, even when you are using a YubiKey or other smart card as the storage for your private keys.
>
> GnuPG needs to maintain a local copy of the private key metadata for various operations, such as key listing, key revocation, or generating revocation certificates. These metadata files in the `private-keys-v1.d` directory are used to store information about the private keys, including their key IDs and associated metadata.
>
> However, it's important to note that the actual private key material is not stored on your local machine in the `~/.gnupg` directory when using a smart card like YubiKey. The private key material is securely stored on the smart card itself, and GnuPG interacts with the smart card to perform cryptographic operations using the keys stored on the card.
>
> The private key files in the `private-keys-v1.d` directory are necessary for GnuPG to manage and interact with the private keys stored on the smart card, but they do not contain the actual private key material.
{: .prompt-tip }

....................................


#### I did `gpg --armor --export 0x22131D12C070F53B` my public key block, how come all of my subkey public keys are identical to each other and the primary public key?

> You have one public *keyblock*, which *most people* call a "PGP public key" in the general sense. (It could be also called a "certificate".) As you've seen in the linked article, this public keyblock consists of multiple packets:
> ```
> your "primary" public key parameters (the cryptographic values for RSA/DSA/etc)
> ├─ metadata for that public key (creation/expiry time, etc)
> ├─ a list of userids (name+email labels)
> │  └─ each userid followed by a list of signatures (certifications)
> └─ a list of "subkey" public key parameters (RSA numbers, etc)
>    └─ each subkey followed by a list of self-signatures (certificate)
>```
>
> However, it's not the same thing as "one public key" in the cryptographic sense, because it has *multiple sets* of actual RSA/DSA/etc. *public parameters* inside it.
>
>
> > How exactly are the subkeys "associated" with the primary key? There's lots of articles saying they are associated, but none saying how. Do they contain some shared mathematical properties?
>
> They're stored along with a signature (self-certification) made by the primary key. Mathematically, however, they are completely independent and could be generated for different algorithms.
>
> > Which keys should I be publishing?
> > Should I be publishing my primary key, or my subkeys, or all of them? What are the pros/cons of each?
>
> All of them, as a single unit (keyblock).
>
> Each individual key is needed for its own task – the primary key has to be public as it is needed to verify signatures (certifications) you make on other people's keys (userids), and on your own subkeys and userids. The encryption subkey has to be public in order for others to actually encrypt a message to it. And so on.
>
> > Which keys should my friends sign?
>
> In the user interface, the signing process starts by specifying your primary key (by ID or fingerprint). The software then does the correct thing.
>
> Technically, none of the keys get signed. Other people sign your *userids*, which are text labels (name+email) associated with the primary key. The purpose of signing another person's key is to vouch for the binding between the key and the name+email; signing just the key alone would be useless.
>
> > If my friend is adding me to his keyring, does he add my primary key, my subkey(s), or all my keys?
>
> Your friends add the entire key block – primary key, subkeys, userids.
>
> > If I set up a new subkey, how can he validate that it's associated with my primary key?
>
> Your own subkeys and userids are "self-certified", i.e. they are automatically signed by your primary key as soon as you create them. This is why it's enough to distribute the fingerprint of your primary key – it acts as the verification root.
{: .prompt-tip }
