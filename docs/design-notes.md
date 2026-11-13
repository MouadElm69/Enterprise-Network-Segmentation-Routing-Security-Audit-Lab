# Design notes

**English** | [Français](design-notes_FR.md)

## Router-on-a-stick

All five VLANs share one physical link between R1 (`Gi0/0`) and the switch (`Gi0/1`), carried as an 802.1Q trunk. R1 has one sub-interface per VLAN, tagged with the VLAN ID, and each one is the default gateway for that VLAN. It is cheap and fine for a site this size. The downside is that all traffic between VLANs goes through that single link, so a Layer 3 switch would be the next step in a bigger network.

## The ACLs

These are extended ACLs and they are stateless: each packet is judged on its own and the router does not remember connections. So direction matters. `in` filters packets arriving on an interface, `out` filters packets leaving it.

### ACL 100: guests (`Gi0/0.50`, in)

```
access-list 100 deny ip 192.168.50.0 0.0.0.255 192.168.0.0 0.0.255.255
access-list 100 permit ip 192.168.50.0 0.0.0.255 any
```

This sits as close to the guests as possible, so traffic to any internal network is dropped before it is routed. Everything else goes through, which in this lab means the internet. One side effect: the guest gateway (192.168.50.1) is inside 192.168.0.0/16 too, so guests probably can't ping their own gateway either (I did not test that).

### ACL 110: Finance (`Gi0/0.30`, out)

```
access-list 110 permit ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
access-list 110 deny ip 192.168.0.0 0.0.255.255 192.168.30.0 0.0.0.255
access-list 110 permit ip any any
```

Only IT may send traffic to Finance. Every other internal source is denied. Because the ACL is on the way out toward Finance, it protects traffic going to Finance. Finance hosts can still start connections toward other VLANs. For most of them the replies are dropped on the way back, because only IT is allowed through, which is the stateless behaviour showing up. The last line lets internet return traffic through.

### ACL 120: servers (`Gi0/0.40`, out)

```
access-list 120 permit tcp any 192.168.40.0 0.0.0.255 eq www
access-list 120 permit udp any 192.168.40.0 0.0.0.255 eq domain
access-list 120 deny ip any 192.168.40.0 0.0.0.255
access-list 120 permit ip any any
```

Only HTTP and DNS reach the server VLAN. Everything else, including ping and SSH, is dropped. Two limits: HTTPS (443) is not allowed even though the server has it turned on, and IT staff can't reach the servers for management either. An IT-only permit above the deny would fix the second one.

### Remote access to R1 (ACL 10)

```
access-list 10 permit 192.168.10.0 0.0.0.255
access-list 10 deny any
line vty 0 4
 access-class 10 in
 login local
 transport input ssh
```

`access-class` filters connections to the router itself, which the interface ACLs don't do. The `deny any` is there on purpose: the implicit deny at the end of an ACL doesn't count hits, an explicit one does, so a blocked attempt shows up in `show access-lists 10`.

## Scope

I kept management security simple: SSH version 2, a local user, RSA keys and `access-class`. I did not set up AAA servers. CORE-SW has no management configuration at all, which I documented as a gap instead of leaving half-finished settings that look like protection.

## Problems and fixes

### Stateless ACLs and direction

The part that took me longest to get was that ACLs are stateless and that `in`/`out` is relative to the router. That is why ACL 100 is inbound on the guest side and ACL 110 and 120 are outbound toward Finance and the servers.

### Config on the wrong router

I created the `.40` sub-interface and ACL 120 on the ISP router by mistake. The cleanup:

```
Router(config)#interface gigabitEthernet 0/0.40
Router(config-subif)#no ip access-group 120 in
Router(config-subif)#exit
Router(config)#no access-list 120
Router(config)#no interface GigabitEthernet0/0.40
Router#copy running-config startup-config
```

The final ISP router config has no sub-interfaces and no ACLs (`configs/isp-router.cfg`). Now every device has its own hostname, so the prompt tells me where I am.

### SSH closed by the remote host

```
C:\>ssh -l admin 192.168.10.1
[Connection to 192.168.10.1 closed by foreign host]
```

Reading the config showed `transport input ssh` and a local user, but no domain name and no RSA keys, so the router could not accept SSH. After generating the keys (next section), the SSH test from the IT PC worked (`screenshots/test-07-it-ssh-success.png`).

### RSA keys refused

Generating the keys failed twice. This IOS version rejected the `modulus` keyword, and then it refused to generate keys while the hostname was still `Router`:

```
Router(config)#ip domain-name lab.local
Router(config)#crypto key generate rsa modulus 2048
% Invalid input detected at '^' marker.
Router(config)#crypto key generate rsa general-keys
% Please define a hostname other than Router.
Router(config)#hostname R1
R1(config)#crypto key generate rsa general-keys
The name for the keys will be: R1.lab.local
How many bits in the modulus [512]: 2048
% Generating 2048 bit RSA keys, keys will be non-exportable...[OK]
```

The key name is built from the hostname and the domain name, so both have to be set first. The order is hostname, domain name, keys, then SSH. Screenshot: `screenshots/incidents/incident-rsa-hostname-error.png`.

### NAT marked on the wrong interface

I had `ip nat inside` on `Gi0/0`. Traffic from the VLANs comes in through the sub-interfaces, so the marking on the physical interface did nothing for it. I spotted this while reading the exported config, not from a failed test. The fix:

```
R1(config)#interface gigabitEthernet 0/0
R1(config-if)#no ip nat inside
R1(config)#interface gigabitEthernet 0/0.10
R1(config-subif)#ip nat inside
! same on 0/0.20, 0/0.30, 0/0.40 and 0/0.50
```

After that, a guest ping to the ISP router showed up in `show ip nat translations` as 192.168.50.2 translated to 203.0.113.2 (`screenshots/test-03-nat-translations.png`). Screenshot of the fix: `screenshots/incidents/incident-nat-subinterface-fix.png`.
