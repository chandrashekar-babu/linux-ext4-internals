# Slide 1: Title Slide

## Linux Ext4 Internals: Architecture, Features, and Performance Tuning
### Day 4: Advanced Features & Performance

**Duration:** 3.5 hours
**Level:** Expert

*Instructor: Chandrashekar Babu*

Email: training@chandrashekar.info

*Website: https://www.chandrashekar.info/ | https://www.slashprog.com/*

---

# Slide 2: Day 4 Agenda

## What We'll Cover Today

### Module 7: Advanced Ext4 Features (1.5 hours)
- Bigalloc, metadata checksumming, project quotas
- Case-insensitive lookup and normalization
- Encryption (FBE/FDE) deep dive
- IOCTL interfaces
- Userspace tuning interfaces
- Android/Yocto extensions
- Garbage collection and TRIM/UNMAP

### Module 8: Performance Analysis & Tuning (1.5 hours)
- Mount options impact analysis
- Tuning parameters (commit, journal, inode cache)
- Workload-specific optimization
- Storage-specific tuning (eMMC, UFS, NVMe, HDD)
- Memory pressure and shrinker behavior
- Storage interactions (NCQ, flush latency, barriers)

### Hands-On Labs (30 minutes)
- Mount option benchmarking with fio
- Journal and discard tuning
- Memory stress analysis

**Break:** 10 minutes between modules

---

# Slide 3: Module 7 - Advanced Features Overview

## Beyond Basic Ext4

### Feature Evolution

```
Ext4 Feature Timeline:
┌─────────────────────────────────────────────────────┐
│ 2008: Initial release                               │
│   - Extents, delalloc, flex_bg                      │
├─────────────────────────────────────────────────────┤
│ 2010: Scalability features                          │
│   - 64-bit support, metadata checksums              │
├─────────────────────────────────────────────────────┤
│ 2015: Security features                             │
│   - Encryption (fscrypt), project quotas            │
├─────────────────────────────────────────────────────┤
│ 2020: Mobile/Embedded optimizations                 │
│   - Casefold, bigalloc, lazy initialization         │
└─────────────────────────────────────────────────────┘
```

### Feature Categories
| Category | Features |
|----------|----------|
| **Scalability** | Bigalloc, 64-bit, flex_bg |
| **Integrity** | Metadata checksums, journal checksums |
| **Security** | Encryption, project quotas |
| **Mobile** | Casefold, background trim, lazy init |
| **Management** | IOCTLs, sysfs interfaces |

---

# Slide 4: Bigalloc Feature

## Large Block Allocation

### Concept
```
Traditional (4KB blocks):
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ 4K │ 4K │ 4K │ 4K │ 4K │ 4K │ 4K │ 4K │
└────┴────┴────┴────┴────┴────┴────┴────┘
1MB file = 256 block allocations

Bigalloc (1MB clusters):
┌──────────────────────────┐
│         1MB Cluster      │
└──────────────────────────┘
1MB file = 1 cluster allocation
```

### Implementation
```c
// Bigalloc superblock fields
struct ext4_super_block {
    __le32  s_log_block_size;       /* Block size (4KB default) */
    __le32  s_log_cluster_size;     /* Cluster size (multiple of block) */
    __le32  s_clusters_per_group;   /* Clusters per group */
    __le32  s_cluster_bits;         /* Bits for cluster addressing */
};

// Cluster-aware allocation
static int ext4_bigalloc_alloc(struct inode *inode, 
                               struct ext4_map_blocks *map) {
    // Allocate entire clusters
    clusters = (map->m_len + cluster_size - 1) >> cluster_bits;
    
    // Find free clusters
    ret = ext4_mb_new_blocks(handle, &ar, &err);
    
    // Mark cluster used
    ext4_mb_mark_used(ac);
    
    return ret;
}
```

### Trade-offs
| Aspect | Benefit | Drawback |
|--------|---------|----------|
| **Metadata** | Less bitmap overhead | Internal fragmentation |
| **Performance** | Fewer allocations | Poor for small files |
| **SSD** | Better wear leveling | Wasted space |
| **Use Case** | Large files, media | Small files, databases |

### Configuration
```bash
# Create with bigalloc (1MB clusters)
mkfs.ext4 -O bigalloc -C 1048576 /dev/sda1

# Check cluster size
dumpe2fs /dev/sda1 | grep "Cluster size"
```

---

# Slide 5: Metadata Checksumming

## Integrity Protection

### Checksum Coverage
```
┌─────────────────────────────────────┐
│         Superblock                  │ ← CRC32c
├─────────────────────────────────────┤
│      Group Descriptors              │ ← CRC32c
├─────────────────────────────────────┤
│       Inode Table                   │ ← CRC32c per inode
├─────────────────────────────────────┤
│      Extent Tree                    │ ← CRC32c per node
├─────────────────────────────────────┤
│      Directory Entries              │ ← CRC32c per block
├─────────────────────────────────────┤
│      Extended Attributes            │ ← CRC32c per block
└─────────────────────────────────────┘
```

### Implementation
```c
// Superblock checksum
static __le32 ext4_superblock_csum(struct super_block *sb,
                                   struct ext4_super_block *es) {
    struct ext4_sb_info *sbi = EXT4_SB(sb);
    int offset = offsetof(struct ext4_super_block, s_checksum);
    
    return cpu_to_le32(ext4_chksum(sbi, ~0, (char *)es, offset));
}

// Inode checksum
__le32 ext4_inode_csum(struct inode *inode, struct ext4_inode *raw,
                       struct ext4_super_block *es) {
    struct ext4_sb_info *sbi = EXT4_SB(inode->i_sb);
    __u32 csum;
    
    // Calculate checksum over inode data
    csum = ext4_chksum(sbi, sbi->s_csum_seed, (__u8 *)raw,
                       EXT4_GOOD_OLD_INODE_SIZE);
    
    // Include extra fields if present
    if (EXT4_FITS_IN_INODE(raw, es, i_extra_isize)) {
        csum = ext4_chksum(sbi, csum, 
                          (__u8 *)raw + EXT4_GOOD_OLD_INODE_SIZE,
                          le16_to_cpu(raw->i_extra_isize));
    }
    
    return cpu_to_le32(csum);
}

// Verification
static int ext4_verify_inode_checksum(struct inode *inode,
                                      struct ext4_inode *raw) {
    __le32 provided = raw->i_checksum_lo;
    __le32 calculated = ext4_inode_csum(inode, raw, 
                                        EXT4_SB(inode->i_sb)->s_es);
    
    if (provided != calculated)
        return -EFSBADCRC;
    return 0;
}
```

### Enabling Checksums
```bash
# Create with metadata checksums
mkfs.ext4 -O metadata_csum /dev/sda1

# Convert existing filesystem
e2fsck -fD /dev/sda1
tune2fs -O metadata_csum /dev/sda1

# Verify checksumming
dumpe2fs -h /dev/sda1 | grep metadata_csum
```

---

# Slide 6: Project Quotas

## Hierarchical Quota Management

### Project Quota Concept
```
Project ID 1000 (Web Projects)
├── /var/www/project1/     ← Inherits ID 1000
├── /var/www/project2/     ← Inherits ID 1000
└── /var/www/project3/     ← Inherits ID 1000

Total space for Project ID 1000 = 10GB limit
```

### Implementation
```c
// Inode project ID
struct ext4_inode {
    __le32  i_projid;        /* Project ID (in extended fields) */
};

// Project quota limits
struct dquot {
    unsigned int dq_id;       /* Project ID */
    struct mem_dqblk dq_dqb;  /* Limits and usage */
};

// Set project ID on directory
int ext4_ioctl_setproject(struct inode *inode, __u32 projid) {
    struct ext4_inode_info *ei = EXT4_I(inode);
    
    // Set project ID on inode
    ei->i_projid = projid;
    
    // Mark inode dirty
    ext4_mark_inode_dirty(handle, inode);
    
    // Apply to all children (if recursive)
    if (flags & EXT4_PROJINHERIT_FL) {
        ext4_set_project_recursive(inode, projid);
    }
    
    return 0;
}
```

### Configuration
```bash
# Enable project quota
mount -o prjquota /dev/sda1 /mnt

# Set project ID on directory
chattr -p 1000 /mnt/project
lsattr -p /mnt/project

# Set quota limits
setquota -P 1000 10G 12G 0 0 /mnt

# Report project quotas
repquota -P /mnt

# Recursive project ID inheritance
mkdir /mnt/project/subdir
# subdir inherits project ID 1000
```

---

# Slide 7: Case-Insensitive Lookup

## Casefold Feature

### Normalization and Comparison

```c
// Casefold flags
#define EXT4_CASEFOLD_FL      0x01000000  /* Casefolded directory */

// Unicode normalization
struct utf8_data {
    __u8 *data;
    int len;
};

// Case-insensitive comparison
static int ext4_ci_compare(const struct inode *parent,
                           const struct qstr *name1,
                           const struct qstr *name2) {
    struct utf8_data u8name1, u8name2;
    int ret;
    
    // Normalize both names
    ret = utf8_normalize(&name1, &u8name1);
    ret = utf8_normalize(&name2, &u8name2);
    
    // Compare normalized strings
    return utf8_casefold_cmp(&u8name1, &u8name2);
}

// Lookup with casefold
static struct dentry *ext4_ci_lookup(struct inode *dir,
                                     struct dentry *dentry) {
    struct qstr *name = &dentry->d_name;
    struct ext4_dir_entry_2 *de;
    
    // Normalize search name
    utf8_normalize(name, &normalized);
    
    // Search directory
    de = ext4_find_entry(dir, &normalized, &bh);
    
    return d_splice_alias(inode, dentry);
}
```

### Configuration
```bash
# Create casefold directory
mkdir /mnt/data
chattr +F /mnt/data   # Set casefold flag

# Verify
lsattr /mnt/data
# Output: ----F------- /mnt/data

# Filesystem must support
mkfs.ext4 -O casefold /dev/sda1
mount -o casefold /dev/sda1 /mnt
```

---

# Slide 8: Encryption (FBE/FDE) Architecture

## fscrypt Framework

### Encryption Layers
```
┌─────────────────────────────────────┐
│         Application Layer           │
│         read() / write()            │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│         fscrypt Layer               │
│  - Policy management                │
│  - Key derivation                   │
│  - Crypto operations                │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│         Ext4 Layer                  │
│  - Encrypted inodes (EXT4_ENCRYPT_FL)│
│  - Encrypted blocks                 │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│         Crypto API                  │
│  - AES-256-XTS (contents)           │
│  - AES-256-CBC-CTS (filenames)      │
└─────────────────────────────────────┘
```

### Key Structures
```c
// Encryption policy (per directory)
struct fscrypt_policy {
    __u8 version;              /* Policy version */
    __u8 contents_encryption_mode;  /* AES-256-XTS */
    __u8 filenames_encryption_mode; /* AES-256-CBC-CTS */
    __u8 flags;                /* Policy flags */
    __u8 master_key_descriptor[FSCRYPT_KEY_DESCRIPTOR_SIZE];
};

// Per-inode encryption context
struct ext4_encryption_context {
    __u32 version;             /* Context version */
    __u32 contents_encryption_mode;
    __u32 filenames_encryption_mode;
    __u8 master_key_descriptor[FSCRYPT_KEY_DESCRIPTOR_SIZE];
    __u8 nonce[FSCRYPT_FILE_NONCE_SIZE];
} __packed;
```

---

# Slide 9: Encryption Code Walkthrough

## Kernel Implementation

### Inode Encryption Setup
```c
// Set encryption policy
int ext4_ioctl_set_encryption_policy(struct file *filp,
                                     const void __user *arg) {
    struct inode *inode = file_inode(filp);
    struct fscrypt_policy policy;
    
    // Copy policy from userspace
    if (copy_from_user(&policy, arg, sizeof(policy)))
        return -EFAULT;
    
    // Check directory is empty
    if (!ext4_empty_dir(inode))
        return -ENOTEMPTY;
    
    // Set encryption flag on inode
    ext4_set_inode_flag(inode, EXT4_INODE_ENCRYPT);
    
    // Save encryption context
    return fscrypt_set_context(inode, &policy);
}

// Read/write encryption
static int ext4_encrypted_read_iter(struct kiocb *iocb,
                                    struct iov_iter *iter) {
    struct inode *inode = file_inode(iocb->ki_filp);
    
    // Decrypt on read
    return fscrypt_decrypt_page(inode, page, len, off, lblk);
}

static int ext4_encrypted_write_iter(struct kiocb *iocb,
                                     struct iov_iter *iter) {
    struct inode *inode = file_inode(iocb->ki_filp);
    
    // Encrypt on write
    return fscrypt_encrypt_page(inode, page, len, off, lblk);
}
```

### Key Derivation
```c
// Per-file nonce + master key → file key
static int ext4_derive_file_key(struct inode *inode,
                                struct ext4_encryption_context *ctx,
                                u8 *key) {
    struct fscrypt_master_key *mk;
    u8 nonce[FSCRYPT_FILE_NONCE_SIZE];
    
    // Get master key from keyring
    mk = fscrypt_find_master_key(inode->i_sb, 
                                 ctx->master_key_descriptor);
    
    // Derive per-file key using HKDF
    return fscrypt_derive_key(mk, ctx->nonce, key, key_size);
}
```

---

# Slide 10: Encryption Debugging

## Troubleshooting Encryption Issues

### Keyring Debugging
```bash
# List keys in keyring
keyctl show
# Output:
# Session Keyring
#  467343211 --alswrv  0  0  keyring: _ses
#  123456789 --alswrv  0  0   \_ keyring: fscrypt

# Add encryption key
fscrypt unlock /mnt/encrypted
# Or manually
keyctl add logon fscrypt:abcdef1234567890 @s

# Check key status
keyctl list @s
```

### Policy Mismatch Tracing
```bash
# Enable fscrypt tracepoints
echo 1 > /sys/kernel/debug/tracing/events/fscrypt/enable

# Trace encryption operations
cat /sys/kernel/debug/tracing/trace_pipe &
fscrypt unlock /mnt/encrypted
ls /mnt/encrypted

# Sample output
# fscrypt: fscrypt_get_encryption_info: ino=12345 policy_version=1
# fscrypt: fscrypt_setup_key: ino=12345 master_key=abcdef12
# fscrypt: fscrypt_decrypt_page: ino=12345 block=100 size=4096
```

### Common Issues
```bash
# Check if inode has encryption flag
debugfs -R "stat /path/to/file" /dev/sda1 | grep flags
# Should show: Flags: 0x80000 (Encrypted)

# Verify encryption context
getfattr -n security.fscrypt /mnt/encrypted/file

# Clear and re-add key
fscrypt purge
fscrypt unlock /mnt/encrypted

# Check for policy mismatch
fscrypt status /mnt/encrypted
```

---

# Slide 11: IOCTL Interfaces

## Ext4-Specific IOCTLs

### Common IOCTL Operations
```c
// File flags
#define EXT4_IOC_GETFLAGS       _IOR('f', 1, long)
#define EXT4_IOC_SETFLAGS       _IOW('f', 2, long)

// Version handling
#define EXT4_IOC_GETVERSION     _IOR('f', 3, long)
#define EXT4_IOC_SETVERSION     _IOW('f', 4, long)

// Encryption
#define EXT4_IOC_SET_ENCRYPTION_POLICY   _IOW('f', 19, struct fscrypt_policy)
#define EXT4_IOC_GET_ENCRYPTION_POLICY   _IOR('f', 21, struct fscrypt_policy)

// Project ID
#define EXT4_IOC_GETPROJECT     _IOR('f', 27, __u32)
#define EXT4_IOC_SETPROJECT     _IOW('f', 28, __u32)

// Online resize
#define EXT4_IOC_RESIZE_FS      _IOW('f', 16, __u64)
```

### Implementation
```c
// GETFLAGS implementation
static int ext4_ioctl_getflags(struct inode *inode, void __user *arg) {
    unsigned int flags = 0;
    
    // Map internal flags to userspace
    if (EXT4_I(inode)->i_flags & EXT4_APPEND_FL)
        flags |= FS_APPEND_FL;
    if (EXT4_I(inode)->i_flags & EXT4_IMMUTABLE_FL)
        flags |= FS_IMMUTABLE_FL;
    if (EXT4_I(inode)->i_flags & EXT4_NOATIME_FL)
        flags |= FS_NOATIME_FL;
    if (EXT4_I(inode)->i_flags & EXT4_ENCRYPT_FL)
        flags |= FS_ENCRYPT_FL;
    if (EXT4_I(inode)->i_flags & EXT4_CASEFOLD_FL)
        flags |= FS_CASEFOLD_FL;
    
    return put_user(flags, (int __user *)arg);
}

// SETFLAGS implementation
static int ext4_ioctl_setflags(struct inode *inode, void __user *arg) {
    unsigned int flags;
    
    if (get_user(flags, (int __user *)arg))
        return -EFAULT;
    
    // Validate flags
    if (!capable(CAP_LINUX_IMMUTABLE)) {
        if (flags & (FS_APPEND_FL | FS_IMMUTABLE_FL))
            return -EPERM;
    }
    
    // Map and apply
    if (flags & FS_APPEND_FL)
        EXT4_I(inode)->i_flags |= EXT4_APPEND_FL;
    if (flags & FS_IMMUTABLE_FL)
        EXT4_I(inode)->i_flags |= EXT4_IMMUTABLE_FL;
    
    ext4_mark_inode_dirty(handle, inode);
    
    return 0;
}
```

### Userspace Usage
```c
#include <sys/ioctl.h>
#include <linux/fs.h>

int main() {
    int fd = open("/mnt/test/file", O_RDWR);
    int flags;
    
    // Get flags
    ioctl(fd, EXT4_IOC_GETFLAGS, &flags);
    printf("Flags: %x\n", flags);
    
    // Set immutable flag
    flags |= FS_IMMUTABLE_FL;
    ioctl(fd, EXT4_IOC_SETFLAGS, &flags);
    
    close(fd);
    return 0;
}
```

---

# Slide 12: Userspace Tuning Interfaces

## sysfs and procfs Interfaces

### /sys/fs/ext4 Interface
```bash
# Per-filesystem tunables
ls /sys/fs/ext4/sda1/
# inode_readahead_blks    mb_max_to_scan
# inode_goal              mb_min_to_scan
# journal_task            mb_stats
# lifetime_write_kbytes   mb_stream_req
# max_writeback_mb_bump   mb_order2_req

# Example tunings
echo 16 > /sys/fs/ext4/sda1/inode_readahead_blks
echo 256 > /sys/fs/ext4/sda1/mb_max_to_scan
echo 64 > /sys/fs/ext4/sda1/mb_min_to_scan
```

### /proc/fs/ext4 Interface
```bash
# Mount options
cat /proc/fs/ext4/sda1/options
# (no)acl, (no)user_xattr, (no)data=ordered, (no)journal_checksum

# Debug information
cat /proc/fs/ext4/sda1/mb_groups
# group:0 free:32768 frags:0
# group:1 free:31234 frags:2
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

---

# Slide 13: Android/Yocto Extensions

## Mobile and Embedded Optimizations

### Idle Maintenance
```c
// Background operations when device idle
static void ext4_idle_maintenance(struct super_block *sb) {
    struct ext4_sb_info *sbi = EXT4_SB(sb);
    
    // Check if idle
    if (time_after(jiffies, sbi->s_last_active + IDLE_TIMEOUT)) {
        // Perform background tasks
        ext4_trim_all_free(sb, 0, ULLONG_MAX, 0);
        ext4_run_lazyinit_thread(sb);
        ext4_free_orphan_inodes(sb);
    }
}
```

### Lazy Initialization
```c
// Background inode table initialization
static int ext4_lazyinit_thread(void *arg) {
    struct super_block *sb = arg;
    
    while (!kthread_should_stop()) {
        // Initialize uninitialized inode tables
        for (group = sbi->s_initialized_groups; 
             group < ngroups; group++) {
            if (ext4_group_needs_init(sb, group)) {
                ext4_init_inode_table(sb, group);
                break;
            }
        }
        
        // Sleep until more work needed
        schedule_timeout_interruptible(LAZYINIT_TIMEOUT);
    }
}
```

### Background TRIM
```c
// Periodic background discard
static void ext4_background_trim(struct work_struct *work) {
    struct ext4_sb_info *sbi = container_of(work, 
                                           struct ext4_sb_info,
                                           s_trim_work);
    
    // Trim free space
    ext4_trim_all_free(sbi->s_sb, 0, ULLONG_MAX, 0);
    
    // Reschedule
    schedule_delayed_work(&sbi->s_trim_work, 
                          TRIM_INTERVAL * HZ);
}
```

### Android-Specific Settings
```bash
# Typical Android Ext4 mount options
mount -t ext4 -o noatime,nodiratime,noauto_da_alloc,data=ordered \
    /dev/block/by-name/userdata /data

# F2FS-like optimizations
echo 0 > /sys/fs/ext4/loop0/mb_group_prealloc
echo 512 > /sys/fs/ext4/loop0/inode_readahead_blks
```

---

# Slide 14: Garbage Collection & Discard (TRIM/UNMAP)

## TRIM Implementation

### Synchronous vs Asynchronous Discard

```c
// Synchronous discard (on free)
static void ext4_sync_discard(struct super_block *sb, 
                              ext4_grpblk_t start,
                              ext4_grpblk_t len) {
    struct block_device *bdev = sb->s_bdev;
    sector_t start_sector = (sector_t)start << (sb->s_blocksize_bits - 9);
    sector_t nr_sectors = (sector_t)len << (sb->s_blocksize_bits - 9);
    
    // Issue discard synchronously
    blkdev_issue_discard(bdev, start_sector, nr_sectors, GFP_NOFS, 0);
}

// Asynchronous discard (queued)
static void ext4_async_discard(struct super_block *sb,
                               ext4_grpblk_t start,
                               ext4_grpblk_t len) {
    struct bio *bio;
    
    // Create async bio
    bio = bio_alloc(GFP_NOFS, 1);
    bio->bi_iter.bi_sector = start_sector;
    bio_set_op_attrs(bio, REQ_OP_DISCARD, 0);
    
    // Submit without waiting
    submit_bio(bio);
}
```

### Mount Options
```bash
# Synchronous discard (every free)
mount -o discard /dev/sda1 /mnt

# Asynchronous (requires fstrim)
mount -o nodiscard /dev/sda1 /mnt

# Batch discard
fstrim -v /mnt
fstrim -v -o 1G -l 10G /mnt  # Range discard
```

---

# Slide 15: TRIM Optimization Strategies

## eMMC/UFS Tuning

### eMMC Considerations
```c
// eMMC has limited TRIM queue depth
static int ext4_trim_emmc(struct super_block *sb,
                          ext4_grpblk_t start,
                          ext4_grpblk_t len) {
    // Small batches for eMMC
    while (remaining > 0) {
        batch = min(remaining, 256);  // 256 blocks max per batch
        blkdev_issue_discard(bdev, sector, batch * blocksize, 
                             GFP_NOFS, 0);
        msleep(10);  // Allow eMMC to process
        remaining -= batch;
    }
}
```

### UFS Optimizations
```c
// UFS supports larger discard commands
static int ext4_trim_ufs(struct super_block *sb,
                         ext4_grpblk_t start,
                         ext4_grpblk_t len) {
    // UFS can handle larger batches
    blkdev_issue_discard(bdev, sector, len * blocksize, 
                         GFP_NOFS, 0);
}
```

### Performance Comparison
| Device | Sync Discard | Async Discard | fstrim |
|--------|--------------|---------------|--------|
| **HDD** | Poor | N/A | Good |
| **SSD** | Good | Better | Best |
| **eMMC** | Poor | Good | Best |
| **UFS** | Good | Better | Best |

### Best Practices
```bash
# For eMMC/UFS - use periodic fstrim
echo "0 3 * * * root /sbin/fstrim -v /data" >> /etc/crontab

# For high-end SSD - discard mount option may be fine
mount -o discard,noatime /dev/nvme0n1 /mnt

# Check discard support
cat /sys/block/sda/queue/discard_granularity
cat /sys/block/sda/queue/discard_max_bytes
```

---

# Slide 16: Module 7 Summary

## Advanced Features Key Takeaways

### Bigalloc
- Reduces metadata for large files
- 1MB clusters vs 4KB blocks
- Trade-off: internal fragmentation

### Metadata Checksums
- CRC32c protection for all metadata
- Detection of corruption early
- Enables self-healing

### Project Quotas
- Hierarchical limits across directories
- Container/tenant isolation
- Combined with user/group quotas

### Casefold
- Case-insensitive lookups
- Unicode normalization
- Android compatibility

### Encryption
- fscrypt framework
- Per-file keys from master key
- Contents and filename encryption

### Mobile Optimizations
- Lazy initialization
- Background operations
- TRIM scheduling

### TRIM/Discard
- Synchronous: per-free operation
- Asynchronous: batched fstrim
- Device-specific tuning

---

# Slide 17: Module 8 - Performance Analysis & Tuning

## Performance Tuning Framework

### Tuning Areas
```
┌─────────────────────────────────────────────────┐
│            Mount Options                        │
│  noatime, data=ordered, barrier, discard       │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│            Journal Tuning                       │
│  commit interval, journal size, checksums       │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│            Memory Management                    │
│  inode cache, page cache, shrinker              │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│            Storage Layer                        │
│  NCQ, I/O scheduler, device alignment           │
└─────────────────────────────────────────────────┘
```

---

# Slide 18: Mount Options Performance Impact

## Comprehensive Analysis

### Atime Options
```bash
# Benchmark results (fio random read)
# noatime:     100% baseline
# relatime:    98% of baseline
# strictatime: 85% of baseline

# Recommendation: use noatime for most workloads
mount -o noatime,nodiratime /dev/sda1 /mnt
```

### Data Journaling Modes
```bash
# Performance ranking (write throughput)
# writeback:  100% (fastest)
# ordered:    85% (default)
# journal:    60% (slowest, safest)

# Recommendation by workload
# - Database: ordered or journal
# - Backup: writeback
# - General: ordered
```

### Barrier Effects
```bash
# barrier=1: 100% (safe)
# barrier=0: 115-140% (risk of corruption)

# When to disable barriers:
# - Battery-backed RAID controller
# - UPS-protected systems
# - Testing environments
```

### Discard Options
```bash
# discard mount: -5-10% performance
# fstrim weekly: 0% runtime impact
# Recommendation: Use fstrim for SSDs
```

---

# Slide 19: Key Mount Flags Reference

## Complete Mount Option Matrix

| Option | Category | Performance Impact | Safety | Recommended For |
|--------|----------|-------------------|--------|-----------------|
| `noatime` | Atime | +5-15% | Safe | All systems |
| `nodiratime` | Atime | +2-5% | Safe | All systems |
| `relatime` | Atime | 0% | Safe | Default |
| `data=writeback` | Journal | +10-20% | Low | Backup, scratch |
| `data=ordered` | Journal | 0% | Medium | Default, general |
| `data=journal` | Journal | -30-40% | High | Critical data |
| `barrier=0` | Barrier | +15-40% | Low | Battery-backed |
| `nobarrier` | Barrier | +15-40% | Low | Same as barrier=0 |
| `discard` | TRIM | -5-10% | Safe | fstrim better |
| `nodiscard` | TRIM | 0% | Safe | Use fstrim |
| `noauto_da_alloc` | Delalloc | +2-5% | Medium | Database |
| `errors=remount-ro` | Recovery | 0% | High | Critical |
| `commit=60` | Journal | +5-10% | Medium | Batch workloads |

---

# Slide 20: Tuning Parameters Deep Dive

## Journal and Commit Tuning

### Commit Interval
```c
// Default: 5 seconds
// Tuned: commit=60 (60 seconds)

// Impact:
// - Longer commit = better write coalescing
// - Longer commit = more data at risk on crash
// - Longer commit = less journal overhead

// Code path
static void ext4_commit_timer(struct timer_list *t) {
    struct ext4_sb_info *sbi = from_timer(sbi, t, s_commit_timer);
    
    if (jbd2_journal_start_commit(sbi->s_journal, &tid)) {
        // Commit started
    }
    
    // Reschedule
    mod_timer(&sbi->s_commit_timer,
              jiffies + sbi->s_commit_interval * HZ);
}
```

### Journal Size
```bash
# Create with custom journal size
mkfs.ext4 -J size=400 /dev/sda1   # 400MB journal

# Default: 128MB for 4TB filesystem
# Rule of thumb: 1GB journal per 10TB data
# Larger journal = better for bursty writes

# Check current size
dumpe2fs /dev/sda1 | grep "Journal size"
```

### Inode Cache Tuning
```bash
# System-wide inode limits
echo 500000 > /proc/sys/fs/inode-max
echo 250000 > /proc/sys/fs/inode-nr

# Per-filesystem inode readahead
echo 16 > /sys/fs/ext4/sda1/inode_readahead_blks

# Inode goal (allocation preference)
echo 0 > /sys/fs/ext4/sda1/inode_goal
```

---

# Slide 21: Workload-Specific Optimization

## Database Workloads

### MySQL/PostgreSQL Optimization
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

### VM Workloads
```bash
# Mount options
mount -o noatime,data=ordered,barrier=1 /dev/sda1 /var/lib/libvirt

# Preallocate disk images
fallocate -l 100G /var/lib/libvirt/images/vm.qcow2

# Enable huge pages for memory
echo 2048 > /proc/sys/vm/nr_hugepages
```

### Backup Workloads
```bash
# Mount options (fastest)
mount -o noatime,data=writeback,nobarrier /dev/sda1 /backup

# Larger journal for sequential writes
tune2fs -J size=2048 /dev/sda1

# Disable quota
mount -o noquota /dev/sda1 /backup
```

---

# Slide 22: Storage-Specific Tuning

## HDD Optimization

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

## SSD/NVMe Optimization

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

## eMMC/UFS Optimization

```bash
# eMMC-specific
mount -o noatime,data=ordered,nobarrier,noauto_da_alloc /dev/mmcblk0p1 /data

# Reduce journal size (eMMC limited)
tune2fs -J size=64 /dev/mmcblk0p1

# Periodic fstrim (not mount discard)
echo "0 2 * * * /sbin/fstrim -v /data" >> /etc/crontab
```

---

# Slide 23: Crypto Performance Considerations

## Encryption Overhead

### Performance Impact
| Workload | No Encryption | AES-256-XTS | Overhead |
|----------|---------------|-------------|----------|
| Sequential Read | 100% | 85-90% | 10-15% |
| Sequential Write | 100% | 80-85% | 15-20% |
| Random Read | 100% | 70-80% | 20-30% |
| Random Write | 100% | 65-75% | 25-35% |

### Optimization Techniques
```bash
# Use hardware acceleration
cat /proc/cpuinfo | grep aes  # Check AES-NI support

# Kernel crypto configuration
CONFIG_CRYPTO_AES_NI_INTEL=y
CONFIG_CRYPTO_AES_X86_64=y

# Mount with crypto optimization
mount -o noatime,data=ordered /dev/sda1 /mnt/encrypted

# Benchmark crypto performance
cryptsetup benchmark
```

### CPU Affinity
```bash
# Pin crypto work to specific cores
taskset -c 0-3 fio --name=crypto-test --rw=randwrite --bs=4k --size=1G

# IRQ affinity for NVMe with encryption
echo 1 > /proc/irq/42/smp_affinity  # Bind to CPU 0
```

---

# Slide 24: Underlying Storage Interactions

## NCQ (Native Command Queuing)

```c
// NCQ allows reordering of I/O requests
// Optimizes HDD performance by reducing seek time

// Check NCQ status
cat /sys/block/sda/device/queue_depth
# Output: 32 (typical NCQ depth)

// Tuning for workload
echo 1 > /sys/block/sda/device/queue_depth  # Disable NCQ (for SSDs)
echo 31 > /sys/block/sda/device/queue_depth # Limit for HDD
```

### I/O Barriers and Flush Latency
```c
// Flush latency measurement
static void measure_flush_latency(struct block_device *bdev) {
    ktime_t start, end;
    
    start = ktime_get();
    blkdev_issue_flush(bdev, GFP_NOFS);
    end = ktime_get();
    
    latency = ktime_to_us(ktime_sub(end, start));
    printk("Flush latency: %lld us\n", latency);
}

// Typical values:
// HDD: 10-20ms
// SATA SSD: 0.1-1ms
// NVMe: 0.01-0.1ms
```

### I/O Scheduler Selection
```bash
# Check available schedulers
cat /sys/block/sda/queue/scheduler
# [mq-deadline] kyber bfq none

# Recommendations:
# - HDD: mq-deadline or bfq
# - SSD: mq-deadline or none
# - NVMe: none (noop)
# - eMMC: mq-deadline
```

---

# Slide 25: Memory Pressure and Shrinker Behavior

## Understanding Shrinkers

### Ext4 Shrinker Registration
```c
// Extent status cache shrinker
static struct shrinker ext4_es_shrinker = {
    .count_objects = ext4_es_count_objects,
    .scan_objects = ext4_es_scan_objects,
    .seeks = DEFAULT_SEEKS,
};

// Called under memory pressure
static unsigned long ext4_es_scan_objects(struct shrinker *shrink,
                                          struct shrink_control *sc) {
    struct ext4_sb_info *sbi = container_of(shrink, 
                                           struct ext4_sb_info,
                                           s_es_shrinker);
    
    // Number of entries to scan
    nr_to_scan = sc->nr_to_scan;
    
    // Free extent status entries
    while (nr_to_scan-- && ext4_es_free_extent(sbi, 1))
        ;
    
    // Return remaining count
    return ext4_es_count(sbi);
}
```

### Monitoring Memory Pressure
```bash
# Watch memory pressure
watch -n 1 'cat /proc/meminfo | grep -E "(Dirty|Writeback|Cached|MemFree)"'

# Extent status cache usage
cat /sys/fs/ext4/sda1/es_stats
# es_stats_shrunk: 12345
# es_stats_cache_hits: 987654
# es_stats_cache_misses: 12345

# Force reclaim
echo 2 > /proc/sys/vm/drop_caches
echo 3 > /proc/sys/vm/drop_caches  # More aggressive
```

---

# Slide 26: Performance Monitoring Tools

## Real-time Monitoring

### iostat
```bash
# Detailed I/O statistics
iostat -x 1 /dev/sda1

# Key metrics:
# %util: Device utilization (100% = saturation)
# await: Average I/O latency
# svctm: Average service time
# r/s, w/s: Reads/writes per second
```

### blktrace + btt
```bash
# Trace and analyze I/O
blktrace -d /dev/sda -w 10
blkparse -i sda.blktrace -o sda.out
btt -i sda.blktrace -o sda_btt/

# Analyze Q2C (queue to completion) latency
cat sda_btt/sda_btt_avg.dat
```

### perf stat
```bash
# Count ext4 events
perf stat -e ext4:*,jbd2:* -a sleep 10

# Sample output:
# 123,456 ext4:ext4_es_lookup_extent_enter
# 120,234 ext4:ext4_es_lookup_extent_exit
# 3,456 jbd2:jbd2_commit
```

---

# Slide 27: Hands-On Lab 1 - Mount Option Benchmarking

## Objective: Measure impact of mount options

### Setup
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

### Analysis
```bash
# Compare results
for mode in defaults noatime writeback ordered journal; do
    echo "=== $mode ==="
    jq '.jobs[0].write.bw' /tmp/${mode}_seq.json
    jq '.jobs[0].write.iops' /tmp/${mode}_rand.json
done
```

---

# Slide 28: Hands-On Lab 2 - Journal and Discard Tuning

## Objective: Optimize journal and discard settings

### Journal Tuning
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

### Discard Testing
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

---

# Slide 29: Hands-On Lab 3 - Memory Stress Analysis

## Objective: Analyze behavior under memory pressure

### Setup Memory Cgroup
```bash
# Create memory cgroup
mkdir /sys/fs/cgroup/memory/ext4_test
echo 256M > /sys/fs/cgroup/memory/ext4_test/memory.limit_in_bytes
echo 128M > /sys/fs/cgroup/memory/ext4_test/memory.memsw.limit_in_bytes

# Add current shell to cgroup
echo $$ > /sys/fs/cgroup/memory/ext4_test/tasks
```

### Memory Pressure Test
```bash
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
```

### Analysis Script
```bash
# Check shrinker activity
cat /proc/slabinfo | grep -E "(ext4_inode_cache|ext4_extent_status)"
cat /sys/fs/ext4/loop0/es_stats

# Monitor reclaim
vmstat 1
# Look for: si (swap in), so (swap out), wa (wait I/O)
```

---

# Slide 30: Performance Checklist

## Optimization Decision Tree

### Step 1: Identify Workload
```
┌─────────────────────────────────┐
│ What is your primary workload? │
├─────────────────────────────────┤
│ • Database (random I/O)        │ → Use noauto_da_alloc
│ • Streaming (sequential)       │ → Large journal, writeback
│ • Virtualization               │ → Barrier=1, preallocation
│ • Backup (write once)          │ → Writeback, nobarrier
│ • General purpose              │ → Ordered, noatime
└─────────────────────────────────┘
```

### Step 2: Characterize Storage
```
┌─────────────────────────────────┐
│ What storage device?           │
├─────────────────────────────────┤
│ • HDD          │ → Deadline, stride tuning
│ • SATA SSD     │ → mq-deadline, fstrim
│ • NVMe         │ → none scheduler, huge pages
│ • eMMC/UFS     │ → Periodic fstrim, small journal
└─────────────────────────────────┘
```

### Step 3: Apply Mount Options
```
Always:
  noatime, nodiratime

For performance:
  data=writeback, nobarrier

For safety:
  data=ordered, barrier=1, errors=remount-ro

For SSDs:
  Use fstrim (not discard mount option)
```

---

# Slide 31: Performance Tuning Summary

## Quick Reference Guide

### Recommended Settings by Use Case

| Use Case | Mount Options | Journal | Discard |
|----------|---------------|---------|---------|
| **Database** | noatime,data=ordered,noauto_da_alloc | 512MB+ | fstrim weekly |
| **Web Server** | noatime,data=ordered | 256MB | fstrim monthly |
| **Backup** | noatime,data=writeback,nobarrier | 1GB+ | None |
| **VM Storage** | noatime,data=ordered,barrier=1 | 512MB | fstrim weekly |
| **Desktop** | defaults,noatime | Default | fstrim monthly |
| **Embedded** | noatime,data=ordered,nobarrier | 64MB | fstrim on idle |

### Critical Tuning Parameters
```bash
# System-wide
echo 10 > /proc/sys/vm/dirty_background_ratio
echo 20 > /proc/sys/vm/dirty_ratio
echo 500 > /proc/sys/vm/dirty_expire_centisecs

# Filesystem-specific
echo 64 > /sys/fs/ext4/sda1/mb_min_to_scan
echo 512 > /sys/fs/ext4/sda1/mb_max_to_scan
echo 16 > /sys/fs/ext4/sda1/inode_readahead_blks
```

---

# Slide 32: Module 8 Summary

## Performance Tuning Key Takeaways

### Mount Options
- **noatime**: Single biggest performance gain
- **data=ordered**: Best balance for most workloads
- **barrier**: Disable only with battery-backed cache
- **discard**: Use fstrim instead of mount option

### Journal Tuning
- **commit interval**: Balance performance vs safety
- **journal size**: Larger for bursty writes
- **checksums**: Enable for data integrity

### Storage Tuning
- **I/O scheduler**: HDD→deadline, SSD→none, NVMe→none
- **Alignment**: 4K sector alignment critical
- **NCQ**: Depth tuning for workload

### Memory Management
- **Shrinkers**: Monitor extent cache pressure
- **Dirty ratios**: Tune for write-heavy workloads
- **Page cache**: Balance with application memory

### Workload Optimization
- **Database**: Disable delalloc, use ordered
- **Backup**: Writeback mode, large journal
- **Virtualization**: Preallocate, barrier=1

---

# Slide 33: Day 4 Q&A

## Discussion Questions

### Advanced Features
1. **Encryption trade-offs**: When is encryption overhead acceptable?
2. **Bigalloc use cases**: When does cluster allocation make sense?
3. **Casefold impact**: What's the performance cost of case-insensitivity?

### Performance Tuning
1. **Mount options**: Which combination gives best performance for your workload?
2. **Journal size**: How to determine optimal journal size?
3. **Discard strategy**: sync vs async vs fstrim?

### Storage-Specific
1. **NVMe optimization**: What changes from SATA SSD tuning?
2. **eMMC limitations**: How to handle limited TRIM queue depth?
3. **RAID considerations**: How does stripe width affect allocation?

### Common Pitfalls
- **Over-tuning**: Too many options can hurt
- **Discard storm**: Sync discard causing performance issues
- **Journal too small**: Frequent checkpoints
- **Memory pressure**: Page cache vs application memory

---

# Slide 34: Preview of Day 5

## Coming Up Tomorrow

### Module 9: Debugging and Recovery Tools
- debugfs and e2fsck internals
- Kernel oops/crash debugging
- eBPF and tracepoint analysis
- Recovery from corruption

### Module 10: Maintenance and Future Development
- Online resize and maintenance
- Backup and restore strategies
- Development workflow
- Future features (compression, subvolumes)

### Module 11: Summary & Expert Discussion
- Deployment best practices
- Performance checklist
- Community resources
- Final Q&A

### Hands-On Labs
- Corruption recovery simulation
- Patch submission workflow
- Crash analysis

---

# Slide 35: Additional Resources

## References and Further Reading

### Kernel Documentation
- `Documentation/filesystems/ext4/`
- `Documentation/admin-guide/ext4.rst`
- `Documentation/crypto/fscrypt.rst`

### Source Files to Study
- `fs/ext4/ioctl.c` - IOCTL implementations
- `fs/ext4/crypto.c` - Encryption code
- `fs/ext4/mballoc.c` - Allocation and discard
- `fs/ext4/super.c` - Mount options
- `include/linux/fscrypt.h` - Encryption framework

### Tools Reference
```bash
# Performance analysis
man fio
man blktrace
man perf

# Filesystem tools
man tune2fs
man debugfs
man e2fsck

# Encryption tools
man fscrypt
man cryptsetup

# Monitoring
man iostat
man vmstat
```

### Online Resources
- **Ext4 Wiki**: https://ext4.wiki.kernel.org/
- **fscrypt Documentation**: https://www.kernel.org/doc/html/latest/filesystems/fscrypt.html
- **Linux Storage Wiki**: https://wiki.linuxfoundation.org/storage/start

---

**End of Day 4 Presentation**

*Questions?*