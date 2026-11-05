# VoIP System: FreePBX 17 on Debian 13 (Trixie) + Tailscale

Client-focused guide to deploy, secure, and test a VoIP platform for **call termination**, **call center**, and general PBX use.

> **Verify first:** FreePBX 17 officially targets Debian 12. Confirm Debian 13 support in the current FreePBX installer/release notes before production use. Test on a staging VM first.

---

## 1. Architecture

```
 Carrier / SIP Provider  <--public IP-->  [ FreePBX 17 / Asterisk ]  <--Tailscale-->  Admins, remote agents, softphones
                                                  |
                                           LAN IP phones / agents
```

- **Public IP:** carrier trunks (SIP/RTP) only.
- **Tailscale:** admin GUI/SSH, remote agents, site-to-site. Never expose the GUI publicly.

---

## 2. Requirements

### 2.1 Server sizing

| Use case | vCPU | RAM | Disk | Concurrent calls |
|---|---|---|---|---|
| Small office (≤25 ext) | 2 | 4 GB | 60 GB SSD | ≤20 |
| Call center (25–100 agents) | 4–8 | 8–16 GB | 200 GB SSD | 50–150 |
| Termination / high volume | 8+ | 16–32 GB | 250 GB+ SSD | 200+ (no transcoding) |

- Bare metal or VM (KVM/VMware/Proxmox). Avoid oversubscribed VPS for production.
- Add disk for call recordings (≈ 0.5 MB/min per call in WAV; use MP3/compression).

### 2.2 Software
- Debian 13 (Trixie) minimal, 64-bit, fresh install
- FreePBX 17 + Asterisk 21/22 LTS
- MariaDB, Apache, PHP (installed by FreePBX script)
- Tailscale (latest)

### 2.3 Network
- Static public IP (or 1:1 NAT) and static LAN IP
- Reverse DNS + FQDN (e.g. `pbx.example.com`)
- Time sync (chrony/NTP), correct timezone
- Bandwidth per call: **~87 kbps** (G.711) / **~31 kbps** (G.729) each direction. Plan 100 kbps × peak calls, plus 25% headroom.
- Latency < 150 ms, jitter < 30 ms, packet loss < 1%

### 2.4 Ports

| Port | Proto | Purpose | Expose to |
|---|---|---|---|
| 5060 / 5061 | UDP/TCP / TLS | PJSIP signaling | Carrier IPs only |
| 10000–20000 | UDP | RTP media | Carrier IPs / Tailscale |
| 443 | TCP | FreePBX GUI | Tailscale only |
| 22 | TCP | SSH | Tailscale only |
| 41641 | UDP | Tailscale | Outbound/any |

### 2.5 Accounts & info to collect from client
- SIP carrier details: host, auth (IP or user/pass), codecs, max channels, DIDs
- Caller ID rules, emergency (E911) address registration
- Number of extensions, agents, queues, IVR menus, business hours
- Recording/retention and compliance needs (GDPR, PCI, call-recording consent)
- Tailscale tailnet admin access

---

## 3. Base OS Setup (Debian 13)

```bash
apt update && apt full-upgrade -y
hostnamectl set-hostname pbx.example.com
timedatectl set-timezone <Region/City>
apt install -y curl wget gnupg ca-certificates chrony ufw fail2ban sudo
reboot
```

---

## 4. Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --ssh --hostname=pbx
tailscale ip -4
```

- Authenticate via the printed URL; approve the node in the admin console.
- Disable key expiry for the server node.
- Use **ACLs** to limit who reaches ports 443/22/SIP.
- Optional: advertise subnet routes (`--advertise-routes=10.0.0.0/24`) for site-to-site.

---

## 5. Install FreePBX 17

```bash
cd /usr/src
wget https://github.com/FreePBX/sng_freepbx_debian_install/raw/master/sng_freepbx_debian_install.sh
bash sng_freepbx_debian_install.sh
```

Then open `https://<tailscale-ip>` and:
1. Create admin user
2. Activate/enable needed modules (Admin → Module Admin → Check Online → Upgrade All)
3. Set **Settings → Asterisk SIP Settings**:
   - External Address: public IP
   - Local Networks: LAN subnet **and** `100.64.0.0/10` (Tailscale)
   - RTP range: 10000–20000
4. Apply Config

---

## 6. Firewall (UFW example)

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow in on tailscale0                        # admin, remote agents
ufw allow from <CARRIER_IP> to any port 5060 proto udp
ufw allow from <CARRIER_IP> to any port 10000:20000 proto udp
ufw allow from <LAN_SUBNET>                        # local phones
ufw enable
```

Also enable FreePBX **Firewall** module + **Intrusion Detection (Fail2ban)**. Set responsive/trusted zones accordingly.

---

## 7. Call Termination Setup

1. **Connectivity → Trunks → Add PJSIP Trunk**
   - Auth: IP-based (preferred) or username/password
   - Codecs: ulaw, alaw (add g729 only if licensed)
   - Max channels: match carrier limit
   - Set qualify/keepalive
2. **Outbound Routes:** dial patterns (e.g. `NXXNXXXXXX`, `+.`), trunk priority, failover trunk
3. **Inbound Routes:** map DIDs → extensions / queues / IVR
4. **Caller ID:** set per trunk/route; must be a number you own (STIR/SHAKEN attestation is handled by carrier)
5. **Termination/wholesale (if reselling):**
   - Separate trunk per customer/carrier with IP ACLs
   - Limit concurrent calls and CPS (calls per second)
   - Route by prefix; use least-cost routing and failover
   - Enable CDR export/billing; block high-risk destinations
   - Use no-transcoding: same codec end to end
6. **Fraud controls:** restrict international dialing by extension class, set max call duration, alerts on traffic spikes.

---

## 8. Call Center Setup

### 8.1 Components
- **Queues** (Applications → Queues): ring strategy (`rrmemory`, `leastrecent`, `fewestcalls`), max wait, announcements, join-empty rules
- **Agents:** dynamic login via feature codes (`*45` toggle) or static members; use **Queue Pause** with reasons
- **IVR** (Applications → IVR): business-hours and after-hours flows via **Time Conditions**
- **Ring Groups, Voicemail, Call Flow Control** for overflow
- **Call Recording:** per queue/extension; store on separate volume; retention policy
- **Reporting:** Queue Stats / **Asterisk CDR** / Call Event Logging; optional wallboard (e.g. Queue Metrics, Grafana)
- **Supervisor tools:** ChanSpy (listen/whisper/barge) via feature codes
- **Optional:** CRM integration (REST/AMI/ARI), click-to-call, predictive dialer (outbound campaigns need legal/DNC compliance)

### 8.2 Remote agents over Tailscale
- Install Tailscale on agent PC/phone, then register softphone (Zoiper, Linphone, MicroSIP) to the **Tailscale IP** of the PBX
- Use PJSIP extension; set NAT = No (traffic is on tailnet), transport = UDP/TLS
- Prefer headsets with wired connection; QoS on agent LAN

### 8.3 Sizing rule
Agents needed ≈ (calls/hour × avg handle time in hrs) ÷ target occupancy (0.8), plus shrinkage 25–30%.

---

## 9. Other Applications (Optional)

- Conferencing (ConfBridge), Paging/Intercom, Follow-Me, Fax (T.38)
- Voicemail-to-email, SMS/WhatsApp gateways via carrier APIs
- Provisioning (EndPoint Manager) for Yealink/Poly/Grandstream phones
- WebRTC softphone (UCP) for browser-based calling
- High availability: warm standby + shared storage / DB replication

---

## 10. Security Checklist

- [ ] GUI and SSH reachable **only via Tailscale**
- [ ] Strong, unique extension secrets (16+ chars); disable guest/anonymous SIP
- [ ] Carrier IP allow-list on 5060 and RTP
- [ ] Fail2ban + FreePBX Firewall enabled
- [ ] TLS/SRTP for remote endpoints where supported
- [ ] International dialing restricted by default
- [ ] Automatic security updates; patch FreePBX modules monthly
- [ ] Daily backup (Admin → Backup & Restore) copied off-server
- [ ] Tailscale ACLs reviewed; MFA on tailnet identity provider

---

## 11. Testing Plan

| # | Test | Expected |
|---|---|---|
| 1 | `tailscale status`, ping PBX from admin PC | Reachable, <50 ms |
| 2 | GUI over Tailscale; public IP:443 blocked | GUI loads only via tailnet |
| 3 | Register 2 extensions (LAN + Tailscale) | Registered, `pjsip show endpoints` OK |
| 4 | Ext → Ext call, both directions | Two-way audio, no one-way |
| 5 | Inbound DID → IVR/queue | Correct routing, audio OK |
| 6 | Outbound local + international call | Completes, correct caller ID |
| 7 | Trunk failover (disable primary) | Calls use backup trunk |
| 8 | Queue: 3 agents, 5 callers | Strategy and hold music work; stats logged |
| 9 | Pause/unpause, supervisor listen/barge | Works per feature code |
| 10 | Recording playback | File created, playable |
| 11 | DTMF (IVR, transfers) | RFC 2833 recognized |
| 12 | Call transfer (attended/blind), hold, voicemail | Functions correctly |
| 13 | Load test (e.g. SIPp/sipp-ng) at target CC & CPS | CPU < 70%, no drops |
| 14 | Network: MOS ≥ 4.0, jitter < 30 ms | Pass |
| 15 | Fail2ban: 5 bad SIP auths | IP banned |
| 16 | Backup + restore to staging VM | Restores cleanly |
| 17 | E911 test call (per provider procedure) | Correct address delivered |

### Useful commands
```bash
asterisk -rvvv                                  # CLI
pjsip show endpoints | contacts | registrations
core show channels
queue show
pjsip set logger on                             # SIP trace
tcpdump -i any -n port 5060 -w sip.pcap         # packet capture
fwconsole reload | ma list | restart
journalctl -u asterisk -f
tailscale status | tailscale netcheck
```

---

## 12. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| One-way / no audio | NAT: set External Address, Local Nets, open RTP range; check Tailscale subnet in Local Nets |
| Registration fails | Wrong secret/transport; Fail2ban ban; ACL; check `pjsip set logger on` |
| Outbound 403/404/503 | Trunk auth/IP not whitelisted by carrier; wrong dial pattern |
| Choppy audio | Bandwidth, jitter, transcoding; use same codec; QoS |
| Calls drop at ~30 s | SIP ALG on router (disable), missing ACK/NAT |
| GUI unreachable | Check `tailscale status`, UFW rules, Apache status |
| Queue agents not ringing | Agent not logged in/paused, wrong member interface, strategy/timeout |

---

## 13. Go-Live Checklist

- [ ] Carrier ports/DIDs pointed to PBX (schedule porting window)
- [ ] Phones provisioned; agents trained (login, pause, transfer)
- [ ] Monitoring/alerts (Zabbix/Netdata/Uptime Kuma) on CPU, disk, trunk status
- [ ] Backup schedule + offsite copy verified
- [ ] Documentation: IP/DNS, credentials vault, call flow diagram
- [ ] Rollback plan (keep old system 48–72 h)
- [ ] Client sign-off on test table (Section 11)

---

## 14. Maintenance

- **Weekly:** review CDR anomalies, disk usage, fail2ban logs
- **Monthly:** `apt upgrade`, `fwconsole ma upgradeall`, test restore
- **Quarterly:** review ACLs, rotate secrets, capacity review

---

## References
- FreePBX docs: https://sangomakb.atlassian.net/wiki/spaces/PG/overview
- FreePBX install script repo: https://github.com/FreePBX/sng_freepbx_debian_install
- Tailscale docs: https://tailscale.com/kb
- Asterisk docs: https://docs.asterisk.org
