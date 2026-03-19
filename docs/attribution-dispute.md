Droidspaces' network isolation feature came from my project. Here's the proof.

I'm the author of [nspawn.sh](https://github.com/nspawn-sh/nspawn.sh). I published it on Feb 10, 2026 ([commit here](https://github.com/nspawn-sh/nspawn.sh/commit/d2000feeed7758a10f8eaf6d22c086b949b50316)) with full network namespace support on Android — bridge networking, veth pairs, NAT, and policy-based routing for VPN/WiFi/LTE. 24 days later Droidspaces v5 dropped with what the author himself calls an "identical" networking architecture, claimed it was "independently developed." I don't think that's true and I have receipts.

I originally posted this on r/androidroot where the Droidspaces author responded. The post was removed by mods for "harassment." The full thread with his responses is preserved here: [screenshot](https://raw.githubusercontent.com/nspawn-sh/nspawn.sh/main/androidroot-thread-removed.png)

On normal Linux this stuff is straightforward, you make a bridge, create a veth pair, slap a generic MASQUERADE rule on it and you're done. Android is a different beast. There's no single default route sitting in the main table. Instead netd manages separate policy routing tables for each interface, wlan0 gets its own, tun0 gets its own, rmnet_data gets its own. Traffic coming from a bridge interface doesn't match any of them so it just goes nowhere. You have to figure out which table has the active internet route and inject ip rules to push your container's subnet into that table. And you can't just flush iptables because Android's entire networking stack managed by netd depends on those rules staying in place. That's what I figured out and that's exactly what Droidspaces v4 said was too complex to do.

Droidspaces v4 had this in their docs ([commit f09d56b](https://github.com/ravindu644/Droidspaces-OSS/blob/f09d56b3aa8cae2f11799d24ce12df2c610d2c7d/Documentation/Features.md)):

> **Why Not Network Namespace?**
>
> Droidspaces deliberately does **not** use a network namespace (CLONE_NEWNET). The container shares the host's network stack. This is a design choice that greatly simplifies setup: containers get internet access immediately without virtual bridges, NAT, or firewall rules. On Android, where networking is already complex (cellular, Wi-Fi, VPN), avoiding network namespaces prevents a whole category of connectivity issues.

They explicitly said they won't do it because it's too complex on Android. And the v4 source code backs that up. Zero references to CLONE_NEWNET, veth, netns, bridge, or policy routing anywhere. The total networking code was 174 lines, mostly DNS and hostname stuff, plus this iptables function ([android.c at that commit](https://github.com/ravindu644/Droidspaces-OSS/blob/f09d56b3aa8cae2f11799d24ce12df2c610d2c7d/src/android.c)):

```c
{"iptables", "-t", "filter", "-F", NULL},
{"ip6tables", "-t", "filter", "-F", NULL},
{"iptables", "-P", "FORWARD", "ACCEPT", NULL},
{"iptables", "-t", "nat", "-A", "POSTROUTING", "-s",
 DS_DEFAULT_SUBNET, "!", "-d", DS_DEFAULT_SUBNET, "-j", "MASQUERADE", NULL},
{"iptables", "-t", "nat", "-A", "OUTPUT", "-p", "tcp",
 "-d", "127.0.0.1", "-m", "tcp", "--dport", "1:65535",
 "-j", "REDIRECT", "--to-ports", "1-65535", NULL},
```

If you know iptables you can see the problems. It flushes the entire filter chain which would break Android's networking stack since netd depends on those rules. There's a MASQUERADE rule but containers shared the host network so there's literally no traffic to masquerade. And the REDIRECT rule sends all localhost traffic on all ports back to itself which does nothing. This is not code written by someone who understands networking. Looks like LLM output shipped without testing, which makes sense because Android namespace networking was a known gap in AI training data since no working implementation existed to train on.

Then on Mar 6, Droidspaces v5 shows up and suddenly there are two brand new files, `ds_netlink.c` at 815 lines doing pure-C RTNETLINK for bridge/veth/route management and `ds_iptables.c` at 1,058 lines with raw kernel iptables socket API. `network.c` went from 174 to 734 lines with bridge setup, veth pairs, Android policy routing, upstream detection, route monitoring. 146 references to netns/veth/bridge/CLONE_NEWNET where before there were zero.

And now the iptables code has this safety contract at the top: "Never flush any chain (would kill Android tethering/hotspot). Never change any chain policy. Never touch rules we did not create. Only INSERT rules scoped to our bridge. Always check existence before inserting (fully idempotent)." That's exactly how nspawn.sh does it and the exact opposite of what v4 was doing.

**Side-by-side: what actually makes it work on Android**

Bridge and veth pairs are basic Linux stuff you can find in any tutorial. These four patterns are what make namespace networking actually work on Android. None of them exist in any tutorial, StackOverflow answer, or the PR he cited as his reference. They all exist in nspawn.sh published Feb 10, and they all appeared in Droidspaces v5 on Mar 6.

Idempotent iptables — check before insert, never flush:

nspawn.sh ([nspawn](https://github.com/nspawn-sh/nspawn.sh/blob/main/nspawn)):
```sh
add_ipt_rule() {
    $cmd $check_args 2>/dev/null || "$cmd" "$@"
}
```
Droidspaces v5 ([ds_iptables.c](https://github.com/ravindu644/Droidspaces-OSS/blob/main/src/ds_iptables.c)):
```c
rule_exists_in_hook(...)  /* check first */
insert_rule_at_hook(...)  /* then insert */
```

MASQUERADE scoped to bridge subnet with inverted destination:

nspawn.sh ([nspawn](https://github.com/nspawn-sh/nspawn.sh/blob/main/nspawn)):
```sh
add_ipt_rule iptables -t nat -I POSTROUTING -s "$BRIDGE_SUBNET" ! -d "$BRIDGE_SUBNET" -j MASQUERADE
```
Droidspaces v5 ([ds_iptables.c](https://github.com/ravindu644/Droidspaces-OSS/blob/main/src/ds_iptables.c)):
```c
"iptables", "-t", "nat", "-I", "POSTROUTING", "1",
"-s", src_cidr, "!", "-d", src_cidr, "-j", "MASQUERADE"
```

FORWARD rules scoped to bridge interface, not global ACCEPT:

nspawn.sh ([nspawn](https://github.com/nspawn-sh/nspawn.sh/blob/main/nspawn)):
```sh
add_ipt_rule "$ipt_cmd" -t filter -I FORWARD -i "$BRIDGE" -j ACCEPT
add_ipt_rule "$ipt_cmd" -t filter -I FORWARD -o "$BRIDGE" -j ACCEPT
```
Droidspaces v5 ([ds_iptables.c](https://github.com/ravindu644/Droidspaces-OSS/blob/main/src/ds_iptables.c)):
```c
ds_ipt_ensure_forward_accept(DS_NAT_BRIDGE)
/* -I FORWARD -i <iface> -j ACCEPT and -I FORWARD -o <iface> -j ACCEPT */
```

Android policy routing — discover active table, inject ip rules for container subnet:

nspawn.sh ([nspawn](https://github.com/nspawn-sh/nspawn.sh/blob/main/nspawn)):
```sh
ip rule add iif "$BRIDGE" lookup "$table" priority "$prio"
```
Droidspaces v5 ([network.c](https://github.com/ravindu644/Droidspaces-OSS/blob/main/src/network.c)):
```c
ds_nl_get_default_gw_table(ctx, gw_iface, &gw_table);
ds_nl_add_rule4(ctx, subnet_be, prefix, 0, 0, gw_table, 100);
```

After I emailed the author about this (I was polite, just asked for a credit line), this acknowledgment appeared in the README:

> "nspawn.sh -- Droidspaces v5 independently developed a network namespace logic and Android routing implementation identical to this project. We acknowledge nspawn.sh as the prior work published with this architecture."

This acknowledgment has since been removed from the README entirely after the Reddit dispute.

"Independently" just doesn't hold up here. v4 said they deliberately won't do network namespaces because it's too hard. v4's code shows the networking knowledge wasn't there. Then my project publishes a working solution and 24 days later v5 has what the author himself calls an "identical" implementation.

**The author's story keeps changing.** In the r/androidroot thread before it was removed:

- First he said "I didn't even know your project existed. I put this together on my own."
- Then he acknowledged nspawn.sh as "prior work" with an "identical" architecture
- Then he said "I've been working on network isolation privately in my own branch for months. Everyone in my community knows this."
- Then he said the idea came from [a PR by shedowe19](https://github.com/shedowe19/Droidspaces-OSS/pull/1) opened Feb 27
- Then it was "that PR and some random ass 10yo StackOverflow articles"

The StackOverflow link he cited ([this one](https://stackoverflow.com/questions/31667160/running-docker-container-iptables-no-chain-target-match-by-that-name)) is a 10 year old question about Docker's iptables chain missing after flushing filter rules on a desktop Linux server. It has nothing to do with Android networking, policy routing, netd, or network namespaces. The top answer is literally "flush iptables and restart docker." This is where the v4 flush approach came from and it explains why it was broken.

The PR he referenced is basic `ip` and `iptables` shell commands. No bridge, no Android policy routing, no idempotent iptables, no upstream detection, no route monitoring. It's generic Linux namespace setup. The gap between that PR and what v5 actually shipped is exactly what nspawn.sh contains.

He claimed "everyone in my community knows" about months of private networking work. His [Telegram channel](https://t.me/Droidspaces) tells a different story. Every v4.x feature from Feb 21 through Mar 5 got incremental posts, debugging stories, bug reports, even frustrated messages like "Man, I'm fed up. Every time I fix something, something else breaks." Then on March 6 at 13:24, with zero prior mention: "Pure network isolation in Android." Test APK posted 45 minutes later. No work-in-progress posts, no "I'm struggling with policy routing," no failed attempts shared. The hardest feature, the one he publicly said was too complex, just appeared fully formed with no buildup.

I don't care about the C reimplementation, the RTNETLINK stuff, the kernel iptables API. That's real work. What bothers me is calling it "independently developed" when the timeline, the code, the Telegram log, and his own shifting story all point in the same direction.

You don't go from flushing iptables and writing no-op REDIRECT rules to 1,873 lines of RTNETLINK + kernel iptables with Android policy routing in 24 days without looking at someone else's working solution.

I'm not asking for anything to get taken down. Both projects are GPL-3.0. All I originally asked for was a credit line. He added one, but wrapped it in "independently developed." Now after this dispute he's removed the acknowledgment entirely. First no credit, then misleading credit, now no credit again.
I just want the record to be public so the community knows where the networking architecture came from.
