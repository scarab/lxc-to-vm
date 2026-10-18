# Debian 7 (Wheezy) LXC → VM migration

> **Legacy and unsupported OS.** Debian 7 is end-of-life. This compatibility path
> exists to preserve an existing installation; it is not a recommendation to
> deploy Wheezy for new services. Test on an isolated network where possible.

## Scope

The converter detects Wheezy when the source container's
`/etc/debian_version` starts with `7.` or equals `7`.
For this guest only, `lxc-to-vm.sh`:

- Requires SeaBIOS (MBR) and creates ext4 without modern features that
  Linux 3.2 and GRUB 1.99 cannot read, including `64bit`,
  `metadata_csum`, `metadata_csum_seed`, `orphan_file`, and `fast_commit`.
- Installs `linux-image-amd64`, `initramfs-tools`, and `grub-pc` in the
  **copied VM rootfs**, without requiring `qemu-guest-agent` or changing init.
- Uses legacy `grub-install --no-floppy --recheck --force` syntax on the
  temporary *loop device*. It does not select a host physical disk for GRUB.
- Allows expired repository signatures only on the copied guest's APT install
  command; the source CT and Proxmox host APT configuration are unchanged.
- Attaches the new root disk as IDE (`ide0`), disables the QEMU guest agent,
  and skips guest-agent-based health checks.
- Uses `eth0`, preserving the container `net0` MAC and (by default)
  translating its IP/prefix, gateway, and MTU to `/etc/network/interfaces`.
  The original guest interfaces file is saved as
  `/etc/network/interfaces.pre-lxc-to-vm` inside the new VM.

For modern distributions, existing disk and boot logic remains unchanged.
Wheezy currently supports IPv4 (DHCP, static, or manual) for `net0`; review
guest networking separately if you also have IPv6, VLANs, extra NICs, unusual
persistent-udev rules, or routing configuration.

## Before converting

1. Ensure you have an independently restorable backup. A backup stored on
   the same server does **not** protect against physical disk failure.
2. Disable automatic startup of the source CT before the migration test.
   The converted VM retains the source IP **and MAC**, so both machines
   must never be online on the same L2 network simultaneously.
3. Check that archived Wheezy packages are present in the guest's APT cache
   and that the guest can resolve/reach `archive.debian.org`.
4. Keep `--destroy-source` and `--start` **off** during the first test.

For example, on Proxmox VE:

```bash
pct set 102 --onboot 0
vzdump 102 --mode stop --dumpdir /mnt/fast/dump --compress zstd
./lxc-to-vm.sh -c 102 -v 104 -s fast -d 20 -b vmbr2 -t /mnt/fast/dump
```

The conversion stops the source CT for a consistent copy. It **does not**
delete the source CT unless you explicitly request that behavior.

## First boot

Check the VM configuration before starting:

```bash
qm config 104
pct status 102       # must be stopped
qm start 104
```

Open the VM graphical console from the Proxmox interface. Debian 7 has no
QEMU guest agent in its Wheezy archive, so the guest must be inspected
manually. Verify:

```bash
cat /etc/debian_version
uname -r
ip addr show eth0
ip route
mount
```

If boot fails, collect the GRUB/kernel console message and
`tail -n 150 /var/log/lxc-to-vm.log` from the Proxmox host.
Stop VM 104 before restarting CT 102 to avoid IP and MAC collisions.

## Security and limitations

Archived GPG signing keys have expired. The converter uses the Wheezy
APT `AllowUnauthenticated` option for the **copied guest only**, which
reduces package authenticity assurance. Use a trusted, controlled network
and validate packages separately if possible.

This path is not tested against a live Wheezy machine in CI. GitHub Actions
checks Bash syntax and ShellCheck, but only an actual VM boot can validate
old GRUB, kernel driver availability, init and networking behavior.

Using `--keep-network` deliberately preserves the guest interfaces file
and does **not** inject a new `ens18` or IP setup. LXC often configures
the network at runtime, so the default mode is recommended for Wheezy.
