Here is the **Airlock Room Function & Flow Table** — a clean, step-by-step breakdown of the `net-airlock` (cyberspace-airlock) room in Anchor OS v1.5.

This is the single privileged room that mediates all external network access. It is the choke point: no room touches the internet directly. All traffic is routed through the airlock.

| Stage | What Happens | Trigger / Actor | Key Components Involved | Direction / Flow | User Visibility / Consent | Security Guarantee / Enforcement | Time / Overhead |
|-------|--------------|-----------------|--------------------------|------------------|---------------------------|-----------------------------------|-----------------|
| 1. Airlock Spawn | Bay0 creates the airlock room as the first/only WAN-capable room (if any room requests network) | System boot or first room with `network != none` | bay0 (PID 1), passport (net-airlock=true) | Host → airlock | Visible indicator: “Network airlock active” on system bar | Fixed passport: wan-only/full only for airlock; other rooms default none | ~0.5–1 s |
| 2. veth Pair Creation | bay0 creates virtual Ethernet pair: one end in airlock namespace, one in host namespace | Airlock room spawn | bay0 + Linux netns + veth driver | Host ↔ airlock | None (transparent) | Airlock gets eth0 with IP 10.0.0.2/24; host gets gateway 10.0.0.1 | <100 ms |
| 3. Routing & NAT Setup | Host side sets IP forwarding + iptables MASQUERADE on host eth0 → airlock can reach internet | Airlock spawn (bay0) | bay0 + iptables/nftables | Airlock → Internet (outbound only) | None (transparent) | nftables forward chain: only allow from airlock veth → drop all else | <100 ms |
| 4. DNSmasq Startup (in airlock) | DNSmasq runs in airlock namespace, listens on 10.0.0.1, forwards to real upstream (8.8.8.8) | Airlock boot (/init script) | DNSmasq (in airlock) | Room → airlock → Internet | None | Blocks telemetry domains via NXDOMAIN or sink (127.0.0.1) | ~1 s |
| 5. Room Network Request | Normal room requests WAN → bay0 creates veth pair to airlock (not direct to host) | Room spawn with `network = wan-only` | bay0 + netns | Room → airlock → Internet | Indicator: “Network active (WAN-only)” in room status bar | All traffic forced through airlock; no direct host netns | ~0.5 s |
| 6. Traffic Flow | Packet from room → veth → airlock → DNSmasq (resolve) → nftables (allow/deny) → host NAT → Internet | Any network request from room | veth + nftables + DNSmasq | Room → Internet (mediated) | Visible per-room network badge | nftables drops LAN/private IPs; DNSmasq sinks telemetry (e.g., telemetry.openai.com → 127.0.0.1) | ~1–5 ms per packet |
| 7. Fake Response / Sink | Blocked request (e.g. telemetry) → airlock returns HTTP 200 OK or NXDOMAIN | Blocked domain / IP | DNSmasq + tinyhttpd (in airlock) | Internet → Room (fake) | None (app thinks it succeeded) | App believes transfer worked → no retry storm | Instant |
| 8. Airlock Kill / Teardown | Room with WAN closes → bay0 kills airlock if no other rooms need it | Last WAN room closes | bay0 + cgroup kill + veth teardown | N/A | Indicator disappears | All veth pairs torn down, iptables rules flushed, no lingering socket | 1–2 s |
| 9. Audit Logging | Every blocked/faked/allowed request logged by bay0 | Any network event | bay0 + /run/log/bay0.log | N/A | Log viewable via CLI (`anchor logs`) | Timestamp, room ID, source/dest IP/port, bytes, action (allow/sink/fake) | Negligible |

### Key Guarantees

- **No direct internet from any room** — all traffic must go through the airlock.
- **No persistent connection** — airlock dies when last WAN room closes.
- **No ambient leak** — DNS sink + fake HTTP prevents silent exfil or retry storms.
- **Visible & revocable** — network badge shows active status; kill room = instant network cutoff.
- **Low overhead** — veth + nftables + tiny DNSmasq = <1% CPU, <50 MB RAM.

This is the airlock — the single, disciplined door to the outside world.  
No room gets out without knocking.  
No knock = no exit.
