# High Availability Nagios

## nagios

```sh
# [ng01]
cd /vagrant
ansible-playbook site.yml -t yum
ansible-playbook site.yml -CD
ansible-playbook site.yml
```

## yum

```sh
# [ng01/ng02]
sudo yum -y install http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
sudo yum -y install kmod-drbd84 pacemaker corosync resource-agents pcs
```

## drbd

```sh
# [ng01/ng02]
sudo fallocate -l 1G /var/opt/drbddisk
sudo losetup /dev/loop0 /var/opt/drbddisk

sudo tee /etc/drbd.d/data.res <<EOS
resource data
{
    protocol C;
    device /dev/drbd0;
    meta-disk internal;
    on ng01 {
        address 192.168.33.21:7791;
        disk /dev/loop0;
    }
    on ng02 {
        address 192.168.33.22:7791;
        disk /dev/loop0;
    }
}
EOS

sudo drbdadm create-md data
sudo systemctl start drbd.service
sudo systemctl status drbd.service
```

```sh
# [ng01]
sudo drbdadm primary data --force
sudo mkfs.xfs /dev/drbd/by-res/data

sudo mkdir -p /drbd/data
sudo mount /dev/drbd/by-res/data /drbd/data

sudo mv /var/log/nagios/ /drbd/data/
sudo ln -sfn /drbd/data/nagios /var/log/nagios

sudo umount /drbd/data
sudo drbdadm secondary data
```

```sh
# [ng02]
cat /proc/drbd
sudo drbdadm primary data

sudo mkdir -p /drbd/data
sudo mount /dev/drbd/by-res/data /drbd/data

sudo rm -fr /var/log/nagios
sudo ln -sfn /drbd/data/nagios /var/log/nagios

sudo umount /drbd/data
sudo drbdadm secondary data
```

```sh
# [ng01/ng02]
sudo systemctl stop drbd.service
sudo systemctl status drbd.service
```

## corosync

```sh
# [ng01/ng02]
sudo tee /etc/corosync/corosync.conf <<EOS
totem {
    version: 2
    crypto_cipher: none
    crypto_hash: none
    token: 4000
    rrp_mode: active
    transport: udpu

    interface {
        ringnumber: 0
        bindnetaddr: 192.168.33.0
        mcastaddr: 239.255.1.1
        mcastport: 5405
    }
}

logging {
    timestamp: on
    debug: off
    to_stderr: no
    to_syslog: no
    to_logfile: yes
    logfile: /var/log/cluster/corosync.log
}

nodelist {
    node {
        ring0_addr: 192.168.33.21
    }
    node {
        ring0_addr: 192.168.33.22
    }
}

quorum {
    provider: corosync_votequorum
    two_node: 1
}
EOS

sudo systemctl start pacemaker.service
sudo systemctl status pacemaker.service
sudo crm_mon -1
```

## pacemaker

```sh
# [ng01]
sudo pcs property set stonith-enabled="false"
sudo pcs property set start-failure-is-fatal="false"
sudo pcs resource defaults migration-threshold="5"
sudo pcs resource defaults resource-stickiness="INFINITY"
sudo pcs resource defaults failure-timeout="3600s"

sudo pcs resource create drbd ocf:linbit:drbd \
  drbd_resource="data" drbdconf="/etc/drbd.conf" \
    op start timeout="240s" on-fail="restart" \
    op stop  timeout="100s" on-fail="block" \
    op monitor interval="20s" timeout="20s" start-delay="1m" on-fail="restart" \
      --master meta notify="true" 

sudo pcs resource create fs ocf:heartbeat:Filesystem \
  device="/dev/drbd/by-res/data" directory="/drbd/data" fstype="xfs" \
    op start timeout="60s" on-fail="restart" \
    op stop  timeout="60s" on-fail="block" \
    op monitor interval="10s" timeout="60s" on-fail="restart"

sudo pcs resource create vip ocf:heartbeat:IPaddr2 \
  ip="192.168.33.20" cidr_netmask="24" nic="enp0s8" \
    op start timeout="20s" on-fail="restart" \
    op stop  timeout="20s" on-fail="block" \
    op monitor interval="10s" timeout="20s" on-fail="restart"

sudo pcs resource create nagios systemd:nagios \
  op monitor interval="10s"

sudo pcs resource group add group-nagios fs vip nagios

sudo pcs constraint colocation add master drbd-master with group-nagios INFINITY
sudo pcs constraint order promote drbd-master then start group-nagios

sudo pcs resource create nsca systemd:nsca \
  op monitor interval="10s" --clone

sudo pcs resource create httpd systemd:httpd \
  op monitor interval="10s" --clone

sudo pcs constraint colocation add group-nagios with nsca-clone INFINITY
sudo pcs constraint colocation add group-nagios with httpd-clone INFINITY
```

## proxy

Nagios の vip にホストからアクセスするためのプロキシを作る。

```sh
# [proxy]
sudo yum install iptables-services
sudo systemctl start iptables
sudo systemctl status iptables
sudo systemctl enable iptables
sudo iptables -F
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.33.20
sudo iptables -t nat -A POSTROUTING -p tcp --dport 80 -j SNAT --to-source 192.168.33.10
sudo sysctl -w net.ipv4.ip_forward=1
```

## 動作確認

パッシブチェック用のスクリプトを作成。

```sh
# [sv01]
sudo tee /tmp/send_nsca <<'EOS'
#!/bin/bash
set -eu
name="$(uname -n)"
printf "%s\t%s\t%s\t%s\n" "$name" "systemalarm" "$1" "$2" |sudo send_nsca -H 192.168.33.20 -to 10
wait
EOS

sudo chmod +x /tmp/send_nsca
```

パッシブチェックの結果を送ってみる。

```sh
# [sv01]
/tmp/send_nsca 1 "this is test"
```

連続でパッシブチェックの結果を送る。

```sh
# [sv01]
no=0
while sleep 1; do
  /tmp/send_nsca 1 "oreore $((no++))"
  echo $no
done
```

Nagios のアクティブ側を強制停止して様子をみてみる。

```sh
vagrant halt -f ng01
```

停止した Nagios を再開する。

```sh
vagrant up
vagrant ssh ng01

sudo rm -f /var/spool/nagios/cmd/*
sudo losetup /dev/loop0 /var/opt/drbddisk
sudo systemctl start pacemaker.service
sudo systemctl status pacemaker.service
sudo crm_mon
```

