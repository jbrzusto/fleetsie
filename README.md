# fleetsie

Simple tool to provision and manage a fleet of linux-based iot devices.

## Parts

**`fleetsie_mod`**: modifies a stock OS image for use with `fleetsie`

**`fleetsie_provision`**: script which provisions the system by using the code, configuration and credentials
supplied on an attached USB disk.  The script is intended to be run once, at provisioning time.

**`fleetsie-provision.service`**: systemd service that runs `fleetsie_provision` when the system is **stable**.
The meaning of **stable** might depend on the OS.  Disables itself after a successful run of
`fleetsie_provision`.

**`fleetsie_gen`**: generate a USB provisioning disk and the server-side inventory for a fleet

**`fleetsie_srv`**: set up a server for use with `fleetsie_gen`

## `fleetsie_mod` - Manage Creation of a Modified OS Image

### Requirements
The host where `fleetsie_mod` is used to modify an OS image requires these packages:

```sh
sudo apt install xz-utils kpartx awk unionfs-fuse
```

### Commands:

```sh
  fleetsie_mod init PATH/OSNAME.img.xz
```

- decompresses the image using xz and writes it to file OSNAME.img in the current directory
- mounts all partitions in the image, as is done by the `fleetsie mount` command below
- creates a symlink from `.image -> OSNAME.img`
- initializes a git repo in `my_work`, where overlays for each of the partitions in `OSNAME.img`
will be created, so that you can use version control to track your work.

```sh
  fleetsie_mod mount
```

- mounts each partition in `OSNAME.img` read-only to directory `original/part_N`
- creates an overlay mount for each partition; the merged mount is in `part_N`, where N is 1, 2, ...
- changes to the underlying image will be reflected in directories `.new_N`; these are created
if they do not already exist.
- symlinks are created from the underlying partition labels to part_N; e.g. `bootfs -> part_1`

```sh
  fleetsie_mod unmount
```

- unmounts the merged and original filesystems
- changes made remain available in the `.new_N` directories and will be
automatically restored the next time `fleetsie mount OSNAME.img` is run

```sh
  fleetsie_mod install [OSTYPE]
```

- uses instructions customized for `OSTYPE` to install
  `fleetsie.service` and `fleetsie_provision` in the overlain OS
  image.
- `OSTYPE` defaults to `Raspberry Pi OS`; do `fleetsie_mod install --help` to list other options, if any.
- after this command, you can make any further customizations to the image by manipulating files
in the `part_N` directories
- do `fleetsie_mod save` to create a new installable image that includes `fleetsie` and your changes

```sh
  fleetsie_mod save [NEW_IMAGE_NAME]
```

- writes the filesystem with your changes to a new, xz-compressed image
- if `NEW_IMAGE_NAME` is omitted, it defaults to `OSNAME_fleetsie.img.xz`,
- normally, you would only do `fleetsie_mod save` after doing `fleetsie_mod install`
and, optionally, making further changes to the `part_N` directories

```sh
  fleetsie_mod updatesd DEVPART1 DEVPART2 ...
```

- updates the filesystem partitions on an attached SD card from the current image
- `DEVPART1` is updated from the partition 1 (at `merged/part_1`)
- `DEVPART2` is updated from the partition 2 (at `merged/part_2`)
- and so on...
- don't include the `/dev/` in `DEVPARTN`
- example:  `fleetsie_mod updatesd sdc1 sdc2`

## fleetsie.service

- installed and enabled by `fleetsie_mod` on the OS image
- once the system is **stable**, runs the `fleetsie` script, which is also installed by `fleetsie_mod`
- if `fleetsie` exits with success, `fleetsie.service` disables itself.

## fleetsie_provision

- installed into /usr/bin by `fleetsie_mod` on the OS image
- run by the `fleetsie-provision.service` which is also installed and enabled there
- only runs when the system is **stable**
- is disabled from running after the first successful run

- searches for a script to run on an attached USB disk.
- the script must be called `setup` and must be in a top-level directory called `fleetsie` on the USB disk
- after switching to the top-level `fleetsie` directory on the disk, the script is run as root
- if the script run is successful, `fleetsie-provision.service` will disable itself.

## ssh-tunnel service

`fleetsie` uses ssh to connect devices to the fleet host.  During
provisioning, each device is assigned a unique set of ssh keys for
logging into the fleet host, and a unique tunnel port number.  The
`ssh-tunnel` service will maintain a reverse tunnel from that port on
the fleet host to its local ssh port (22).

## fleetsie_gen

`fleetsie_gen` generates a USB provisioning disk and server inventory
for a fleet of devices.

### Usage:

```sh
   fleetsie_gen FLEET_NAME FLEET_HOST [NUM] [USB_PARTITION]
```

where

- `FLEET_NAME` is the name of the fleet.  It should be short and
   composed of alphanumeric characters and underscores.  Devices
   belonging to the fleet will be assigned the hostnames `FLEET_NAME-1`,
   `FLEET_NAME-2`, ...
   Also, a user called `fleetsie_FLEET_NAME` will be created on the
   server, and devices in the fleet will login as that user.

- `FLEET_HOST` is the server (e.g. `whoflewby.org`) where the fleet
   will be hosted.  The user running `fleetsie_gen` must have ssh set-up
   on `FLEET_HOST` so that they can login to fleetsie@FLEET_HOST, and
   so that user has sudo privileges.

-  `NUM` (optional) is the number of devices to pre-allocate for the
   fleet on the server.  If this is zero or missing, no devices are
   pre-allocated.  Otherwise, SIZE new devices are allocated for the
   fleet, adding to any which are already there.  e.g. if the fleet
   already has 100 device allocated (from a previous use of
   `fleetsie_gen`), the new devices will be named `FLEET_NAME-101`,
   `FLEET_NAME-102`, ...

-  `USB_PARTITION` (optional) is the name of the disk partition on the
   user's machine (e.g. `sda2`) where the fleetsie files used for
   provisioning devices will be installed.  If missing, no
   provisioning files are installed anywhere; this allows you to just
   allocate new fleet devices on the server without creating a USB
   disk.

   If `USB_PARTITION` begins with a `/`, it is treated as a path to a
   directory, and `fleetsie_gen` will create or use a subdirectory there
   called `fleetsie` as the destination for installing files, rather
   than a disk partition. This can be used for testing.

`fleetsie_gen` creates files on a USB drive and on the fleet server, such
that a number of devices can be provisioned (using the USB drive) with
access to the server.

### USB drive layout
fleetsie_gen creates this layout on the USB drive:

```
/fleetsie
```

- top-level folder

```
/fleetsie/wifi.txt
```

- file containing ESSIDs and passwords for wifi networks, one per line
  i.e. line 1 = ESSID1, line 2 = password1, line 3 = ESSID2, line 3 =
  password2, ...  During provisioning, the device will attempt to
  connect to wifi using these credentials, one set at a time, until a
  connection succeeds.

```
/fleetsie/fleet.txt
```

- file containing the fleet hostname on line 1, and the fleet name on line 2

```
/fleetsie/devices.sqlite
```

- database containing allocated but unclaimed devices for this fleet.

```
/fleetsie/fleetsieauth.pub
/fleetsie/fleetsieauth
```

- ssh keys used by the device to login to the fleet server at
  provisioning time.  On the server, ssh is configured so that logging
  in with these keys runs the server-side provisioning code.  No other
  use for these keys is permitted by the server.

```
/fleetsie/ssh-tunnel.service
/fleetsie/ssh_tunnel.sh
```

- systemd service and the script it runs to maintain a ssh connection
  to the fleet server, with a port mapped from the server back to the
  local ssh server.  It also maintains a local port which maps to the
  zabbix data port on the fleet server.

```
/fleetsie/customize/setup
```

   - the `customize` folder is where fleetsie looks for other files
     you want to install.  If a `setup` script is found there, then it
     is run from within that directory run after all other
     provisioning steps have succeeded.  `fleetsie_provision` ignores
     all other files and subdirectories of /fleetsie/custom, so you
     can populated it with whatever directories and files are needed
     by your `setup` script.

EOF
    exit 1
}

## fleetsie_srv - set up a server for use with fleetsie_gen

`fleetsie_srv` is run on the fleet manager's PC like so:

```sh
fleetsie_srv USER@SERVER
```

where `USER` must either be `root`, or a user with sudo privileges on `SERVER`

This script will set up `SERVER` via ssh like so:

- install any of these packages which are missing:
  - sqlite3
- create user `fleetsie` with `sudo` privileges; you will be prompted to enter
  a password for this user
- create sqlite database `/home/fleetsie/fleets.sqlite` with this schema:

```sql
CREATE TABLE devices (
     id INTEGER UNIQUE PRIMARY KEY NOT NULL,  -- unique ID for device, across all fleets
     fleet VARCHAR NOT NULL,                  -- name of fleet device belongs to
	 id_in_fleet INTEGER NOT NULL,            -- unique ID for device within fleet
     fleetuser VARCHAR NOT NULL,              -- name of user device uses for ssh to fleet server; typically, fleetsie_FLEET
     hostname VARCHAR NOT NULL,               -- hostname for device; typically FLEET-ID_IN_FLEET
     hwid VARCHAR,                            -- hardware ID of device; NULL means no device registered to this record yet
     otp VARCHAR NOT NULL,                    -- one-time password used by device to register
     ts_generated DOUBLE NOT NULL,            -- unix timestamp for when this device record was generated
     ts_registered DOUBLE,                    -- unix timestamp for when this device was registered; NULL means not registered yet
     tunnel_port INTEGER NOT NULL,            -- TCP port mapped on server back to device SSH server port
     device_public_key VARCHAR NOT NULL,      -- public key which can be used to login as user 1000 on device
     device_private_key VARCHAR NOT NULL,     -- private key which can be used to login as user 1000 on device
     server_public_key VARCHAR NOT NULL,      -- public key which device will use to ssh into fleet server
     server_private_key VARCHAR NOT NULL,     -- private key which device will use to ssh into fleet server
     ip_provisioned_from VARCHAR NOT NULL     -- IP address from which request to provision this device originated
 );
 CREATE UNIQUE INDEX ON devices(hwid);
 CREATE UNIQUE INDEX ON devices(fleet, otp, hwid);
 CREATE UNIQUE INDEX ON devices(fleet, id_in_fleet);
```

This table, initially empty, holds pre-allocated device records for one or more
fleets.  Entries in this table will be created by fleet administrators
using `fleetsie_gen`.  When entries are created, these fields are left
NULL:

```
ts_provisioned
ip_provisioned_from
hwid
```

The NULL fields get set during device provisioning when a physical
device presents an unused OTP password to register itself with the
server.  It is not known in advance which physical device will end up
registered to each pre-allocated device, but `fleetsie_auth` ensures
that no two physical devices get registered to the same device record.

## `fleetsie_gen_srv`

This script runs the server-side of the `fleetsie_gen` process.
It runs as user `fleetsie`.

It manages three versions of fleet database:

### **Main**
- contains records for all allocated devices for all fleets
- lives in /home/fleetsie/fleets.sqlite
- new device records are created here because the set of ssh
  tunnel ports is shared across fleets on one server.

### **Fleet**
- contains device records for a single fleet, `FLEET`
- lives in `/home/fleetsie/fleets/FLEET/fleet.sqlite`
- owned by user `fleetsie_FLEET`
- new device records get copied here when generated in **Main**
- new registrations are copied from here to **Main** whenever
  `fleetsie_gen_srv` is run for fleet `FLEET`
- `fleetsie_auth` uses this database when a new device is provisioning

### **Provisioning**
- generated as `/home/fleetsie/fleets/FLEET/provisioning.sqlite`
- only contains the `id_in_fleet` and `otp` columns
- owned by user `fleetsie_FLEET`
- copied to the USB provisioning disk for a fleet by `fleetsie_gen`
- only contains records for the unclaimed devices in a fleet
- each time it is used to provision a device, the corresponding record
  is removed from the DB on the USB disk, and from
  `/home/fleetsie/fleets/FLEET/provisioning.sqlite` on the fleet server

whenever `fleetsie_gen_srv` is run for fleet FLEET, it does this:
- if the One-Fleet database for FLEET exists, any new registrations
  are copied from it to the Main database.

- if requested, new devices are allocated in the Main database
  - these are copied to the Fleet database
  - the reduced records (`id_in_fleet`, `otp`) are copied to the Provisioning database
