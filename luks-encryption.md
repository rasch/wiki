# Disk Encryption with LUKS

The following command will wipe an entire device by overwriting it with
random data. Change `<device>` to an actual drive such as `/dev/sdc`.
**CAUTION**: this command will destroy all data stored on the device so
be sure that it is the correct device before continuing.

```sh
dd if=/dev/urandom of=<device> bs=4K status=progress
```

Encrypt the device. Change `<device>` to the same device used in the
previous step.

```sh
cryptsetup \
  --key-size 512 --hash sha512 --use-random --iter-time 5000 \
  luksFormat --type luks2 <device>
```

A filesystem must be created before data can be written to the encrypted
device. `<dm_name>` is used by LUKS for unlocking and mapping the device
to a new device name. Once again, change `<device>` to the same device
from previous steps and also choose a name for `<dm_name>`.

```sh
cryptsetup open <device> <dm_name>
mkfs.ext4 /dev/mapper/<dm_name>
cryptsetup close <dm_name>
```

To use the encrypted device it must be opened and mounted.

```sh
cryptsetup open <device> <dm_name>
mount /dev/mapper/<dm_name> /mnt/<path>
```

See the Arch Linux wiki page on [device encryption] for more detail.

[device encryption]: https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption
