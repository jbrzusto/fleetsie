# custom pre-provisioning file tree

This folder contains files used early in the `fleetsie` device
provisioning process by `fleetsie`, before the USB provisioning disk has
been mounted, and so very likely before the network is available.  It
lives in `/opt/fleetsie/custom_pre` on the disk image you create with
`fleetsie_mod save`.  You can customize it however you wish, but should
preserve any files already placed in this hierarchy by `fleetsie_mod install`.

When running on the provisioning device, `fleetsie_provision` treats a
few locations here specially, in the following order:

- `custom_pre/setup` is run, if it exists

- any `.deb` files in `./packages` are installed with a single `dpkg -i` command

- `overlay.tar.xz`, if it exists, is extracted to `/` on the device, preserving
file ownership and permissions

- `cleanup` is run, if it exists
