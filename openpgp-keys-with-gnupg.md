# OpenPGP Key Management with GnuPG

In this guide [GnuPG] is used to generate four (4) OpenPGP keys and
ensure that they are safely backed up. The keys can be used to:

- sign Git commits/tags
- encrypt/decrypt private files
- authenticate SSH connections
- manage subkeys

## Prepare to Safety Generate Keys

The most secure way to create the keys is from a secure operating system
without persistent storage. My personal favorite is [Tails]. After
booting into Tails a Welcome Screen will eventually show with Additional
Settings available. Disable all networking and set an administrator
password before continuing. The root account is needed to encrypt and
create a file system on USB drives for storing the OpenPGP keys. In
addition to disabling the networking, it may be wise to actually
disconnect physically from the network to be certain that no internet
connection persists. This is important because we **never want our
Certification key to exist on a networked computer**.

## Create Encrypted USB drives to Store Keys

Create three (3) encrypted USB drives for storing the keys.

1. The first is for using the certification key and can only be used in
   an [air-gapped] environment. All four (4) keys will be stored on this
   device, in addition to the owner trust data.
2. The second is just a copy of the first in case of USB drive failure.
3. The third is for daily use and will contain only the three (3)
   subkeys, public key, and ownertrust.

See ["Creating and using LUKS encrypted volumes"][luks-tails] for
information on preparing the USB drives on Tails OS. Also check out
["Disk Encryption with LUKS"][luks-encryption] for command line
instructions for other Linux distributions.

## Generate Keys

To generate an OpenPGP certification key:

```sh
gpg --quick-generate-key "$GIT_COMMITTER_NAME <$EMAIL>" ed25519 cert 1y
```

Be sure to change `$GIT_COMMITTER_NAME` and `$EMAIL` to their actual
values for signing Git commits. Alternatively, the shell configuration
can be used to `export` the values. Git can use these environment
variables instead of setting `user.name` and `user.email` in the
`.gitconfig`.

```sh
export GIT_COMMITTER_NAME='Your Name'
export EMAIL='youremail@example.com'
```

It may be preferrable for the key to expire on an exact date rather than
one year from now. Change the `1y` value to a date such as `2023-01-01`
if that makes it easier to remember to extend the keys expiration.

Take note of the 40 character long hexadecimal string in the output
after generating the certification key. This is the key's fingerprint.
It may be useful to export this fingerprint as an environment variable
for use in upcoming steps.

```sh
export FPR="$(gpg --list-secret-keys --with-colons --with-fingerprint "$GIT_COMMITTER_NAME <$EMAIL>" | \
  grep -A 2 '^sec' | grep '^fpr' | cut -f 10 -d :)"
```

To generate subkeys for encryption, authentication and signing:

```sh
gpg --quick-add-key "$FPR" cv25519 encr
gpg --quick-add-key "$FPR" ed25519 auth
gpg --quick-add-key "$FPR" ed25519 sign
```

To verify that all the keys have been created, run:

```sh
gpg --list-secret-keys
```

We used ed25519 and cv25519 EdDSA schemes to generate the keys. These
may not be supported everywhere and some may prefer to use RSA instead.
I have not had any problems since Dropbear gained support in version
2013.61test.

## Disaster Level Backup

This is just one method to backup the most important data from GnuPG in
case both of our USB drives containing the certification key fail. We
will be printing out the **certification key**, **encryption key**, and
our **revocation certificate** on paper and storing them in a secure
location away from the USB drives.

First, we'll back up the certification key. This key is important because
it is used to manage all of the other keys. It can also be important if
you've spent time building up a web of trust.

Export the private PGP certification key:

```sh
gpg --armor --export-secret-keys --output /tmp/certkey.asc "$FPR!"
```

Next, we want to save the encryption key. Without this key, any files
encrypted with it will be useless.

Set environment variable to store the encryption key fingerprint:

```sh
export ENCR="$(gpg --list-secret-keys --with-colons --with-fingerprint "$FPR" | \
  grep -A 2 '\(.*:\)\{11\}e:' | grep '^fpr' | cut -f 10 -d :)"
```

Export the private PGP encryption key:

```sh
gpg --armor --export-secret-subkeys --output /tmp/encrkey.asc "$ENCR!"
```

We will also save the revocation certificate. This can be generated from
the certification key, but it is much easier (shorter) to type the
revocation certificate if it is needed.

Export revocation certificate:

```sh
gpg --armor --gen-revoke --output /tmp/revcert.asc "$FPR"
```

Print the three (3) keys on quality paper, write the password in the
margin, and store in a safe deposit box at your local bank. The password
will be changed before we export the subkeys for daily use, so it is
critical to take note of the current password for the exported keys. Be
sure to change the `PRINTER` variable to the actual printer name.

**NOTE**: Do not follow this step if you are a criminal. In that case,
you will have to memorize your keys and study/practice techniques to
resist torture in order to properly protect the keys.

To find out the Printer Name to send the print job:

```sh
lpstat -a | awk '{print $1}'
```

To print the keys:

```sh
export PRINTER=Printer-Name

printf '# certkey\n\n%s\n\n# encrkey\n\n%s\n\n# revcert\n\n%s\n' \
  "$(cat /tmp/certkey.asc)" \
  "$(cat /tmp/encrkey.asc)" \
  "$(cat /tmp/revcert.asc)" \
  | lp
```

**NOTE**: Do not print out the keys on a network printer and do not use
any printer other than your own.

### Export Keys

Export the private PGP keys:

```sh
gpg --armor --export-secret-keys --output /tmp/prvkey.asc "$FPR"
```

Export public keys:

```sh
gpg --armor --export --output /tmp/pubkey.asc "$FPR"
```

Export trust information:

```sh
gpg --export-ownertrust > /tmp/otrust.txt
```

Change password and export all of the private PGP subkeys:

```sh
gpg  --edit-key "$FPR"
> passwd
> save
gpg --armor --export-secret-subkeys --output /tmp/subkey.asc "$FPR"
```

### Save the Keys

Now there is a paper copy of the most important data in case of drive
failure. Getting those keys from paper to your computer can be a bit of
a hassle, so we'll copy the keys to a second encrypted USB drive to
ensure entering keys from paper is a last resort. Let's save all of our
keys.

```sh
export USB=/USB_MOUNT_PATH
export USB_BAK=/USB_BAK_MOUNT_PATH
export USB_DAILY=/USB_MOUNT_PATH

cp -p /tmp/prvkey.asc "$USB"/
cp -p /tmp/otrust.txt "$USB"/

cp -p /tmp/prvkey.asc "$USB_BAK"/
cp -p /tmp/otrust.txt "$USB_BAK"/

cp -p /tmp/subkey.asc "$USB_DAILY"/
cp -p /tmp/pubkey.asc "$USB_DAILY"/
cp -p /tmp/otrust.txt "$USB_DAILY"/
```

Here are some tables to show the USB drive file organization to make it
easier to see where all of the keys are.

| Drive | Purpose           | Files                              |
| ----- | ----------------- | ---------------------------------- |
|   A   | Subkey Management | prvkey.asc, otrust.txt             |
|   B   | Key Backup        | prvkey.asc, otrust.txt             |
|   C   | Daily Use         | subkey.asc, pubkey.asc, otrust.txt |

| File        | Keys In File           |
| ----------- | ---------------------- |
| prvkey.asc  | cert, auth, encr, sign |
| pubkey.asc  | public                 |
| subkey.asc  | auth, encr, sign       |
| otrust.txt  | trust database         |

Unmount all three (3) USB drives, log out of the Tails OS, and reboot
into your normal operating system.

## Finally Time to Use the Keys

Once logged back into your daily computer, decrypt and mount the
USB drive containing your daily use keys. Import the subkeys:

```sh
gpg --import /path/to/pubkey.asc
gpg --import /path/to/subkey.asc
gpg --import-ownertrust /path/to/otrust.txt
```

### Set Up SSH Authentication

To configure SSH authentication to use the gpg-agent instead of the
ssh-agent, add the following to your shell environment:

```sh
# ~/.bashrc
# -----------------------------------------------------

unset SSH_AGENT_PID

SSH_AUTH_SOCK="$(gpgconf --list-dirs agent-ssh-socket)"
export SSH_AUTH_SOCK

GPG_TTY="$(tty)"
export GPG_TTY

gpg-connect-agent updatestartuptty /bye >/dev/null 2>&1
```

**NOTE**: Different systems may need different configuration than shown
above. I've had success using this setup on Arch Linux and Alpine Linux.
Check out the [Arch Linux GnuPG SSH agent wiki page][ssh-agent] for more
details.

To enable the authentication key that we created above, we need to add
the key's keygrip to `~/.gnupg/sshcontrol`.

```sh
export GRP="$(gpg --list-secret-keys --with-colons --with-fingerprint "$FPR" | \
  grep -A 2 '\(.*:\)\{11\}a:' | grep '^grp' | cut -f 10 -d :)"

echo "$GRP" >> "$HOME"/.gnupg/sshcontrol
```

The public SSH key can be retrieved by running:

```sh
gpg --export-ssh-key "$FPR"

# or even better, copy it to the clipboard
gpg --export-ssh-key "$FPR" | xclip
```

Add this key anywhere that you typically connect to with SSH, such as:

- Code Repositories (GitHub, GitLab, Codeberg, Sourcehut, etc)
- Media Center (Kodi)
- Routers (OpenWRT)
- Remote Servers

### Signing Git Commits/Tags

Copy the public key to the clipboard and add it to remote code
repositories for signature verification.

```sh
gpg --export --armor "$FPR" | xclip
```

Now commits can be signed by using `git commit -S` or always require GPG
signatures by setting the option in Git configuration:

```sh
git config --global commit.gpgSign true
```

### Use Web Key Directory (WKD) for Key Distribution.

If the email address attached to the keys uses a personal domain, then
our personal site can self host the key distribution. The following commands
should be run in the root directory of the website.

```sh
WKD="$(gpg --list-keys --with-wkd-hash "$FPR" | grep -oE '[a-z0-f]{32}')"
export WKD

mkdir -p .well-known/openpgpkey/hu
touch .well-known/openpgpkey/policy
gpg --export --armor --output .well-known/openpgpkey/hu/"$WKD" "$FPR"
```

If using Netlify for hosting, then run the following to configure the
required headers.

```sh
printf './well-known/openpgpkey/hu/%s\n  Access-Control-Allow-Origin: *' "$WKD"
```

## Key Maintenance

### Extend Expiration Date

1. Log into Tails OS.
2. Decrypt and mount the USB drive that contains the private
   certification key.
3. Import the keys: `gpg --import /<path/to>/prvkey.asc`.
4. Edit the key's expiration date: `gpg --quick-set-expire "$FPR" 1y`.
   Change `1y` to an actual date, such as `2025-01-01` if preferred.
5. Export the public key `gpg --armor --export --output pubkey.asc "$FPR"`.
6. Move the `pubkey.asc` file to the "daily use subkeys" USB drive.
7. Replace the public keys everywhere with the new `pubkey.asc`.

**NOTE**: There is no need to update the backups of the private keys.
The signature of the expiration date is attached to the public key.

## References

- <https://github.com/lfit/itpol/blob/master/protecting-code-integrity.md>
- <https://wiki.archlinux.org/index.php/GnuPG>
- <https://wiki.debian.org/Subkeys>
- <https://wiki.debian.org/GnuPG/AirgappedMasterKey>
- <https://metacode.biz/openpgp/web-key-directory>
- <https://datatracker.ietf.org/doc/html/draft-koch-openpgp-webkey-service>
- <https://keys.openpgp.org/about/usage>
- <https://crp.to/p>

[tails]: https://tails.boum.org/install/index.en.html
[air-gapped]: https://en.wikipedia.org/wiki/Air_gap_%28networking%29
[gnupg]: https://gnupg.org
[luks-tails]: https://tails.boum.org/doc/encryption_and_privacy/encrypted_volumes/index.en.html
[luks-encryption]: luks-encryption.md
[ssh-agent]: https://wiki.archlinux.org/title/GnuPG#SSH_agent
