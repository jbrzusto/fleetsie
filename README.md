# fleetsie

Simple tool to provision and manage a fleet of linux-based iot devices.

## Parts

**`fleetsiemod`**: modifies a stock OS image for use with `fleetsie`
... coming soon!

## Requirements
The host where `fleetsiemod` is used to modify an OS image requires these packages:

```sh
sudo apt install xz-utils kpartx awk mergerfs
```

## `fleetsiemod` - Manage Creation of a Modified OS Image

### Commands:

```sh
  fleetsiemod init PATH/OSNAME.img.xz
```

- decompresses the image using xz and writes it to file OSNAME.img in the current directory
- mounts all partitions in the image, as is done by the `fleetsie mount` command below

```sh
  fleetsie mount OSNAME.img
```

- mounts each partition in `OSNAME.img` read-only to directory `.original_N`
- creates an overlay mount for each partition; the merged mount is in `part_N`, where N is 1, 2, ...
- changes to the underlying image will be reflected in directories `.new_N`; these are created
if they do not already exist.
- symlinks are created from the underlying partition labels to part_N; e.g. `bootfs -> part_1`

```sh
  fleetsimod unmount OSNAME.img
```

- unmounts the merged and original filesystems
- changes made remain available in the `.new_N` directories and will be
automatically restored the next time `fleetsie mount OSNAME.img` is run
