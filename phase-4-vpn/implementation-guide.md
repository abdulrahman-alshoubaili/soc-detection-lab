# SOC Lab — Phase 4 Implementation Guide (hands-on runbook)

**Goal of the phase:** stand up a second site behind its own firewall, join the two
sites with an **encrypted IPsec site-to-site VPN** across an untrusted transit, then
prove that an attack you cannot read on the wire is still **detected at the point where
the firewall decrypts it**, with the real attacker named.

This is the *do-it-yourself* runbook: every step is something **you** type or click,
with a verification gate before you move on. Follow it top to bottom.

- **fw-a** (Site A / SOC firewall) — GUI `https://192.168.56.9`, SSH `ssh fw-a` (root). Shell is **csh** — no `2>&1` / `2>/dev/null`.
- **fw-b** (Site B / branch firewall) — GUI `https://192.168.56.13`. **SSH is denied** — drive it from the GUI/console only.
- **attacker** (branch host, `10.10.30.40`) — `ssh attacker`.
- **throwaway** (Site A target, `10.10.10.50`) — `ssh throwaway`.
- **sensor** — `ssh sensor` (taps: `enp0s8`=transit, `enp0s10`=Site A LAN, `enp0s9`=mgmt).

> If you restore the **`phase4-vpn`** snapshot, everything below is already built and the
> tunnel auto-establishes on boot — skip to **Part 6 (run the tests)**. The parts below
> are for building it from scratch (Phase 3 baseline) or understanding/repairing it.

---

## The target picture

```
  SITE A — SOC                 TRANSIT (untrusted)              SITE B — BRANCH
  10.10.10.0/24  ── fw-a ======= 10.10.0.0/24 ======= fw-b ──  10.10.30.0/24
  sensor,SIEM,   .10.1  WAN .0.1   vyos + KALI    WAN .0.2  .30.1   attacker .30.40
  client .10.50  DMZ .20.1        (on-path seat)
                        \___ IPsec tunnel: 10.10.10.0/24 <=> 10.10.30.0/24 ___/
                             IKEv2 · AES-GCM · pre-shared key
```

**The one idea:** on the transit the traffic is only ESP (unreadable); at the Site A LAN,
*after fw-a decrypts*, the sensor reads the whole attack and names `10.10.30.40`.

---

## PART 0 — Prepare the two sites (VirtualBox wiring)

Do this with the VMs **powered off**.

1. **Transit segment.** Both firewall WANs and vyos sit on one internal network
   (intnet `vlan20` = `10.10.0.0/24`). Confirm fw-a WAN NIC and fw-b WAN NIC are both on it.
2. **Branch segment.** The branch LAN is intnet `vlan30` (`10.10.30.0/24`). Put fw-b's LAN
   NIC and the attacker's LAN NIC on it:
   ```
   VBoxManage modifyvm attacker --nic2 intnet --intnet2 vlan30
   ```
3. **Promiscuous mode.** For any sensor tap NIC and the internal segments, set
   *Allow All* (Adapter → Advanced → Promiscuous Mode) so the sensor can see the wire.

**Verify (gate):** `VBoxManage showvminfo attacker --machinereadable | grep intnet2`
→ must print `intnet2="vlan30"`.

---

## PART 1 — Reach both firewalls (out-of-band)

Each firewall is managed only on the management network, never from the transit.

- fw-a: browse `https://192.168.56.9` (login `root`)
- fw-b: browse `https://192.168.56.13` (login `root`)

> **Trap — the VM console mangles paste.** With a non-US keyboard layout the VirtualBox
> console corrupts `{ } _ = "` when you paste. **Never** type swanctl config at the VM
> console — use the browser GUI (host keyboard) or push files over SSH on fw-a.

**Say (for the video):** *"Two independent firewalls, each managed on its own out-of-band
leg. Neither exposes management to the transit."*

---

## PART 2 — WAN addressing + let the peers see each other

On **each** firewall GUI → **Interfaces → WAN**, set the static WAN address:

- fw-a WAN = `10.10.0.1/24`
- fw-b WAN = `10.10.0.2/24`

Then, **critically**, on each WAN interface **uncheck** *Block private networks* and
*Block bogon networks* (Interfaces → WAN, scroll to bottom). `10.10.0.0/24` is a private
range, and if this box is ticked the firewall **silently drops the peer's IKE**.

**Verify (gate):** on fw-a shell, `ping -c2 10.10.0.2` → replies. The two firewalls are on
one L2 segment and ARP each other directly (vyos does **not** route between them).

---

## PART 3 — Build the IPsec tunnel (the core of the phase)

Same parameters on both ends: **IKEv2 · AES-GCM · SHA-256 · pre-shared key**, joining
`10.10.10.0/24` (A) with `10.10.30.0/24` (B).

### 3a. fw-b side — do it in the GUI (SSH is denied on fw-b)

**VPN → IPsec → Tunnel Settings** (or Connections, depending on version):

1. **Phase 1 / Connection:** Key Exchange = **IKEv2**; Remote gateway = `10.10.0.1`;
   Authentication = **Mutual PSK**.
2. **Pre-Shared Key:** create the secret with your key string (e.g. `<YOUR-PRE-SHARED-KEY>`).
   Make sure it actually **exists** in the Pre-Shared Keys list — see the trap below.
3. **Remote authentication ID = `10.10.0.1`** (the PEER's address, *not* fw-b's own).
4. **Phase 2 / Child:** Local network = `10.10.30.0/24`; Remote network = `10.10.10.0/24`;
   ESP with AES-GCM.
5. Enable IPsec and **Apply**.

### 3b. fw-a side — push a swanctl config over SSH (reliable)

On your desktop, write the two files, then copy them in with `cat | ssh` (csh-safe):

`tunnel.conf`:
```
connections {
  labtun {
    version = 2
    local_addrs = 10.10.0.1
    remote_addrs = 10.10.0.2
    proposals = default
    local  { auth = psk }
    remote { auth = psk }
    children {
      labchild {
        local_ts  = 10.10.10.0/24
        remote_ts = 10.10.30.0/24
        esp_proposals = default
      }
    }
  }
}
```
`labsecret.conf`:
```
secrets {
  ike-lab {
    secret = "<YOUR-PRE-SHARED-KEY>"
  }
}
```
Copy them onto fw-a and load:
```
cat tunnel.conf     | ssh fw-a 'cat > /usr/local/etc/swanctl/conf.d/tunnel.conf'
cat labsecret.conf  | ssh fw-a 'cat > /usr/local/etc/swanctl/conf.d/labsecret.conf'
ssh fw-a 'swanctl --load-all'
ssh fw-a 'swanctl --initiate --child labchild'
```
> Note: `labtun` sets **no explicit identity**, so each end's identity defaults to its
> address. That deliberately sidesteps the mirrored-ID trap below.

### The identity traps (these cost hours — check them first if it won't come up)

- **No PSK secret actually exists.** The connection says "auth psk" but the Pre-Shared
  Keys list is empty → log says `no shared key found`. Create the secret explicitly.
- **Mirrored remote-ID.** If each firewall sets **both** its local and remote auth ID to
  its *own* address, each rejects the other → `AUTHENTICATION_FAILED`. The **remote ID must
  be the peer's address** (fw-a→`10.10.0.2`, fw-b→`10.10.0.1`), or leave it unset.

**Verify (gate):** `ssh fw-a 'swanctl --list-sas'`
→ `labtun ... ESTABLISHED, IKEv2` and `labchild ... INSTALLED, TUNNEL, ESP:AES_GCM`.
(No `--list-secrets` / `--unload-conn` on this version.)

---

## PART 4 — Make the tunnelled traffic actually flow

A tunnel that authenticates is **not** the same as traffic that flows. Two more things:

### 4a. Permit ESP on the WAN (both firewalls)

The auto-generated IPsec WAN rules can reference an old peer address and can't be edited
(link-icon only), so fw-a may silently drop inbound ESP. On **each** firewall GUI →
**Firewall → Rules → WAN**, add a **pass** rule:
- Action **Pass**, Protocol **any**, Source **WAN net** (= `10.10.0.0/24`), Dest **any**.

This is where "**ESP is protocol 50, not a port**" matters — a port rule never matches it.

### 4b. Routes to the far LAN (both hosts)

Each end host must send far-site traffic to its own firewall. With the persistent netplan
(already baked into the `phase4-vpn` snapshot) this is automatic; to set it by hand:
```
# branch attacker:
ssh attacker 'sudo ip route replace 10.10.10.0/24 via 10.10.30.1'
# Site A target:
ssh throwaway 'sudo ip route replace 10.10.30.0/24 via 10.10.10.1'
```

> **Debug tip:** if the tunnel is up but nothing crosses, read the firewall **drop** log,
> not the IPsec log: `ssh fw-a 'tcpdump -nei pflog0'` — a line like
> `block in on em1 ... ESP` names the rule that's dropping it.

**Verify (gate):** `ssh attacker 'ping -c4 10.10.10.50'`
→ 4 replies with **TTL 62** (64 minus two firewall hops = it rode the tunnel).

---

## PART 5 — Place the eye at the decryption point

No new detection content is needed — that's the lesson. The sensor's existing
`/var/lib/suricata/rules/local.rules` (the tuned **LOCAL NMAP** rules and the **LAB LFI**
signature `sid:9000010` from Phase 3) fire on the **decrypted** traffic, because the sensor
taps the **Site A LAN** (`enp0s10`) — where fw-a hands the packets back in the clear — not
the encrypted transit.

**Verify (gate):** `ssh sensor 'systemctl is-active suricata'` → `active`, and
`ssh sensor 'sudo grep -c LOCAL /var/lib/suricata/rules/local.rules'` → non-zero.

---

## PART 6 — Run the tests (this is the demo)

### Test 1 — connectivity through the tunnel
```
ssh attacker 'ping -c4 10.10.10.50'
```
PASS = 4/4, **TTL 62**.

### Test 2 — nmap over the VPN, caught at the decryption point
```
ssh attacker 'sudo nmap -sS -T4 --top-ports 200 10.10.10.50'
ssh sensor  "sudo tail -n 300 /var/log/suricata/eve.json | jq -Rrc 'fromjson? | select(.event_type==\"alert\") | \"\(.src_ip) -> \(.dest_ip) | \(.alert.signature)\"' | sort -u"
```
PASS = `LOCAL NMAP` SYN scan / connect sweep / ICMP sweep, all **src `10.10.30.40`**
(the real branch attacker — attribution survives the tunnel).

### Test 3 — LFI over the VPN, caught by the Phase-3 signature
First give the target a web service (runtime-only, start it each session):
```
ssh throwaway 'nohup sudo python3 -m http.server 80 >/tmp/web.log 2>&1 &'
```
Fire LFI requests (Nikto's generic probes do **not** match — the rule looks for the URI
string `etc/passwd`, so send it explicitly):
```
ssh attacker 'for p in "download.php?file=../../../../etc/passwd" "?file=/etc/passwd" "view.php?doc=/etc/passwd"; do curl -s -o /dev/null "http://10.10.10.50/$p"; done'
ssh sensor "sudo tail -n 400 /var/log/suricata/eve.json | jq -Rrc 'fromjson? | select(.alert.signature|test(\"LFI\")) | \"\(.src_ip) -> \(.dest_ip):\(.dest_port) | \(.alert.signature)\"' | sort | uniq -c"
```
PASS = `LAB LFI attempt - /etc/passwd requested in URL`, **src `10.10.30.40`**.
(You can also run `nikto -h http://10.10.10.50` for realism; add the curls to guarantee the
LFI signature fires.)

### Test 4 — blind on the wire (confidentiality proof)
Capture the transit while the scan runs:
```
ssh sensor 'sudo timeout 12 tcpdump -nni enp0s8 -c 12 esp > /tmp/transit.txt 2>&1 &'
ssh attacker 'sudo nmap -sS -T4 --top-ports 50 10.10.10.50'
ssh sensor 'cat /tmp/transit.txt'
```
PASS = only `ESP(spi=...) 10.10.0.2 > 10.10.0.1` (and back). **No ports, no flags, no
`10.10.30.40`, no scan pattern.** The man-in-the-middle sees nothing but encrypted blobs.

### Test 5 — correlation (the payoff)
Line up the two witnesses by timestamp:
- **Transit tap** at time T → an encrypted surge `fw-b 10.10.0.2 ⇄ fw-a 10.10.0.1` (when + how much).
- **Site A sensor** at time T → `10.10.30.40` scanning `10.10.10.50` (who + what).

Neither alone tells the story; aligned, the encrypted surge **is** the scan the sensor
decoded → the branch attacker hidden in the tunnel is **unmasked**.

---

## PART 7 — Capture the report evidence (10 insert fields)

For `Phase-4-Report.docx`, grab these screenshots as you go:
1. Topology diagram · 2. Routed-vs-tunnelled packet · 3. `swanctl --list-sas` ESTABLISHED
· 4. Two-places-to-watch diagram · 5. swanctl `tunnel.conf` · 6. WAN pass rule · 7. nmap
alert (src .30.40) · 8. LFI alert (src .30.40) · 9. transit ESP-only capture · 10.
correlation timeline.

---

## PART 8 — Snapshot discipline

- **Cold snapshots only.** Live snapshots on this lab have hung and failed to persist.
  Power the VMs **off** first (`VBoxManage controlvm <vm> acpipowerbutton`), wait until
  `VBoxManage list runningvms` is empty, then `VBoxManage snapshot <vm> take phase4-vpn`.
- The **`phase4-vpn`** snapshot (8 VMs: fw-a, fw-b, vyos, sensor, wazuh, grafana, attacker,
  throwaway) already holds this exact working state, with the branch IP/routes persistent.
  Restore it and the tunnel auto-establishes on boot.

---

## Quick reference

| Thing | Value |
|---|---|
| Tunnel endpoints | fw-a `10.10.0.1` ⇄ fw-b `10.10.0.2` |
| Protected nets | `10.10.10.0/24` (A) ⇄ `10.10.30.0/24` (B) |
| Crypto | IKEv2 · AES-GCM · SHA-256 · PSK |
| Branch attacker | `10.10.30.40` (`ssh attacker`) |
| Site A target | `10.10.10.50` (`ssh throwaway`) |
| Decryption-point tap | sensor `enp0s10` (Site A LAN) |
| Transit tap (ESP only) | sensor `enp0s8` |
| Tunnel state check | `ssh fw-a 'swanctl --list-sas'` |
| Detection log | `ssh sensor 'sudo tail /var/log/suricata/eve.json'` |

**The phase in one line:** *encryption blinds the middle; the decryption point restores
sight; correlation names the attacker.*
