# FreePBX Setup on Debian 13 (Trixie) + Tailscale

I'll do production-ready FreePBX setup on Debian 13 "Trixie", with Tailscale for secure admin and remote-agent access. Covers VoIP call termination, call center and general PBX use.

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

## 2. DM for setup support

- 💬 [Telegram](https://t.me/AnythingLinux)
- 📩 [WhatsApp](https://wa.me/8801890757616)
- ▶️ [Demo](https://youtu.be/wI6sLyb2ZQw)

## 3. Requirements

### 3.1 Server sizing

| Use case | vCPU | RAM | Disk | Concurrent calls |
|---|---|---|---|---|
| Small office (≤25 ext) | 2 | 4 GB | 60 GB SSD | ≤20 |
| Call center (25–100 agents) | 4–8 | 8–16 GB | 200 GB SSD | 50–150 |
| Termination / high volume | 8+ | 16–32 GB | 250 GB+ SSD | 200+ (no transcoding) |

- Bare metal or VM (KVM/VMware/Proxmox). Avoid oversubscribed VPS for production.
- Add disk for call recordings (≈ 0.5 MB/min per call in WAV; use MP3/compression).

### 3.2 Software
- Debian 13 (Trixie) minimal, 64-bit, fresh install
- FreePBX 17 + Asterisk 21/22 LTS
- MariaDB, Apache, PHP (installed by FreePBX script)
- Tailscale (latest)

### 3.3 Network
- Static public IP (or 1:1 NAT) and static LAN IP
- Reverse DNS + FQDN (e.g. `pbx.example.com`)
- Time sync (chrony/NTP), correct timezone
- Bandwidth per call: **~87 kbps** (G.711) / **~31 kbps** (G.729) each direction. Plan 100 kbps × peak calls, plus 25% headroom.
- Latency < 150 ms, jitter < 30 ms, packet loss < 1%

### 3.4 Ports

| Port | Proto | Purpose | Expose to |
|---|---|---|---|
| 5060 / 5061 | UDP/TCP / TLS | PJSIP signaling | Carrier IPs only |
| 10000–20000 | UDP | RTP media | Carrier IPs / Tailscale |
| 443 | TCP | FreePBX GUI | Tailscale only |
| 22 | TCP | SSH | Tailscale only |
| 41641 | UDP | Tailscale | Outbound/any |

### 3.5 Accounts & info to collect from client
- SIP carrier details: host, auth (IP or user/pass), codecs, max channels, DIDs
- Caller ID rules, emergency (E911) address registration
- Number of extensions, agents, queues, IVR menus, business hours
- Recording/retention and compliance needs (GDPR, PCI, call-recording consent)
- Tailscale tailnet admin access

---

## References
- [FreePBX docs](https://sangomakb.atlassian.net/wiki/spaces/PG/overview)
- [Tailscale docs](https://tailscale.com/kb)
- [Asterisk docs](https://docs.asterisk.org)
- [Credit Sangoma](https://sangoma.com)
