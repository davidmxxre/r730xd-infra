# Onboarding: getting into the iDRAC from a cold start

Use this when you've been away and need out-of-band access again. Valid while the
iDRAC is on the provisional direct-cable link (ADR-0002). The network module will
replace it with management-VLAN access over WireGuard.

## What you need

- The MacBook with the USB/Thunderbolt Ethernet adapter, cabled to the iDRAC's
  dedicated port (rear, wrench icon).
- The RHEL VM `app1` in VMware Fusion.
- The iDRAC private key `~/.ssh/id_rsa_idrac` in the VM (and its passphrase).
- The `dave` iDRAC password (web UI and Virtual Console).

## 1. Network path

Packet path: VM `app1` → Fusion bridged adapter → USB Ethernet adapter → cable →
iDRAC 192.168.0.120.

1. **Fusion:** Virtual Machine → Settings → the second Network Adapter →
   Bridged Networking → the Ethernet adapter **by name** (not Autodetect, not Wi-Fi).
2. **In the VM:**

   ```
   ip -br addr
   nmcli connection up idrac-link
   ping -c 3 192.168.0.120
   ```

   - `ip -br addr`: brief list of interfaces and addresses. 192.168.0.10/24 must be
     on the **bridged** NIC.
   - `nmcli connection up idrac-link`: activate the saved profile. If it is on the
     wrong NIC: `nmcli connection modify idrac-link connection.interface-name <iface>`.
   - `ping -c 3`: three echo requests.

   First-time setup of the profile:

   ```
   nmcli connection add type ethernet ifname <iface> con-name idrac-link \
     ipv4.method manual ipv4.addresses 192.168.0.10/24 ipv6.method disabled
   ```

   No gateway, on purpose: the default route stays on the NAT adapter.

3. **If ping fails,** follow "iDRAC unreachable" in `/docs/runbooks/idrac.md`.

## 2. SSH configuration (VM)

`~/.ssh/config`:

```
Host idrac
    HostName 192.168.0.120
    User dave
    IdentityFile ~/.ssh/id_rsa_idrac
    IdentitiesOnly yes
```

Permissions: `chmod 600 ~/.ssh/config`. Test:

```
ssh -o PasswordAuthentication=no idrac racadm getsysinfo
```

`-o PasswordAuthentication=no` proves the key, not a password, logged you in.

Optional: `echo 'IDRAC=192.168.0.120' >> ~/.bashrc` so `$IDRAC` exists in every shell.

## 3. Web UI and Virtual Console (Mac)

1. macOS Network → the USB Ethernet adapter → TCP/IP → Manually:
   192.168.0.99, 255.255.255.0, no router.
2. Browser → `https://192.168.0.120`, accept the self-signed certificate,
   log in as `dave`.
3. Dashboard → Virtual Console → Launch (HTML5, encrypted).

## 4. New key or new machine

```
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_idrac -C "dave-idrac"
ssh idrac "racadm sshpkauth -i 3 -k 2 -t '$(cat ~/.ssh/id_rsa_idrac.pub)'"
```

- `-t rsa -b 4096`: RSA 4096 (iDRAC8 has no Ed25519 support).
- `sshpkauth -i 3 -k 2`: user slot 3, key slot 2 (slots 1–4 exist), so the old key
  in slot 1 keeps working until the new one is proven. Remove the old one with
  `racadm sshpkauth -i 3 -k 1 -d`.

Run the install from a machine that already has access, or use the web UI
(iDRAC Settings → User Authentication → Local Users → `dave` → SSH Key).
