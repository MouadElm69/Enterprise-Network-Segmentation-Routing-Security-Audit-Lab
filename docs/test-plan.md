# Test plan and results

**English** | [Français](test-plan_FR.md)

Each test was run from a PC's command prompt (or the web browser for HTTP). For the blocked cases I also looked at the ACL hit counters on R1.

| # | From -> to | Test | Expected | What I saw | Evidence |
|---|---|---|---|---|---|
| 1 | Guest -> Finance | `ping 192.168.30.2` | Blocked | 4 of 4 lost, "Destination host unreachable" from 192.168.50.1 | `test-01-02-guest-blocked.png` |
| 2 | Guest -> Servers | `ping 192.168.40.2` | Blocked | 4 of 4 lost, unreachable from 192.168.50.1 | `test-01-02-guest-blocked.png` |
| 3 | Guest -> internet | `ping 203.0.113.1` | Allowed | 4 of 4 replies, TTL 254. NAT table shows 192.168.50.2 -> 203.0.113.2 | `test-03-guest-internet-ping.png`, `test-03-nat-translations.png` |
| 4 | HR -> Finance | `ping 192.168.30.2` | Blocked | 4 of 4 lost, unreachable from 192.168.20.1 | `test-04-hr-to-finance-blocked.png` |
| 5 | HR -> Servers | browser, `http://192.168.40.2` | Allowed | Page loads. ACL 120 `eq www` line: 5 matches | `test-05-hr-browser-http.png`, `acl120-counters.png` |
| 6 | HR -> Servers | `ping 192.168.40.2` and `.40.3` | Blocked | 8 of 8 lost, unreachable from 192.168.20.1. ACL 120 deny line: 8 matches | `test-06-hr-to-servers-blocked.png`, `acl120-counters.png` |
| 7 | IT -> R1 | `ssh -l admin 192.168.10.1` | Allowed | Password prompt, then the `R1#` prompt | `test-07-it-ssh-success.png` |
| 8 | HR -> R1 | `ssh -l admin 192.168.20.1` | Blocked | "Connection refused by remote host". ACL 10 `deny any`: 5 matches | `test-08-hr-ssh-refused.png`, `test-08-acl10-counters.png` |
| 9 | IT -> Finance | `ping 192.168.30.2` | Allowed | 3 of 4 replies (the first packet is lost to ARP) | `test-09-10-it-ping-ipconfig.png` |
| 10 | One PC per VLAN | `ipconfig` | Address from its own VLAN | See the table below | `test-09-10-it-ping-ipconfig.png` |

The first ping in a test often times out because the PC still has to resolve the gateway with ARP. Only the later packets count.

## Addresses handed out by DHCP

| VLAN | Host | Address | Gateway |
|---|---|---|---|
| 10 IT | IT PC | 192.168.10.3 | 192.168.10.1 |
| 20 HR | HR PC | 192.168.20.2 | 192.168.20.1 |
| 30 Finance | Finance PC | 192.168.30.3 | 192.168.30.1 |
| 40 Servers | Servers PC | 192.168.40.3 | 192.168.40.1 |
| 50 Guest | Guest PC | 192.168.50.2 | 192.168.50.1 |

## Other checks

- RSA keys exist on R1: `design-rsa-keys-r1.png` (`show crypto key mypubkey rsa`).
- The web server has HTTP turned on: `design-server-http-service.png`.

## A result I had to redo

My first run of test 6 showed replies from 192.168.40.3, which should have been blocked. I cleared the counters, ran both pings again in one session, and all 8 were blocked and counted. I think the first run happened before the ACL was applied, but I did not confirm that.
