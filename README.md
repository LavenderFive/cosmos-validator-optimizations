## Super-secret tuning tips for secret networks

I'll make a few assumptions here mostly that the reader can find the settings in the .toml files without help; validating on secret isn't exactly a good "my-first-validator" choice. 

config:

* Expose validator p2p via firewall, enable pex, make frens.
* Set inbound peers to 80 outbound to 60.
* Disable kv indexer.
* log level set to error with json output -- probably doesn't matter in this case, just less IO.
* disable mempool broadcasting on validator -- less traffic is more cycles, we already have what we need.
* don't retry previously failed transactions: `keep-invalid-txs-in-cache = true`
* remove all persistent_peers and EXCLUSIVELY use seeds

```sh
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" config.toml
sed -i -e "s/^log_level *=.*/log_level = \"error\"/" config.toml
sed -i -e "s/^log_format *=.*/log_format = \"json\"/" config.toml
sed -i -e "s/^broadcast *=.*/broadcast = false/" config.toml
sed -i -e "s/^keep-invalid-txs-in-cache *=.*/keep-invalid-txs-in-cache = true/" config.toml
sed -i -e "s/^max_num_inbound_peers *=.*/max_num_inbound_peers = 80/" config.toml
sed -i -e "s/^max_num_outbound_peers *=.*/max_num_outbound_peers = 60/" config.toml
```

app:

* Set num states to retain to a different prime on each node around 100 (ie 109, 107, 103 etc.)
* Set prune interval to different primes on each node (ie 17, 43, 61)
* Use cosmprund to trim state to match retention
* disable state snapshots (please run them on at least one node, network needs snapshots to state-sync.)

```sh
pruning="custom" && \
pruning_keep_recent=107 && \
pruning_keep_every=0 && \
pruning_interval=73 && \
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" app.toml && \
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" app.toml && \
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" app.toml && \
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" app.toml
```

## system tuning:

*apologies to non-ubuntu admins, but you are already used to this shit so figure it out, you are used to it*

* Set CPU governor to performance

```bash
apt-get -y install cpufrequtils

cat > /etc/default/cpufrequtils << EOF
ENABLE="true"
GOVERNOR="performance"
EOF
systemctl restart cpufrequtils
```

* Set Niceness for secretd to -20 via systemd local config

```
mkdir -p /etc/systemd/system/secret-node.service.d/
cat > /etc/systemd/system/secret-node.service.d/local.conf << EOF
[Service]
Nice=-20

EOF
systemctl daemon-reload
systemctl restart secret-node
```

* Bump initial TCP congestion windows to 32 and use fair queueing:

*this is route-dependent, we use ansible to build it, but this example assumes you are adding it by hand... adjust accordingly*

add the following to root's crontab:

```
@reboot /sbin/tc qdisc add dev <FIXME-default-interface> root fq
@reboot /usr/bin/ip route change default via <FIXME-default-route-IP-addr> initcwnd 32 initrwnd 32
```

* Bump up network buffer sizes in kernel, a bunch of other misc tuning for faster TCP

*you should research each of these settings on your own, not gonna explain here. You are responsible, not me.*

```bash
cat >> /etc/sysctl.conf << EOF
net.core.rmem_max=16777216
net.core.wmem_max=16777216
net.ipv4.tcp_max_syn_backlog=8192
net.core.netdev_max_backlog=65536
net.ipv4.tcp_slow_start_after_idle=0
net.ipv4.tcp_mtu_probing=1
net.ipv4.tcp_sack=0
net.ipv4.tcp_dsack=0
net.ipv4.tcp_fack=0
net.ipv4.tcp_congestion_control=bbr

EOF
sysctl -p
```

* Using striped raid instead of mirrored

*We use zfs almost exclusively because of the ability to snapshot. It sucks for TM chains because of the overhead. This is a short example of creating a stripe, and the settings we use to reduce the IO overhead zfs imposes from about 8x to only about 2x. Once again, don't blindly trust these settings. Google the hell out of these, they might not be what **you** want.*

```bash
zpool create data nvme0n1 nvme1n1 # obviously fix these for correct device names.
zfs set compression=lz4 data
zfs set atime=off data
zfs set logbias=throughput data
zfs set redundant_metadata=most data
zfs set secondarycache=none data

echo 1 > /sys/module/zfs/parameters/zfs_prefetch_disable
echo 'options zfs zfs_prefetch_disable=1' >> /etc/modprobe.d/zfs.conf

zfs create -o mountpoint=/opt/secret data/secret
```
