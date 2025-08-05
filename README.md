# fleetsie

Simple tool to provision and manage a fleet of linux-based iot devices.

## Parts

**`fleetsiemod`**: modifies a stock OS image for use with `fleetsie`

**`fleetsie_provision`**: script which provisions the system by using the code, configuration and credentials
supplied on an attached USB disk.  The script is intended to be run once, at provisioning time.

**`fleetsie.service`**: systemd service that runs `fleetsie_provision` when the system is **stable**.
The meaning of **stable** might depend on the OS.  Disables itself after a successful run of
`fleetsie`.

## `fleetsiemod` - Manage Creation of a Modified OS Image

### Requirements
The host where `fleetsiemod` is used to modify an OS image requires these packages:

```sh
sudo apt install xz-utils kpartx awk unionfs-fuse
```

### Commands:

```sh
  fleetsiemod init PATH/OSNAME.img.xz
```

- decompresses the image using xz and writes it to file OSNAME.img in the current directory
- mounts all partitions in the image, as is done by the `fleetsie mount` command below
- creates a symlink from `.image -> OSNAME.img`
- initializes a git repo in `my_work`, where overlays for each of the partitions in `OSNAME.img`
will be created, so that you can use version control to track your work.

```sh
  fleetsiemod mount
```

- mounts each partition in `OSNAME.img` read-only to directory `original/part_N`
- creates an overlay mount for each partition; the merged mount is in `part_N`, where N is 1, 2, ...
- changes to the underlying image will be reflected in directories `.new_N`; these are created
if they do not already exist.
- symlinks are created from the underlying partition labels to part_N; e.g. `bootfs -> part_1`

```sh
  fleetsiemod unmount
```

- unmounts the merged and original filesystems
- changes made remain available in the `.new_N` directories and will be
automatically restored the next time `fleetsie mount OSNAME.img` is run

```sh
  fleetsiemod install [OSTYPE]
```

- uses instructions customized for `OSTYPE` to install
  `fleetsie.service` and `fleetsie_provision` in the overlain OS
  image.
- `OSTYPE` defaults to `Raspberry Pi OS`; do `fleetsiemod install --help` to list other options, if any.
- after this command, you can make any further customizations to the image by manipulating files
in the `part_N` directories
- do `fleetsiemod save` to create a new installable image that includes `fleetsie` and your changes

```sh
  fleetsiemod save [NEW_IMAGE_NAME]
```

- writes the filesystem with your changes to a new, xz-compressed image
- if `NEW_IMAGE_NAME` is omitted, it defaults to `OSNAME_fleetsie.img.xz`,
- normally, you would only do `fleetsiemod save` after doing `fleetsiemod install`
and, optionally, making further changes to the `part_N` directories

```sh
  fleetsiemod updatesd DEVPART1 DEVPART2 ...
```

- updates the filesystem partitions on an attached SD card from the current image
- `DEVPART1` is updated from the partition 1 (at `merged/part_1`)
- `DEVPART2` is updated from the partition 2 (at `merged/part_2`)
- and so on...
- don't include the `/dev/` in `DEVPARTN`
- example:  `fleetsiemod updatesd sdc1 sdc2`

## fleetsie.service

- installed and enabled by `fleetsiemod` on the OS image
- once the system is **stable**, runs the `fleetsie` script, which is also installed by `fleetsiemod`
- if `fleetsie` exits with success, `fleetsie.service` disables itself.

## fleetsie_provision

- installed into /usr/bin by `fleetsiemod` on the OS image
- run by the `fleetsie.service` which is also installed and enabled there
- only runs when the system is **stable**
- is disabled from running after the first successful run

- searches for a script to run on an attached USB disk.
- the script must be called `setup` and must be in a top-level directory called `fleetsie` on the USB disk
- after switching to the top-level `fleetsie` directory on the disk, the script is run as root
- if the script run is successful, `fleetsie.service` will disable itself.
