# fleetsie

Simple tool to provision and manage a fleet of linux-based iot devices.

## Parts

**`fleetsiemod`**: modifies a stock OS image for use with `fleetsie`
... coming soon!

## Requirements
The host where `fleetsiemod` is used to modify an OS image requires these packages:

```sh
sudo apt install xz-utils kpartx awk unionfs-fuse
```

## `fleetsiemod` - Manage Creation of a Modified OS Image

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

- uses instructions customized for `OSTYPE` to install `fleetsie` in the overlain OS image.
- `OSTYPE` defaults to `Raspberry Pi OS`; do `fleetsiemod install --help` to list others.
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
