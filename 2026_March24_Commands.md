
### Analyze call-stack of `ext4_fill_super` using bpftrace
```
   bpftrace -l '*ext4_fill_super*'  # Check if this function can be traced
   bpftrace -e 'fentry:vmlinux:ext4_fill_super { @[kstack()] = count(); }'
   # Attach probe and count the number of times ext4_fill_super was 
   # invoked and print the kernel call-stack of the same with the count, 
   # when stopped with Ctrl-C. 
```

### Trace calls to jbd2 related functions using bpftrace
```
   bpftrace -l `jbd2:*`  # List all available tracepoints and events under jbd2

   bpftrace -lv 'tracepoint:jbd2:jbd2_start_commit` 
   # Print the arguments passed to the function

   bpftrace -e 'tracepoint:jbd2:* { @[kstack()] = count(); }'
   # Attach probes to all functions under jbd2 and trace their call-stacks
   # and number of times they were invoked.
```
### Analyze kernel data-structures using pahole
```
   pahole journal_s

   pahole fs_context

   pahole ext4_fs_context
   
   pahole ext4_super_block

   pahole ext4_dir_entry

   pahole ext4_inode

```



