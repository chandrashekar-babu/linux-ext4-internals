# Linux Ext4 Internals: Command Notes

### Mount Options for R/O access
```bash
# Mount existing filesystem read-only
mount -t ext4 -o ro /dev/sda1 /mnt

# Force read-only even if journal needs recovery
mount -t ext4 -o ro,noload /dev/sda1 /mnt

# Auto remount read-only on errors
mount -t ext4 -o errors=remount-ro /dev/sda1 /mnt
```

#### Manage Stride and Stripe Width
```bash
# For RAID arrays
mkfs.ext4 -E stride=64,stripe_width=128 /dev/md0

# Mount with alignment hints
mount -o stripe=64 /dev/md0 /mnt
```

#### Setup reserved blocks
```bash
# Reserve blocks for root (5% default)
tune2fs -m 2 /dev/sda1  # Reduce to 2%

# Prevent ENOSPC fragmentation
echo 10 > /proc/sys/fs/ext4/mb_min_to_scan
```

#### Discard options for mount
```bash
# Synchronous discard (every free)
mount -o discard /dev/sda1 /mnt

# Asynchronous (requires fstrim)
mount -o nodiscard /dev/sda1 /mnt

# Batch discard
fstrim -v /mnt
fstrim -v -o 1G -l 10G /mnt  # Range discard
```

#### Discard / TRIM best practices
```bash
# For eMMC/UFS - use periodic fstrim
echo "0 3 * * * root /sbin/fstrim -v /data" >> /etc/crontab

# For high-end SSD - discard mount option may be fine
mount -o discard,noatime /dev/nvme0n1 /mnt

# Check discard support
cat /sys/block/sda/queue/discard_granularity
cat /sys/block/sda/queue/discard_max_bytes
```

#### Configure Journal Size
```bash
# Create with custom journal size
mkfs.ext4 -J size=400 /dev/sda1   # 400MB journal

# Default: 128MB for 4TB filesystem
# Rule of thumb: 1GB journal per 10TB data
# Larger journal = better for bursty writes

# Check current size
dumpe2fs /dev/sda1 | grep "Journal size"
```



### Inspect Ext4 on-disk structures using debugfs
```bash
# Create test filesystem
dd if=/dev/zero of=ext4_test.img bs=1M count=512
mkfs.ext4 -F ext4_test.img

# Open with debugfs
debugfs -w ext4_test.img

# Inside debugfs
debugfs:  stats                    # Show superblock info
debugfs:  show_super_stats -h      # Detailed stats
debugfs:  extents /path/to/file    # Show extent tree
debugfs:  imap <inode_number>      # Show inode bitmap
debugfs:  block_dump <block>       # Dump block contents
debugfs:  ncheck <inode>           # Find path by inode
debugfs:  icheck <block>           # Find inode by block
debugfs:  logdump -a               # Dump journal

# Exit debugfs
debugfs:  quit
```

### Tracing Ext4 kernel code using ftrace
```bash
# Enable function trace
echo function > /sys/kernel/debug/tracing/current_tracer
echo ext4_fill_super > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on

# Mount a filesystem
mount -t ext4 /dev/sda1 /mnt

# View trace
cat /sys/kernel/debug/tracing/trace
```

### Setup journal tracing
```bash
# Enable JBD2 tracepoints
echo 1 > /sys/kernel/debug/tracing/events/jbd2/enable

# Available tracepoints
ls /sys/kernel/debug/tracing/events/jbd2/
# jbd2_checkpoint         jbd2_commit_flushing
# jbd2_commit_logging     jbd2_commit_locking
# jbd2_end_commit         jbd2_run_stats
# jbd2_start_commit       jbd2_submit_inode_data

# Trace specific operations
echo 'jbd2_start_commit' > /sys/kernel/debug/tracing/set_event
echo 'jbd2_end_commit' >> /sys/kernel/debug/tracing/set_event

# Run workload
dd if=/dev/zero of=testfile bs=4k count=1000 conv=fsync

# View trace
cat /sys/kernel/debug/tracing/trace
```

### Journal tracing with perf
```bash
# JBD2 performance stats
perf stat -e jbd2:* -a -- sleep 10

# Call graph analysis
perf record -e jbd2:* -ag -- sleep 10
perf report
```

### Journal Recovery Simulation
```bash
# 1. Setup environment
# Create test filesystem
dd if=/dev/zero of=/tmp/ext4_test.img bs=1M count=1024
mkfs.ext4 -F /tmp/ext4_test.img

# Mount with journal
mkdir /mnt/ext4_test
mount -o loop,data=ordered /tmp/ext4_test.img /mnt/ext4_test

# Create workload
cd /mnt/ext4_test
for i in {1..1000}; do
    echo "Data $i" > file$i
done

# 2. Simulate crash
# Method 1: Force power loss (VM environment)
echo 1 > /proc/sys/kernel/sysrq
echo b > /proc/sysrq-trigger

# Method 2: Emergency remount
mount -o remount,ro /mnt/ext4_test
# Then unplug (in VM)

# Method 3: Use fault injection
echo Y > /sys/module/jbd2/parameters/jbd2_debug
# Crash during journal commit

# 3. Recovery Analysis
# After reboot/crash
mount /dev/loop0 /mnt/ext4_test
dmesg | grep -i journal

# Check filesystem
umount /mnt/ext4_test
e2fsck -f /tmp/ext4_test.img

# Examine journal
debugfs -R "logdump -a" /tmp/ext4_test.img
```

### Defragmenting Ext4 filesystem
```bash
# Online defragmentation
e4defrag /path/to/file
e4defrag /mount/point  # Defrag entire FS

# Check fragmentation level
filefrag -v /path/to/file

# Kernel support
echo 1 > /proc/sys/fs/ext4/extent_max_zeroout_kb
```

### Tracing Quota subsystem using ftrace
```bash
# Enable quota tracepoints
echo 1 > /sys/kernel/debug/tracing/events/quota/enable

# Available tracepoints
ls /sys/kernel/debug/tracing/events/quota/
# dquot_acquire  dquot_alloc_space  dquot_free_space
# dquot_release  dquot_writeback_dquots

# Trace quota allocations
echo 'dquot_alloc_space' > /sys/kernel/debug/tracing/set_event

# Run quota-intensive workload
dd if=/dev/zero of=/mnt/test/file bs=1M count=100

# Analyze trace
cat /sys/kernel/debug/tracing/trace
```

### Quota Management and Diagnostics Tools
```bash
# Check quota usage
repquota -av
quota -v user

# Debug quota files
debugfs -R "stat quota.user" /dev/sda1

# Force quota sync
quotasync -av

# Quota check and repair
quotacheck -avugm
```

### Benchmark Journal Modes
#### Setup Environment
```bash
# Create test filesystem images
for mode in ordered writeback journal; do
    dd if=/dev/zero of=/tmp/ext4_${mode}.img bs=1M count=1024
    mkfs.ext4 -F /tmp/ext4_${mode}.img
    
    mkdir /mnt/${mode}
    mount -o loop,data=${mode} /tmp/ext4_${mode}.img /mnt/${mode}
done
```

#### Fio Benchmark
```bash
# Create fio job file (journal-test.fio)
cat > journal-test.fio << EOF
[global]
ioengine=sync
bs=4k
size=256M
directory=/mnt/\$mode
stonewall

[seqwrite]
rw=write
name=seqwrite

[randwrite]
rw=randwrite
name=randwrite

[fsync-test]
rw=write
bs=4k
size=16M
fsync=1
name=fsync-test
EOF

# Run for each mode
for mode in ordered writeback journal; do
    echo "=== Testing $mode mode ==="
    fio --section=seqwrite --section=randwrite --section=fsync-test \
        --mode=$mode journal-test.fio
done
```

#### Analysis
```bash
# Check fragmentation
for mode in ordered writeback journal; do
    echo "=== $mode fragmentation ==="
    filefrag /mnt/${mode}/seqwrite.*
done

# Clean up
umount /mnt/{ordered,writeback,journal}
```

### Setup Quota
```bash
# Create test filesystem with quota
dd if=/dev/zero of=/tmp/quota_test.img bs=1M count=512
mkfs.ext4 -F -O quota /tmp/quota_test.img

# Mount with quota
mkdir /mnt/quota
mount -o loop,usrquota,grpquota /tmp/quota_test.img /mnt/quota

# Initialize quota files
quotacheck -cug /mnt/quota
quotaon /mnt/quota

# Set limits
setquota -u testuser 100M 120M 1000 1200 /mnt/quota
```

#### Trace Quota Events
```bash
# Terminal 1: Trace quota events
trace-cmd record -e quota* sleep 30

# Terminal 2: Generate quota activity
sudo -u testuser bash -c "
cd /mnt/quota
for i in {1..200}; do
    dd if=/dev/zero of=file\$i bs=1M count=1 2>/dev/null
done
"

# Analyze trace
trace-cmd report | grep -E "alloc_space|exceeded"

# Debug specific issues
cat /sys/kernel/debug/tracing/trace | grep -B5 -A5 "EDQUOT"
```

### Quota Tools Reference
```bash
# Journal debugging
debugfs -R "logdump" /dev/sda1
dumpe2fs /dev/sda1 | grep -A10 "Journal"

# Quota tools
man quotactl
man setquota
man repquota

# Performance analysis
perf list | grep ext4
perf list | grep jbd2
```

### blktrace Analysis
```bash
# Install tools
apt-get install blktrace fio

# Create test filesystem
dd if=/dev/zero of=/tmp/ext4_test.img bs=1M count=1024
mkfs.ext4 -F /tmp/ext4_test.img
mount -o loop /tmp/ext4_test.img /mnt/test

# Start tracing
blktrace -d /dev/loop0 -o trace -w 30

# Terminal 1: Run workload
cd /mnt/test
fio --name=seqwrite --rw=write --bs=4k --size=256M
fio --name=randwrite --rw=randwrite --bs=4k --size=256M
fio --name=seqread --rw=read --bs=4k --size=256M

# Terminal 2: Analyze trace
blkparse -i trace -o trace.out
btt -i trace -o analysis/

```

### I/O Tracing using bpftrace
```perl
# Trace ext4 read/write calls
cat > ext4_trace.bt << EOF
kprobe:ext4_file_read_iter
{
    printf("read: %s size=%d\n", str(arg1->f_path.dentry->d_name.name), arg2->count);
}

kprobe:ext4_file_write_iter
{
    printf("write: %s size=%d\n", str(arg1->f_path.dentry->d_name.name), arg2->count);
}

kprobe:ext4_writepages
{
    printf("writeback: %s\n", str(((struct address_space *)arg0)->host->i_sb->s_id));
}
EOF

bpftrace ext4_trace.bt
```

### Buffered I/O vs Direct I/O Performance comparison
```bash
# Create test files
cd /mnt/test
fallocate -l 1G testfile

# Clear caches for accurate tests
echo 3 > /proc/sys/vm/drop_caches

# Buffered I/O Test
# Sequential write with buffered I/O
fio --name=buffered-write \
    --filename=testfile \
    --rw=write \
    --bs=1M \
    --size=512M \
    --direct=0 \
    --fsync=0 \
    --output=buffered-write.json \
    --output-format=json

# Sequential read with buffered I/O (warmed cache)
fio --name=buffered-read-warm \
    --filename=testfile \
    --rw=read \
    --bs=1M \
    --size=512M \
    --direct=0 \
    --output=buffered-read-warm.json

# Cold cache read
echo 3 > /proc/sys/vm/drop_caches
fio --name=buffered-read-cold \
    --filename=testfile \
    --rw=read \
    --bs=1M \
    --size=512M \
    --direct=0 \
    --output=buffered-read-cold.json

# Direct I/O write
fio --name=direct-write \
    --filename=testfile \
    --rw=write \
    --bs=1M \
    --size=512M \
    --direct=1 \
    --fsync=0 \
    --output=direct-write.json

# Direct I/O read
echo 3 > /proc/sys/vm/drop_caches
fio --name=direct-read \
    --filename=testfile \
    --rw=read \
    --bs=1M \
    --size=512M \
    --direct=1 \
    --output=direct-read.json
```

#### Analysis Script
```bash
#!/bin/bash
echo "=== Buffered Write ==="
jq '.jobs[0].write.bw' buffered-write.json
jq '.jobs[0].write.iops' buffered-write.json

echo "=== Direct Write ==="
jq '.jobs[0].write.bw' direct-write.json
jq '.jobs[0].write.iops' direct-write.json

echo "=== Buffered Read (Cold) ==="
jq '.jobs[0].read.bw' buffered-read-cold.json

echo "=== Buffered Read (Warm) ==="
jq '.jobs[0].read.bw' buffered-read-warm.json

echo "=== Direct Read ==="
jq '.jobs[0].read.bw' direct-read.json
```

### Memory Pressure Experiments
```bash
# Create large test file
dd if=/dev/zero of=/mnt/test/largefile bs=1M count=2048

# Monitor memory
watch -n 1 'cat /proc/meminfo | grep -E "^(Cached|Dirty|Writeback)"'

# 1. Fill page cache
cat /mnt/test/largefile > /dev/null
# Observe Cached memory increase

# 2. Modify file to create dirty pages
dd if=/dev/zero of=/mnt/test/largefile bs=1M count=1024 conv=notrunc
# Observe Dirty memory increase

# 3. Trigger writeback
sync
# Observe Dirty decrease, Writeback may spike

# 4. Drop caches
echo 1 > /proc/sys/vm/drop_caches  # Page cache
echo 2 > /proc/sys/vm/drop_caches  # Slab objects
echo 3 > /proc/sys/vm/drop_caches  # Both

# Use memory cgroup to limit available memory
mkdir /sys/fs/cgroup/memory/ext4_test
echo 100M > /sys/fs/cgroup/memory/ext4_test/memory.limit_in_bytes

# Run workload under memory pressure
echo $$ > /sys/fs/cgroup/memory/ext4_test/tasks
fio --name=mem-pressure --rw=write --bs=4k --size=200M --filename=/mnt/test/pressure

# Observe reclaim behavior
cat /sys/fs/cgroup/memory/ext4_test/memory.stat
```

### Trace Ext4 functions using perf
```bash
# Record ext4 events
perf record -e ext4:* -a -- sleep 10

# View report
perf report

# Sample ext4 function calls
perf record -e 'ext4:*' -a -g -- sleep 10
perf script | grep -E "(ext4_write|ext4_read)"

# Specific tracepoints
perf stat -e ext4:ext4_es_lookup_extent_enter,ext4:ext4_es_lookup_extent_exit \
         -a -- sleep 10
```

#### Flamegraph generation
```bash
# Record stack traces
perf record -F 99 -a -g -- sleep 30

# Generate flame graph
perf script | stackcollapse-perf.pl | flamegraph.pl > ext4_flame.svg

# Focus on ext4
perf report --sort comm,dso,symbol --graph=0.5
```

#### I/O Latency analysis
```bash
# Trace block I/O latency
perf record -e block:block_rq_issue,block:block_rq_complete -a -- sleep 10

# Calculate latency
perf script | awk '/block_rq_issue/ {start[$4]=$2} 
                   /block_rq_complete/ {if(start[$4]) 
                       print $4, ($2-start[$4])*1000}'
```

### tune2fs Options
```bash
# Filesystem tunables
tune2fs -l /dev/sda1                    # List parameters
tune2fs -c 30 /dev/sda1                 # Max mount count
tune2fs -i 30d /dev/sda1                # Interval between checks
tune2fs -m 2 /dev/sda1                  # Reserved blocks %
tune2fs -O ^metadata_csum /dev/sda1     # Disable feature
tune2fs -E stride=64 /dev/sda1          # RAID stride
```

### Workload-specific tuning

#### Database workloads
```bash
# Mount options
mount -o noatime,data=ordered,nobarrier,noauto_da_alloc /dev/sda1 /var/lib/mysql

# Filesystem creation
mkfs.ext4 -O extent,flex_bg -E stride=128,stripe_width=1024 /dev/sda1

# Kernel tuning
echo 0 > /proc/sys/vm/dirty_ratio
echo 10 > /proc/sys/vm/dirty_background_ratio
echo 500 > /proc/sys/vm/dirty_expire_centisecs
```

#### VM workloads
```bash
# Mount options
mount -o noatime,data=ordered,barrier=1 /dev/sda1 /var/lib/libvirt

# Preallocate disk images
fallocate -l 100G /var/lib/libvirt/images/vm.qcow2

# Enable huge pages for memory
echo 2048 > /proc/sys/vm/nr_hugepages
```

#### Backup workloads
```bash
# Mount options (fastest)
mount -o noatime,data=writeback,nobarrier /dev/sda1 /backup

# Larger journal for sequential writes
tune2fs -J size=2048 /dev/sda1

# Disable quota
mount -o noquota /dev/sda1 /backup
```

### Storage-specific tuning

#### HDD Optimization
```bash
# Align to 4K sectors
mkfs.ext4 -b 4096 -E stride=64,stripe_width=64 /dev/sda1

# Mount options
mount -o noatime,data=ordered,barrier=1 /dev/sda1 /mnt

# I/O scheduler
echo deadline > /sys/block/sda/queue/scheduler
echo 1024 > /sys/block/sda/queue/read_ahead_kb

# Avoid discards on HDD
mount -o nodiscard /dev/sda1 /mnt
```

#### SSD/NVMe specific optimization
```bash
# Align to 4K (already aligned)
mkfs.ext4 -E stride=1,stripe_width=1 /dev/nvme0n1p1

# Mount options
mount -o noatime,data=ordered,nobarrier,discard /dev/nvme0n1p1 /mnt

# I/O scheduler
echo none > /sys/block/nvme0n1/queue/scheduler  # NVMe
echo mq-deadline > /sys/block/sda/queue/scheduler  # SATA SSD

# Queue depth
echo 256 > /sys/block/nvme0n1/queue/nr_requests

# NVMe optimization
echo 0 > /sys/block/nvme0n1/queue/io_poll_delay
```

#### eMMC specific optimization
```bash
# eMMC-specific
mount -o noatime,data=ordered,nobarrier,noauto_da_alloc /dev/mmcblk0p1 /data

# Reduce journal size (eMMC limited)
tune2fs -J size=64 /dev/mmcblk0p1

# Periodic fstrim (not mount discard)
echo "0 2 * * * /sbin/fstrim -v /data" >> /etc/crontab
```

### NCQ Command Queuing
```bash
// NCQ allows reordering of I/O requests
// Optimizes HDD performance by reducing seek time

// Check NCQ status
cat /sys/block/sda/device/queue_depth
# Output: 32 (typical NCQ depth)

// Tuning for workload
echo 1 > /sys/block/sda/device/queue_depth  # Disable NCQ (for SSDs)
echo 31 > /sys/block/sda/device/queue_depth # Limit for HDD
```

### Mount Options Benchmarking
```bash
# Create test filesystem
dd if=/dev/zero of=/tmp/test.img bs=1M count=4096
mkfs.ext4 -F /tmp/test.img

# Prepare benchmark script
cat > bench_mount.sh << 'EOF'
#!/bin/bash
MOUNTS=(
    "defaults"
    "noatime,nodiratime"
    "data=writeback,nobarrier"
    "data=ordered,barrier=1"
    "data=journal"
)

for opts in "${MOUNTS[@]}"; do
    echo "Testing: $opts"
    mount -o loop,${opts//,/ } /tmp/test.img /mnt/test
    
    # Sequential write
    fio --name=seqwrite --filename=/mnt/test/file --rw=write \
        --bs=1M --size=1G --direct=0 --output-format=json \
        --output=/tmp/${opts//,/}_seq.json
    
    # Random write
    fio --name=randwrite --filename=/mnt/test/file --rw=randwrite \
        --bs=4k --size=512M --direct=0 --output-format=json \
        --output=/tmp/${opts//,/}_rand.json
    
    umount /mnt/test
done
EOF

chmod +x bench_mount.sh
./bench_mount.sh
```

### Tuning Journaling
```bash
# Create filesystems with different journal sizes
for size in 128 256 512 1024; do
    dd if=/dev/zero of=/tmp/journal_${size}.img bs=1M count=2048
    mkfs.ext4 -F -J size=$size /tmp/journal_${size}.img
done

# Benchmark with bursty writes
for img in /tmp/journal_*.img; do
    mount -o loop $img /mnt/test
    
    fio --name=burst --filename=/mnt/test/file --rw=write \
        --bs=4k --size=1G --iodepth=32 --numjobs=4 \
        --group_reporting --output-format=json \
        --output=/tmp/$(basename $img .img).json
    
    umount /mnt/test
done

# Compare results
ls -lh /tmp/journal_*.json
```

#### Discard Testing
```bash
# Test with discard mount option
mount -o loop,discard /tmp/test.img /mnt/test
fio --name=discard_test --filename=/mnt/test/file --rw=write \
    --bs=1M --size=2G --direct=1 --fsync=1
umount /mnt/test

# Test with fstrim
mount -o loop,nodiscard /tmp/test.img /mnt/test
fio --name=discard_test --filename=/mnt/test/file --rw=write \
    --bs=1M --size=2G --direct=1 --fsync=1
time fstrim -v /mnt/test
umount /mnt/test

# Monitor discard commands
blktrace -d /dev/loop0 -o discard_trace &
# Run workload
fstrim -v /mnt/test
blkparse -i discard_trace | grep DISCARD
```

#### Analyze behavious under memory pressure
```bash

# Create memory cgroup
mkdir /sys/fs/cgroup/memory/ext4_test
echo 256M > /sys/fs/cgroup/memory/ext4_test/memory.limit_in_bytes
echo 128M > /sys/fs/cgroup/memory/ext4_test/memory.memsw.limit_in_bytes

# Add current shell to cgroup
echo $$ > /sys/fs/cgroup/memory/ext4_test/tasks

# Create large file
dd if=/dev/zero of=/mnt/test/largefile bs=1M count=512

# Monitor memory while reading
watch -n 0.5 'cat /proc/meminfo | grep -E "(Cached|Dirty|Writeback|MemFree)"' &

# Generate memory pressure
cat /mnt/test/largefile > /dev/null &
while true; do
    dd if=/dev/zero of=/tmp/temp bs=1M count=100 2>/dev/null
    sleep 1
    rm -f /tmp/temp
done &

# Trace page reclaim
echo 1 > /sys/kernel/debug/tracing/events/vmscan/enable
cat /sys/kernel/debug/tracing/trace_pipe | grep -E "shrink|reclaim"

# Check shrinker activity
cat /proc/slabinfo | grep -E "(ext4_inode_cache|ext4_extent_status)"
cat /sys/fs/ext4/loop0/es_stats

# Monitor reclaim
vmstat 1
# Look for: si (swap in), so (swap out), wa (wait I/O)

```
