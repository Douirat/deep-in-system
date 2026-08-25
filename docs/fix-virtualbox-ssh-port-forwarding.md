# Fix VirtualBox SSH Port Forwarding

## Diagnosis

Guest-side checks confirm sshd is healthy:

```
sudo systemctl status ssh
● ssh.service - OpenBSD Secure Shell server
   Active: active (running)

sudo ss -tlnp | grep ssh
LISTEN 0  4096  0.0.0.0:22  0.0.0.0:*  users:(("sshd",...))
LISTEN 0  4096  [::]:22     [::]:*     users:(("sshd",...))
```

The server itself is fine — sshd is listening on port **22** inside the guest.

The host-side error:

```
ssh -p 2222 server-host@127.0.0.1
kex_exchange_identification: read: Connection reset by peer
Connection reset by 127.0.0.1 port 2222
```

is a classic sign of a **port-forwarding mismatch** in VirtualBox. The rule is
almost certainly forwarding host port 2222 to guest port 2222 (nothing there)
instead of guest port 22 (where sshd actually listens).

## Steps to fix

1. **Open VirtualBox Network Settings**
   With the VM selected (or running), go to `Devices > Network Settings` in
   the VM window menu, or `Machine > Settings > Network` in the VirtualBox
   Manager. Make sure "Attached to" is set to **NAT**.

2. **Open Port Forwarding rules**
   Click the "Port Forwarding" button (or `Advanced > Port Forwarding`).
   You'll see a table of rules with Host Port and Guest Port columns.

3. **Fix the Guest Port value**
   Find (or add) the SSH rule. Host Port should be `2222`, but Guest Port
   must be `22` — not `2222`. If it currently says `2222` for both, that's
   the bug: change Guest Port to `22` and leave Host Port as `2222`.

4. **Leave Host IP and Guest IP blank**
   Empty means "listen on all host interfaces" and "route to the guest's
   default adapter," which is what you want here.

5. **Apply and retry from the host**
   Click OK to save the rule (no VM restart needed for NAT port forwarding
   changes). Then from your host terminal run:
   ```
   ssh -p 2222 server-host@127.0.0.1
   ```

## Other things to check if that alone doesn't fix it

- Make sure the VM is actually using **NAT** (not Bridged/Host-only) — port
  forwarding only applies to NAT.
- Watch for typos: an earlier attempt ran `sh -p 2222 localhost@127.0.0.1`
  (missing the second `s` in `ssh`), which produced an unrelated
  `cannot open 2222: No such file` error — not the reset issue.
- Once the Guest Port is corrected to 22, `ssh -p 2222 server-host@127.0.0.1`
  should get you to a host-key prompt instead of a connection reset.
