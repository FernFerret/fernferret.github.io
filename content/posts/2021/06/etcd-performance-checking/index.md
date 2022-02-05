---
title: "Etcd Performance Checking"
date: 2021-06-13T14:43:00Z
draft: true
---

I wanted to see how different nodes in my cluster were performing with for etcd
as a control-plane for Kubernetes. Here are some of my notes.

I started this because I thought I wasn't reading the grafana charts for etcd
correctly...

TLDR: I had a bad SSD.

First I shell'd into each of the etcd pods:
<!--more-->

```shell
kubectl exec -it -n kube-system etcd-cubert -- sh
kubectl exec -it -n kube-system etcd-igner -- sh
kubectl exec -it -n kube-system etcd-larry -- sh
```

Now we can run the performance test

```shell
ETCDCTL=3 etcdctl check perf \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --key /etc/kubernetes/pki/etcd/peer.key \
  --cert /etc/kubernetes/pki/etcd/peer.crt
```

You'll see the test results start to come across as a progress bar. Here were
some of my results:

### cubert (EVO 860)

```text
 60 / 60 Boooooooooooooooooo! 100.00% 1m0s
FAIL: Throughput too low: 72 writes/s
Slowest request took too long: 1.354317s
Stddev too high: 0.452125s
FAIL
---
 60 / 60 Boooooooooooooooooo! 100.00% 1m0s
FAIL: Throughput too low: 83 writes/s
Slowest request took too long: 1.302802s
Stddev too high: 0.347069s
FAIL
---
 60 / 60 Boooooooooooooooooo! 100.00% 1m0s
FAIL: Throughput too low: 62 writes/s
Slowest request took too long: 5.019048s
Stddev too high: 0.886701s
FAIL
```

### igner (Hitachi 5400)

```text
 60 / 60 Boooooooooooooooooo! 100.00% 1m0s
PASS: Throughput is 150 writes/s
Slowest request took too long: 0.612762s
PASS: Stddev is 0.066612s
FAIL
---
 60 / 60 Boooooooooooooooooo! 100.00% 1m0s
PASS: Throughput is 150 writes/s
Slowest request took too long: 0.705537s
PASS: Stddev is 0.070527s
FAIL
---
 60 / 60 Boooooooooooooooooo! 100.00% 1m0s
PASS: Throughput is 150 writes/s
PASS: Slowest request took 0.387492s
PASS: Stddev is 0.045938s
PASS
```

### larry (Hitachi 5400)

```text
 60 / 60 Boooooooooooooooooo! 100.00% 1m0s
PASS: Throughput is 147 writes/s
Slowest request took too long: 1.029967s
Stddev too high: 0.104906s
FAIL
---
 60 / 60 Boooooooooooooooooo! 100.00% 1m0s
PASS: Throughput is 149 writes/s
Slowest request took too long: 0.674924s
PASS: Stddev is 0.078296s
FAIL
---
 60 / 60 Boooooooooooooooooo! 100.00% 1m0s
PASS: Throughput is 144 writes/s
Slowest request took too long: 1.013068s
Stddev too high: 0.105969s
FAIL
```

Of these tests there was only 1 whole pass and that was on `igner`. I believe it
"barely" passed this test. `cubert` never passed any tests...

That's not good... this seems like I'm reading my plots correctly but I have a
bad SSD.

## FIO Tests

[IBM has a pretty good
article](https://www.ibm.com/cloud/blog/using-fio-to-tell-whether-your-storage-is-fast-enough-for-etcd)
on using `fio` to test disk performance targeted at `etcd`. So I ran those
benchmarks below on each of my `etcd` servers. The results match the `etcd`
reported metrics (which is great) but that means they're really bad too...

### cubert (EVO 860) - fio

```text
$ mkdir test-data

$ fio --rw=write --ioengine=sync --fdatasync=1 --directory=test-data --size=22m --bs=2300 --name=mytest
mytest: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.16
Starting 1 process
mytest: Laying out IO file (1 file / 22MiB)
Jobs: 1 (f=1): [W(1)][100.0%][w=89KiB/s][w=40 IOPS][eta 00m:00s]
mytest: (groupid=0, jobs=1): err= 0: pid=828951: Sun Jun 13 10:05:41 2021
  write: IOPS=11, BW=25.9KiB/s (26.5kB/s)(21.0MiB/871213msec); 0 zone resets
    clat (usec): min=4, max=94356, avg=44.03, stdev=942.11
     lat (usec): min=4, max=94357, avg=44.97, stdev=942.11
    clat percentiles (usec):
     |  1.00th=[   10],  5.00th=[   15], 10.00th=[   18], 20.00th=[   23],
     | 30.00th=[   26], 40.00th=[   30], 50.00th=[   32], 60.00th=[   35],
     | 70.00th=[   39], 80.00th=[   45], 90.00th=[   51], 95.00th=[   59],
     | 99.00th=[   98], 99.50th=[  147], 99.90th=[  235], 99.95th=[  269],
     | 99.99th=[ 1045]
   bw (  KiB/s): min=    4, max=  215, per=100.00%, avg=25.14, stdev=45.89, samples=1740
   iops        : min=    1, max=   96, avg=11.50, stdev=20.38, samples=1740
  lat (usec)   : 10=1.08%, 20=11.73%, 50=75.80%, 100=10.45%, 250=0.87%
  lat (usec)   : 500=0.05%, 750=0.01%
  lat (msec)   : 2=0.01%, 100=0.01%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=887, max=543792, avg=86813.80, stdev=122000.65
    sync percentiles (usec):
     |  1.00th=[   996],  5.00th=[  1029], 10.00th=[  1106], 20.00th=[  2278],
     | 30.00th=[  5604], 40.00th=[  9896], 50.00th=[ 11994], 60.00th=[ 23725],
     | 70.00th=[102237], 80.00th=[196084], 90.00th=[295699], 95.00th=[333448],
     | 99.00th=[434111], 99.50th=[497026], 99.90th=[534774], 99.95th=[534774],
     | 99.99th=[541066]
  cpu          : usr=0.02%, sys=0.13%, ctx=24693, majf=0, minf=13
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,10029,0,0 short=10029,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=25.9KiB/s (26.5kB/s), 25.9KiB/s-25.9KiB/s (26.5kB/s-26.5kB/s), io=21.0MiB (23.1MB), run=871213-871213msec

Disk stats (read/write):
  sda: ios=198/51581, merge=32/26209, ticks=12595/2326664, in_queue=2270740, util=85.35%
```

#### cubert's results

These are interesting. `etcd` wants the 99th percentile of the fsync stat to be
`10ms`... We're at `434ms`. However, the 50th percentile is `11ms` which is 1/4
of the other 2 servers. This server has an 

```text
  fsync/fdatasync/sync_file_range:
    sync (msec): min=8, max=332, avg=48.06, stdev=26.85
    sync percentiles (msec):
     |  1.00th=[   12],  5.00th=[   14], 10.00th=[   16], 20.00th=[   21],
     | 30.00th=[   34], 40.00th=[   40], 50.00th=[   45], 60.00th=[   52],
     | 70.00th=[   59], 80.00th=[   68], 90.00th=[   84], 95.00th=[   94],
     | 99.00th=[  120], 99.50th=[  132], 99.90th=[  230], 99.95th=[  259],
     | 99.99th=[  288]
```

### igner (Hitachi 5400) - fio

```text
$ mkdir test-data
$ fio --rw=write --ioengine=sync --fdatasync=1 --directory=test-data --size=22m --bs=2300 --name=mytest
mytest: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.16
Starting 1 process
mytest: Laying out IO file (1 file / 22MiB)
Jobs: 1 (f=1): [W(1)][100.0%][w=53KiB/s][w=24 IOPS][eta 00m:00s]
mytest: (groupid=0, jobs=1): err= 0: pid=2583395: Sun Jun 13 09:59:12 2021
  write: IOPS=20, BW=46.7KiB/s (47.8kB/s)(21.0MiB/482609msec); 0 zone resets
    clat (usec): min=9, max=2593, avg=47.61, stdev=102.02
     lat (usec): min=9, max=2594, avg=48.60, stdev=102.02
    clat percentiles (usec):
     |  1.00th=[   19],  5.00th=[   24], 10.00th=[   26], 20.00th=[   29],
     | 30.00th=[   31], 40.00th=[   33], 50.00th=[   35], 60.00th=[   38],
     | 70.00th=[   41], 80.00th=[   45], 90.00th=[   50], 95.00th=[   56],
     | 99.00th=[  429], 99.50th=[ 1045], 99.90th=[ 1221], 99.95th=[ 1237],
     | 99.99th=[ 1303]
   bw (  KiB/s): min=   13, max=   71, per=100.00%, avg=46.04, stdev= 9.10, samples=965
   iops        : min=    6, max=   32, avg=20.76, stdev= 4.05, samples=965
  lat (usec)   : 10=0.01%, 20=1.68%, 50=89.09%, 100=7.47%, 250=0.06%
  lat (usec)   : 500=0.82%, 750=0.04%, 1000=0.18%
  lat (msec)   : 2=0.65%, 4=0.01%
  fsync/fdatasync/sync_file_range:
    sync (msec): min=8, max=332, avg=48.06, stdev=26.85
    sync percentiles (msec):
     |  1.00th=[   12],  5.00th=[   14], 10.00th=[   16], 20.00th=[   21],
     | 30.00th=[   34], 40.00th=[   40], 50.00th=[   45], 60.00th=[   52],
     | 70.00th=[   59], 80.00th=[   68], 90.00th=[   84], 95.00th=[   94],
     | 99.00th=[  120], 99.50th=[  132], 99.90th=[  230], 99.95th=[  259],
     | 99.99th=[  288]
  cpu          : usr=0.06%, sys=0.24%, ctx=24078, majf=0, minf=13
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,10029,0,0 short=10029,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=46.7KiB/s (47.8kB/s), 46.7KiB/s-46.7KiB/s (47.8kB/s-47.8kB/s), io=21.0MiB (23.1MB), run=482609-482609msec

Disk stats (read/write):
  sda: ios=881/47558, merge=5/21221, ticks=34320/946245, in_queue=912040, util=89.52%
```

#### igner's results

Oof. These are bad. `etcd` wants the 99th percentile of the fsync stat to be
`10ms`... We're at `120ms` so an order of magnitude greater:

```text
  fsync/fdatasync/sync_file_range:
    sync (msec): min=8, max=332, avg=48.06, stdev=26.85
    sync percentiles (msec):
     |  1.00th=[   12],  5.00th=[   14], 10.00th=[   16], 20.00th=[   21],
     | 30.00th=[   34], 40.00th=[   40], 50.00th=[   45], 60.00th=[   52],
     | 70.00th=[   59], 80.00th=[   68], 90.00th=[   84], 95.00th=[   94],
     | 99.00th=[  120], 99.50th=[  132], 99.90th=[  230], 99.95th=[  259],
     | 99.99th=[  288]
```

### larry (Hitachi 5400) - fio

```text
$ mkdir test-data
$ fio --rw=write --ioengine=sync --fdatasync=1 --directory=test-data --size=22m --bs=2300 --name=mytest
mytest: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.16
Starting 1 process
mytest: Laying out IO file (1 file / 22MiB)
Jobs: 1 (f=1): [W(1)][100.0%][w=67KiB/s][w=30 IOPS][eta 00m:00s]
mytest: (groupid=0, jobs=1): err= 0: pid=3219932: Sun Jun 13 09:58:36 2021
  write: IOPS=22, BW=50.5KiB/s (51.7kB/s)(21.0MiB/446371msec); 0 zone resets
    clat (usec): min=10, max=1921, avg=41.64, stdev=75.21
     lat (usec): min=11, max=1922, avg=42.60, stdev=75.23
    clat percentiles (usec):
     |  1.00th=[   19],  5.00th=[   23], 10.00th=[   24], 20.00th=[   28],
     | 30.00th=[   30], 40.00th=[   32], 50.00th=[   34], 60.00th=[   36],
     | 70.00th=[   39], 80.00th=[   42], 90.00th=[   48], 95.00th=[   54],
     | 99.00th=[  285], 99.50th=[  469], 99.90th=[ 1074], 99.95th=[ 1139],
     | 99.99th=[ 1336]
   bw (  KiB/s): min=   13, max=   80, per=99.61%, avg=49.81, stdev=10.69, samples=892
   iops        : min=    6, max=   36, avg=22.45, stdev= 4.76, samples=892
  lat (usec)   : 20=1.85%, 50=91.04%, 100=5.77%, 250=0.08%, 500=0.77%
  lat (usec)   : 1000=0.19%
  lat (msec)   : 2=0.30%
  fsync/fdatasync/sync_file_range:
    sync (msec): min=3, max=257, avg=44.45, stdev=27.08
    sync percentiles (msec):
     |  1.00th=[    8],  5.00th=[   11], 10.00th=[   13], 20.00th=[   19],
     | 30.00th=[   32], 40.00th=[   36], 50.00th=[   41], 60.00th=[   47],
     | 70.00th=[   54], 80.00th=[   64], 90.00th=[   80], 95.00th=[   95],
     | 99.00th=[  125], 99.50th=[  142], 99.90th=[  209], 99.95th=[  239],
     | 99.99th=[  251]
  cpu          : usr=0.05%, sys=0.26%, ctx=24219, majf=0, minf=13
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,10029,0,0 short=10029,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=50.5KiB/s (51.7kB/s), 50.5KiB/s-50.5KiB/s (51.7kB/s-51.7kB/s), io=21.0MiB (23.1MB), run=446371-446371msec

Disk stats (read/write):
  sda: ios=326/45763, merge=103/20509, ticks=12723/853787, in_queue=801240, util=89.23%
```

#### Larry's Results

Oof. These are bad. `etcd` wants the 99th percentile of the fsync stat to be
`10ms`... We're at `125ms` so an order of magnitude greater:

```text
  fsync/fdatasync/sync_file_range:
    sync (msec): min=3, max=257, avg=44.45, stdev=27.08
    sync percentiles (msec):
     |  1.00th=[    8],  5.00th=[   11], 10.00th=[   13], 20.00th=[   19],
     | 30.00th=[   32], 40.00th=[   36], 50.00th=[   41], 60.00th=[   47],
     | 70.00th=[   54], 80.00th=[   64], 90.00th=[   80], 95.00th=[   95],
     | 99.00th=[  125], 99.50th=[  142], 99.90th=[  209], 99.95th=[  239],
     | 99.99th=[  251]
```

### walt (870 EVO)

```text
$ mkdir test-data
$ fio --rw=write --ioengine=sync --fdatasync=1 --directory=test-data --size=22m --bs=2300 --name=mytest
mytest: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.16
Starting 1 process
mytest: Laying out IO file (1 file / 22MiB)
Jobs: 1 (f=1): [W(1)][100.0%][w=1688KiB/s][w=751 IOPS][eta 00m:00s]
mytest: (groupid=0, jobs=1): err= 0: pid=866717: Sun Jun 13 12:38:39 2021
  write: IOPS=737, BW=1658KiB/s (1697kB/s)(21.0MiB/13590msec); 0 zone resets
    clat (usec): min=3, max=441, avg=16.43, stdev= 7.18
     lat (usec): min=3, max=442, avg=16.91, stdev= 7.29
    clat percentiles (nsec):
     |  1.00th=[ 7840],  5.00th=[ 8384], 10.00th=[ 9024], 20.00th=[12480],
     | 30.00th=[13120], 40.00th=[14272], 50.00th=[14784], 60.00th=[15296],
     | 70.00th=[18816], 80.00th=[21376], 90.00th=[23680], 95.00th=[28800],
     | 99.00th=[31360], 99.50th=[34560], 99.90th=[43776], 99.95th=[46336],
     | 99.99th=[61696]
   bw (  KiB/s): min= 1545, max= 1720, per=100.00%, avg=1657.63, stdev=44.92, samples=27
   iops        : min=  688, max=  766, avg=738.15, stdev=19.99, samples=27
  lat (usec)   : 4=0.05%, 10=12.95%, 20=59.30%, 50=27.66%, 100=0.03%
  lat (usec)   : 500=0.01%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=666, max=8970, avg=1332.68, stdev=541.16
    sync percentiles (usec):
     |  1.00th=[  701],  5.00th=[  717], 10.00th=[  734], 20.00th=[  758],
     | 30.00th=[  783], 40.00th=[  848], 50.00th=[ 1647], 60.00th=[ 1713],
     | 70.00th=[ 1729], 80.00th=[ 1778], 90.00th=[ 1827], 95.00th=[ 1860],
     | 99.00th=[ 2073], 99.50th=[ 2311], 99.90th=[ 5211], 99.95th=[ 5669],
     | 99.99th=[ 8979]
  cpu          : usr=1.07%, sys=5.95%, ctx=24468, majf=0, minf=12
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,10029,0,0 short=10029,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=1658KiB/s (1697kB/s), 1658KiB/s-1658KiB/s (1697kB/s-1697kB/s), io=21.0MiB (23.1MB), run=13590-13590msec

Disk stats (read/write):
    dm-0: ios=457/42186, merge=0/0, ticks=316/12644, in_queue=12960, util=99.33%, aggrios=455/31472, aggrmerge=2/11305, aggrticks=353/12438, aggrin_queue=96, aggrutil=99.28%
  sda: ios=455/31472, merge=2/11305, ticks=353/12438, in_queue=96, util=99.28%
```

#### TLDR

These actually look fine, we're well within the margin of `10ms`, we're at
`2ms`.

```text
    sync percentiles (usec):
     |  1.00th=[  701],  5.00th=[  717], 10.00th=[  734], 20.00th=[  758],
     | 30.00th=[  783], 40.00th=[  848], 50.00th=[ 1647], 60.00th=[ 1713],
     | 70.00th=[ 1729], 80.00th=[ 1778], 90.00th=[ 1827], 95.00th=[ 1860],
     | 99.00th=[ 2073], 99.50th=[ 2311], 99.90th=[ 5211], 99.95th=[ 5669],
     | 99.99th=[ 8979]
```

## Hardware Summary

### cubert (EVO 860)

```text
$ sudo hdparm -I /dev/sda

/dev/sda:

ATA device, with non-removable media
    Model Number:       Samsung SSD 860 QVO 1TB
    Serial Number:      XXXXXXXX
    Firmware Revision:  RVQ02B6Q
    Transport:          Serial, ATA8-AST, SATA 1.0a, SATA II Extensions, 
                        SATA Rev 2.5, SATA Rev 2.6, SATA Rev 3.0
Standards:
    Used: unknown (minor revision code 0x005e)
    Supported: 11 8 7 6 5
    Likely used: 11
Configuration:
    Logical      max  current
    cylinders  16383    16383
    heads         16       16
    sectors/track 63       63
    --
    CHS current addressable sectors:    16514064
    LBA    user addressable sectors:   268435455
    LBA48  user addressable sectors:  1953525168
    Logical  Sector size:                   512 bytes
    Physical Sector size:                   512 bytes
    Logical Sector-0 offset:                  0 bytes
    device size with M = 1024*1024:      953869 MBytes
    device size with M = 1000*1000:     1000204 MBytes (1000 GB)
    cache/buffer size  = unknown
    Form Factor: 2.5 inch
    Nominal Media Rotation Rate: Solid State Device

$ sudo smartctl -a /dev/sda | grep SATA
SATA Version is:  SATA 3.2, 6.0 Gb/s (current: 6.0 Gb/s)
```

### igner (Hitachi 5400)

```text
$ sudo hdparm -I /dev/sda

/dev/sda:

ATA device, with non-removable media
    Model Number:       HITACHI HTS543232L9SA00
    Serial Number:      XXXXXXXX
    Firmware Revision:  FB4ZC4EC
    Transport:          Serial, ATA8-AST, SATA 1.0a, SATA II Extensions,
                        SATA Rev 2.5, SATA Rev 2.6; Revision: ATA8-AST T13 Project D1697 Revision 0b
Standards:
    Used: ATA-8-ACS revision 3f
    Supported: 8 7 6 5
Configuration:
    Logical      max  current
    cylinders  16383    16383
    heads         16       16
    sectors/track 63       63
    --
    CHS current addressable sectors:    16514064
    LBA    user addressable sectors:   268435455
    LBA48  user addressable sectors:   625142448
    Logical  Sector size:                   512 bytes
    Physical Sector size:                   512 bytes
    device size with M = 1024*1024:      305245 MBytes
    device size with M = 1000*1000:      320072 MBytes (320 GB)
    cache/buffer size  = 7114 KBytes (type=DualPortCache)
    Nominal Media Rotation Rate: 5400

$ sudo smartctl -a /dev/sda | grep SATA
SATA Version is:  SATA 2.6, 1.5 Gb/s
```

### larry (Hitachi 5400)

```text
$ sudo hdparm -I /dev/sda

/dev/sda:

ATA device, with non-removable media
    Model Number:       HITACHI HTS543232L9SA00
    Serial Number:      XXXXXXXX
    Firmware Revision:  FB4ZC4EC
    Transport:          Serial, ATA8-AST, SATA 1.0a, SATA II Extensions, 
                        SATA Rev 2.5, SATA Rev 2.6; Revision: ATA8-AST T13 Project D1697 Revision 0b
Standards:
    Used: ATA-8-ACS revision 3f
    Supported: 8 7 6 5
Configuration:
    Logical      max  current
    cylinders  16383    16383
    heads         16       16
    sectors/track 63       63
    --
    CHS current addressable sectors:    16514064
    LBA    user addressable sectors:   268435455
    LBA48  user addressable sectors:   625142448
    Logical  Sector size:                   512 bytes
    Physical Sector size:                   512 bytes
    device size with M = 1024*1024:      305245 MBytes
    device size with M = 1000*1000:      320072 MBytes (320 GB)
    cache/buffer size  = 7114 KBytes (type=DualPortCache)
    Nominal Media Rotation Rate: 5400

$ sudo smartctl -a /dev/sda | grep SATA
SATA Version is:  SATA 2.6, 1.5 Gb/s
```

### walt (EVO 870)
```text
$ sudo hdparm -I /dev/sda

/dev/sda:

ATA device, with non-removable media
    Model Number:       Samsung SSD 870 EVO 250GB
    Serial Number:      XXXXXXXX
    Firmware Revision:  SVT01B6Q
    Transport:          Serial, ATA8-AST, SATA 1.0a, SATA II Extensions,
                        SATA Rev 2.5, SATA Rev 2.6, SATA Rev 3.0
Standards:
    Used: unknown (minor revision code 0x005e)
    Supported: 11 8 7 6 5
    Likely used: 11
Configuration:
    Logical      max  current
    cylinders  16383    16383
    heads         16       16
    sectors/track 63       63
    --
    CHS current addressable sectors:    16514064
    LBA    user addressable sectors:   268435455
    LBA48  user addressable sectors:   488397168
    Logical  Sector size:                   512 bytes
    Physical Sector size:                   512 bytes
    Logical Sector-0 offset:                  0 bytes
    device size with M = 1024*1024:      238475 MBytes
    device size with M = 1000*1000:      250059 MBytes (250 GB)
    cache/buffer size  = unknown
    Form Factor: 2.5 inch
    Nominal Media Rotation Rate: Solid State Device

$ sudo smartctl -a /dev/sda | grep SATA
SATA Version is:  SATA 3.3, 6.0 Gb/s (current: 6.0 Gb/s)
```

## Reference Numbers

I also ran the same `fio` tests on `leela`/`bender`. These were insanely fast;
likely due to the disk cache. But this means... what on earth is going on with
the Samsung EVO 960 SSD. It should clearly not be the slowest and I'd have
thought it would beat HP Gen7 era servers.

### leela (Dell 720XD + PERC H710P Mini)

```text
$ cd /mnt/ldata1
$ mkdir test-data
$ fio --rw=write --ioengine=sync --fdatasync=1 --directory=test-data --size=22m --bs=2300 --name=mytest

mytest: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.16
Starting 1 process
mytest: Laying out IO file (1 file / 22MiB)
Jobs: 1 (f=1)
mytest: (groupid=0, jobs=1): err= 0: pid=946397: Sun Jun 13 10:53:07 2021
  write: IOPS=3532, BW=7935KiB/s (8125kB/s)(21.0MiB/2839msec); 0 zone resets
    clat (usec): min=6, max=178, avg=13.87, stdev= 4.63
     lat (usec): min=6, max=178, avg=14.38, stdev= 4.88
    clat percentiles (nsec):
     |  1.00th=[ 7200],  5.00th=[ 8512], 10.00th=[ 9536], 20.00th=[11072],
     | 30.00th=[11712], 40.00th=[12096], 50.00th=[13632], 60.00th=[14144],
     | 70.00th=[14784], 80.00th=[16512], 90.00th=[18816], 95.00th=[19840],
     | 99.00th=[26496], 99.50th=[29312], 99.90th=[55040], 99.95th=[90624],
     | 99.99th=[99840]
   bw (  KiB/s): min= 7120, max= 9191, per=100.00%, avg=8053.20, stdev=827.59, samples=5
   iops        : min= 3170, max= 4092, avg=3585.60, stdev=368.42, samples=5
  lat (usec)   : 10=12.63%, 20=82.74%, 50=4.51%, 100=0.10%, 250=0.02%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=62, max=4810, avg=263.47, stdev=177.44
    sync percentiles (usec):
     |  1.00th=[   67],  5.00th=[   82], 10.00th=[   94], 20.00th=[  120],
     | 30.00th=[  137], 40.00th=[  172], 50.00th=[  210], 60.00th=[  281],
     | 70.00th=[  347], 80.00th=[  416], 90.00th=[  494], 95.00th=[  553],
     | 99.00th=[  635], 99.50th=[  668], 99.90th=[ 1532], 99.95th=[ 2180],
     | 99.99th=[ 4228]
  cpu          : usr=5.21%, sys=29.07%, ctx=32914, majf=0, minf=12
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,10029,0,0 short=10029,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=7935KiB/s (8125kB/s), 7935KiB/s-7935KiB/s (8125kB/s-8125kB/s), io=21.0MiB (23.1MB), run=2839-2839msec

Disk stats (read/write):
  sdb: ios=0/24086, merge=0/0, ticks=0/1784, in_queue=4, util=96.47%
```

### bender (Dell R510 + H700 (512MB BBWC))

```text
$ cd /mnt/ldata1
$ mkdir test-data
$ fio --rw=write --ioengine=sync --fdatasync=1 --directory=test-data --size=22m --bs=2300 --name=mytest
mytest: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.16
Starting 1 process
mytest: Laying out IO file (1 file / 22MiB)
Jobs: 1 (f=1)
mytest: (groupid=0, jobs=1): err= 0: pid=1873068: Sun Jun 13 10:51:24 2021
  write: IOPS=6408, BW=14.1MiB/s (14.7MB/s)(21.0MiB/1565msec); 0 zone resets
    clat (usec): min=5, max=136, avg=12.26, stdev= 6.53
     lat (usec): min=5, max=136, avg=12.52, stdev= 6.63
    clat percentiles (usec):
     |  1.00th=[    6],  5.00th=[    6], 10.00th=[    7], 20.00th=[    9],
     | 30.00th=[   10], 40.00th=[   10], 50.00th=[   11], 60.00th=[   12],
     | 70.00th=[   13], 80.00th=[   16], 90.00th=[   20], 95.00th=[   25],
     | 99.00th=[   36], 99.50th=[   42], 99.90th=[   68], 99.95th=[   72],
     | 99.99th=[  135]
   bw (  KiB/s): min=13795, max=14887, per=100.00%, avg=14440.67, stdev=572.64, samples=3
   iops        : min= 6142, max= 6628, avg=6429.33, stdev=254.84, samples=3
  lat (usec)   : 10=41.00%, 20=50.27%, 50=8.51%, 100=0.19%, 250=0.03%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=49, max=5009, avg=140.56, stdev=94.61
    sync percentiles (usec):
     |  1.00th=[   52],  5.00th=[   54], 10.00th=[   56], 20.00th=[   60],
     | 30.00th=[   72], 40.00th=[   94], 50.00th=[  157], 60.00th=[  165],
     | 70.00th=[  178], 80.00th=[  196], 90.00th=[  227], 95.00th=[  260],
     | 99.00th=[  343], 99.50th=[  408], 99.90th=[  562], 99.95th=[  840],
     | 99.99th=[ 3032]
  cpu          : usr=3.52%, sys=28.52%, ctx=20104, majf=0, minf=13
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,10029,0,0 short=10029,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=14.1MiB/s (14.7MB/s), 14.1MiB/s-14.1MiB/s (14.7MB/s-14.7MB/s), io=21.0MiB (23.1MB), run=1565-1565msec

Disk stats (read/write):
  sdb: ios=0/23645, merge=0/10582, ticks=0/980, in_queue=0, util=93.32%
```

## Other Disk Tests

I decided to gather a few more data points just to make sure i was confident my
disks were the issue.

### sysbench

Run this on each server:

```text
$ mkdir -p /tmp/bench
$ cd /tmp/bench
$ sysbench fileio prepare
```

Now do the benchmark, I'll show a few of the stats:

#### leela

Enterprise RAID

```text
$ sysbench fileio --file-test-mode=rndrw run
File operations:
    reads/s:                      3538.52
    writes/s:                     2359.05
    fsyncs/s:                     7550.55

Throughput:
    read, MiB/s:                  55.29
    written, MiB/s:               36.86
```

#### cubert

Samsung EVO 860 (see above)

```text
$ sysbench fileio --file-test-mode=rndrw run
File operations:
    reads/s:                      15.42
    writes/s:                     10.28
    fsyncs/s:                     42.57

Throughput:
    read, MiB/s:                  0.24
    written, MiB/s:               0.16
```

#### igner

Old/Bad 5400 drive (see above)

```text
$ sysbench fileio --file-test-mode=rndrw run
File operations:
    reads/s:                      45.96
    writes/s:                     30.64
    fsyncs/s:                     99.77

Throughput:
    read, MiB/s:                  0.72
    written, MiB/s:               0.48
```

The question remains, why is my EVO performing worse than a 5400 rpm drive...

#### walt

Walt died too and he got an evo 870 (see above)

```text
File operations:
    reads/s:                      1569.17
    writes/s:                     1046.11
    fsyncs/s:                     3356.44

Throughput:
    read, MiB/s:                  24.52
    written, MiB/s:               16.35
```

These are not awful, still not up to par with the G8s but still much better than
the 5400 or the EVO 860