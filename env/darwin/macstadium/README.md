# Overview

Go's Mac builders run at Mac hosting provider,
[MacStadium](https://macstadium.com). We have a VMware cluster with 8
physical Mac Pros on which we run two VMs each. There is also a single M1 Mac Mini.

In addition to the Mac VMs and M1 Mac mini, we also run one a Linux VM that's our
bastion host & runs various services.

## Bastion Host

The bastion host is **macstadiumd.golang.org** and can be accessed
via:

    $ ssh -D :1080 -i ~/keys/id_ed25519_golang1 gopher@macstadiumd.golang.org

(Where `id_ed255519_golang1` is available from go/go-builders-ssh)

It also runs:

* a DHCP server for the 16 Mac VMs to get IP addresses from (systemd
  unit `isc-dhcp-server.service`, so watch with `journalctl -f -u
  isc-dhcp-server.service`)

* the [**makemac** daemon](../../../cmd/makemac/) daemon (systemd
  unit `makemac.service`, so watch with `journalctl -f -u makemac`).
  This monitors the build coordinator (farmer.golang.org) as well as
  the VMware cluster (via the [`govc` CLI
  tool](https://github.com/vmware/govmomi/tree/master/govc)) and makes
  and destroys Mac VMs as needed. It also serves plaintext HTTP status
  at http://macstadiumd.golang.org:8713 which used to be accessible in
  a browser, but recent HSTS configuration on `*.golang.org` means you
  need to use curl now. But that's a good way to see what it's doing
  remotely, without authentication.

* [WireGuard](https://www.wireguard.com/) listening on port 51820,
  because OpenVPN + Mac's built-in VPN client is painful and buggy.
  See config in `/etc/wireguard/wg0.conf`. This doesn't yet come up on
  boot. It's a recent addition.

## OpenVPN

The method of last resort to access the cluster, which works even if
the bastion host VM is down, is to VPN to our Cisco gateway via go/go-how-to-vpn-into-macstadium.

## VMware web UI

Connect with OpenVPN, WireGuard, or configure your browser to use
localhost:1080 as a SOCKS proxy. Add:

10.87.58.9      vcenter.gglgtm-a-002.macstadium.com

to your hosts file, then access the UI at https://vcenter.gglgtm-a-002.macstadium.com.

VMware web UI at:

   https://10.87.58.9/ui/

## Adding a New Image

When a new version of macOS is released:

* Ensure that the version of vSphere deployed on MacStadium supports the
  new version of macOS. If it doesn't, either request that MacStadium
  upgrade the cluster or seek guidance from them about the upgrade path.

* Clone the latest macOS version on vSphere and upgrade that version
  to the desired macOS version as per the [instructions](vmware-notes.md).

* If a completely new image is required, follow the [images setup notes](../setup-notes.md)
  in order to add a new image.

## Debugging

Common techniques to debug:

* Can you get to the bastion host?

* What does `journalctl -f -u makemac` say? Is it error looping?

* Look at https://10.87.58.9/ui/ and see if VMware is unhappy about
  things. Did hosts die? Did storage disappear?

* Need to hard reboot machines? Eventually we'll fix
  https://github.com/golang/go/issues/32033 but for now you can use
  https://portal.macstadium.com/subscriptions and power cycle
  machines that VMware reports dead (or that you see missing from
  [https://farmer.golang.org](https://farmer.golang.org)'s reverse pool info).

* Worst case, file a ticket: https://portal.macstadium.com/tickets
