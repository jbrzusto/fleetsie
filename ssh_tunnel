#!/bin/bash
#
# set up ssh tunnels:
#   - map this device's reserved port on the fleetsie server back to the ssh port on this device
#   - map port 10051 on this device to port 10051 on the fleetsie server (zabbix)
#   - server login information is assumed to be in ~/.ssh/config, where "~" is for the user
#     under which ssh-tunnel.service runs (e.g. "pi" on a Raspberry Pi OS device).  This
#     allows logging in using just "ssh fleetsie"
#   - the reserved port is available as the target of the symlink in ~/.ssh/fleetsie_tunnel_port
#
# This script is launched from systemd service 'ssh-tunnel.service'
#
# Besides tunneling ports, this script creates a low-bandwidth task
# running on the server whose job is to maintain dynamic connection
# information there.  When this script exists, it should be restarted
# by its systemd service.

TUNNEL_PORT=`readlink ~/.ssh/fleetsie_tunnel_port`
if [[ ! "$TUNNEL_PORT" ]]; then
    echo "No valid tunnel port linked to by ~/.ssh/fleetsie_tunnel_port"
    exit 100
fi

ssh -oExitOnForwardFailure=yes -oControlMaster=auto -oControlPath=/tmp/ssh.fleetsie -Rlocalhost:${TUNNEL_PORT}:localhost:22 -L10051:localhost:10051 fleetsie
