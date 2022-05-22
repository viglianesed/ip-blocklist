# IP Blocklist

![IP Blocklist](./doc/img/logo.png)

<!-- <img src="./doc/img/logo.png"> -->

The intent of this project is to collect IP addresses of port scanners.

All IP addresses are not aggregated, this is to allow the individual IP addresses to be later removed thanks to the timeout. 

My server's traffic is routed via a VPN provider, so all traffic directed to my public IP address can be considered an attack or a port scan.

Excluding the ports opened for the services running on the server. 

---
## Nftables setup

The nftables that I'm running is:
```
nftables v0.9.0 (Fearless Fosdick)
```

Nftables manual show an option to update elements of a set, but it is not changing the timeout for me.
```
update @named_set {ip saddr timeout 20d}
```
You can manually remove an entry, but it can't be done dynamically by the firewall
```
nft delete element inet firewall named_set { IP_ADDRESS }
```
Also re-adding the entry won't work as it's already present in the set.


The workaround that I found was to create a second set in order to properly reset the timeout with an external script that runs every hour.

---
## Firewall rules


The firewall rules with nftables looks like this:
```
# List of IPv4 addresses that are dynamically blocked
set blackhole {
   type ipv4_addr
   flags timeout
}
set blackhole_2 {
   type ipv4_addr
   flags timeout
}

# Accept DNS responses to avoid being filtered later
ip saddr @DNS_servers udp sport 53 udp dport >= 1024 accept

# Block returning IP addresses
iif $WAN_NIC ip saddr @blackhole_2 counter drop

# Log the returning attacks
iif $WAN_NIC ip saddr @blackhole ip saddr != @blackhole_2 log prefix "Update ban time: " add @blackhole_2 {ip saddr timeout 3h} counter

# Drop the traffic from the blocklist
iif $WAN_NIC ip saddr @blackhole drop

# Mark all traffic from non trusted sources
iif $WAN_NIC ip daddr $WAN_IP ip saddr != @whitelistedPublicIP ip saddr != @VPNProvider mark set $BAN02_MARK

mark $BAN02_MARK tcp dport < 1024 tcp dport != @OpenTCPports mark set $BAN01_MARK
mark $BAN02_MARK udp dport < 1024 udp dport != @OpenUDPports mark set $BAN01_MARK
mark $BAN02_MARK ct state new tcp dport != @OpenTCPports mark set $BAN01_MARK
mark $BAN02_MARK ct state new udp dport != @OpenUDPports mark set $BAN01_MARK

# Add to blocklist
mark $BAN01_MARK counter log prefix "New invalid traffic: "
mark $BAN01_MARK counter add @blackhole {ip saddr timeout 20d} drop

```
---
## When is the list updated?

All entries are extracted from */var/log/kern.log* and are pushed to GitHub every hour.

Same entries are also pushed to [AbuseIPDB](https://www.abuseipdb.com/user/44008) once a day.

| Destination | Time    |
| ----------- | ------- |
| Github      | ≈ xx:20 |
| AbuseIPDB   | ≈ 23:59 |


---
## Blocklist files


I provide the entries in different formats.

set_blackhole_with_timeout.nft
```
#!/usr/sbin/nft -f
flush set inet firewall blackhole
add element inet firewall blackhole{
   123.123.123.123 timeout 1234567890s,
   234.234.234.234 timeout 1234567890s,
   ...
}
```
set_blackhole.nft
```
#!/usr/sbin/nft -f
flush set inet firewall blackhole
add element inet firewall blackhole{
   123.123.123.123,
   234.234.234.234,
   ...
}
```
hosts.deny
```
ALL: 123.123.123.123
ALL: 234.234.234.234
...
```

hosts.txt
```
123.123.123.123
234.234.234.234
...
```
---


So far 14488 addresses are stored in the blocklist.

The highest number of stored IP addresses is: 24963

---
## Donate:
<!-- Click on the pictures to donate!<br> -->
<!-- *GithHub does not allow custom URIs so there is no link for Monero :(* -->

| Monero |
| ------ |
| <img src="https://chart.apis.google.com/chart?cht=qr&chs=200x200&chl=monero%3A83navbX4c3ff2Jynb182zriQXHUs3dJt2LwA4rw2m162PqTJsgoGaHkY2RR3YEeCHtUcoaPLuu25gbp1UDKvnG6eLwduvYr%3Ftx_description%3DDonation%26recipient_name%3DDavide%2520Viglianese&&choe=UTF-8&chld=L" width="200" height="200" alt="qr code"> |

Monero: `83navbX4c3ff2Jynb182zriQXHUs3dJt2LwA4rw2m162PqTJsgoGaHkY2RR3YEeCHtUcoaPLuu25gbp1UDKvnG6eLwduvYr`
