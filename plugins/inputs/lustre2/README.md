# Lustre Input Plugin

The [Lustre][]® file system is an open-source, parallel file system that
supports many requirements of leadership class HPC simulation environments.

This plugin monitors the Lustre file system using its entries in the proc
filesystem.

## Global configuration options <!-- @/docs/includes/plugin_config.md -->

In addition to the plugin-specific configuration settings, plugins support
additional global and plugin configuration settings. These settings are used to
modify metrics, tags, and field or create aliases and configure ordering, etc.
See the [CONFIGURATION.md][CONFIGURATION.md] for more details.

[CONFIGURATION.md]: ../../../docs/CONFIGURATION.md#plugins

## Configuration

```toml @sample.conf
# Read metrics from local Lustre service on OST, MDS
# This plugin ONLY supports Linux
[[inputs.lustre2]]
  ## An array of /proc globs to search for Lustre stats
  ## If not specified, the default will work on Lustre 2.5.x
  ##
  # ost_procfiles = [
  #   "/proc/fs/lustre/obdfilter/*/stats",
  #   "/proc/fs/lustre/osd-ldiskfs/*/stats",
  #   "/proc/fs/lustre/obdfilter/*/job_stats",
  #   "/proc/fs/lustre/obdfilter/*/exports/*/stats",
  # ]
  # mds_procfiles = [
  #   "/proc/fs/lustre/mdt/*/md_stats",
  #   "/proc/fs/lustre/mdt/*/job_stats",
  #   "/proc/fs/lustre/mdt/*/exports/*/stats",
  # ]
  # lnet_procfiles = [
  #   "/sys/kernel/debug/lnet/stats",
  # ]
```

## Optional Configuration

In order to read lnet metrics present in debugfs a local systemd module change
is needed to have correct permissions for telegraf process user without running
telegraf as root or completely opening debugfs.

Add a local systemd customization `/etc/systemd/system/sys-kernel-debug.mount`
that adds debugfs mount options that make debugfs mount with gid of telegraf
group id and mode g+rx for the debugfs mountpoint under /sys/kernel.

```
[Unit]
Description=Kernel Debug File System
Documentation=https://www.kernel.org/doc/Documentation/filesystems/debugfs.txt
Documentation=https://www.freedesktop.org/wiki/Software/systemd/APIFileSystems
DefaultDependencies=no
ConditionPathExists=/sys/kernel/debug
ConditionCapability=CAP_SYS_RAWIO
Before=sysinit.target

[Mount]
What=debugfs
Where=/sys/kernel/debug
Type=debugfs
Options=uid=0,gid=XXXX,mode=0750
```
where the value of `gid=XXXX` is the output of `id -g telegraf`

note, any creation of or modification to a systemd module file requires running
`systemctl daemon-reload` and if currently running, a restart of the module
`systemctl restart sys-kernel-debug.mount` to take effect.

## Metrics

From `/proc/fs/lustre/obdfilter/*/stats` and
`/proc/fs/lustre/osd-ldiskfs/*/stats`:

- lustre2
  - tags:
    - name
  - fields:
    - write_bytes
    - write_calls
    - read_bytes
    - read_calls
    - cache_hit
    - cache_miss
    - cache_access

From `/proc/fs/lustre/obdfilter/*/exports/*/stats`:

- lustre2
  - tags:
    - name
    - client
  - fields:
    - write_bytes
    - write_calls
    - read_bytes
    - read_calls

From `/proc/fs/lustre/obdfilter/*/job_stats`:

- lustre2
  - tags:
    - name
    - jobid
  - fields:
    - jobstats_ost_getattr
    - jobstats_ost_setattr
    - jobstats_ost_sync
    - jobstats_punch
    - jobstats_destroy
    - jobstats_create
    - jobstats_ost_statfs
    - jobstats_get_info
    - jobstats_set_info
    - jobstats_quotactl
    - jobstats_read_bytes
    - jobstats_read_calls
    - jobstats_read_max_size
    - jobstats_read_min_size
    - jobstats_write_bytes
    - jobstats_write_calls
    - jobstats_write_max_size
    - jobstats_write_min_size

From `/proc/fs/lustre/mdt/*/md_stats`:

- lustre2
  - tags:
    - name
  - fields:
    - open
    - close
    - mknod
    - link
    - unlink
    - mkdir
    - rmdir
    - rename
    - getattr
    - setattr
    - getxattr
    - setxattr
    - statfs
    - sync
    - samedir_rename
    - crossdir_rename

From `/proc/fs/lustre/mdt/*/exports/*/stats`:

- lustre2
  - tags:
    - name
    - client
  - fields:
    - open
    - close
    - mknod
    - link
    - unlink
    - mkdir
    - rmdir
    - rename
    - getattr
    - setattr
    - getxattr
    - setxattr
    - statfs
    - sync
    - samedir_rename
    - crossdir_rename

From `/proc/fs/lustre/mdt/*/job_stats`:

- lustre2
  - tags:
    - name
    - jobid
  - fields:
    - jobstats_close
    - jobstats_crossdir_rename
    - jobstats_getattr
    - jobstats_getxattr
    - jobstats_link
    - jobstats_mkdir
    - jobstats_mknod
    - jobstats_open
    - jobstats_rename
    - jobstats_rmdir
    - jobstats_samedir_rename
    - jobstats_setattr
    - jobstats_setxattr
    - jobstats_statfs
    - jobstats_sync
    - jobstats_unlink

From `/sys/kernel/debug/lnet/stats`:

The below lnet metrics fields are renamed with prefix 'lnet_' using ReportAs option in lustre2.go 
This is done for clarity, to differentiate lnet metrics from obdfilter based metrics.

- lustre2
  - fields:
    - msgs_alloc     #  renamed to lnet_msgs_alloc
    - msgs_max       #  renamed to lnet_msgs_max
    - rst_alloc      #  renamed to lnet_rst_alloc
    - send_count     #  renamed to lnet_send_count
    - recv_count     #  renamed to lnet_recv_count
    - route_count    #  renamed to lnet_route_count
    - drop_count     #  renamed to lnet_drop_count
    - send_length    #  renamed to lnet_send_length
    - recv_length    #  renamed to lnet_recv_length
    - route_length   #  renamed to lnet_route_length
    - drop_length    #  renamed to lnet_drop_length

## Troubleshooting

Check for the default or custom procfiles in the proc filesystem, and reference
the [Lustre Monitoring and Statistics Guide][guide].  This plugin does not
report all information from these files, only a limited set of items
corresponding to the above metric fields.

## Example Output

```text
lustre2,host=oss2,jobid=42990218,name=wrk-OST0041 jobstats_ost_setattr=0i,jobstats_ost_sync=0i,jobstats_punch=0i,jobstats_read_bytes=4096i,jobstats_read_calls=1i,jobstats_read_max_size=4096i,jobstats_read_min_size=4096i,jobstats_write_bytes=310206488i,jobstats_write_calls=7423i,jobstats_write_max_size=53048i,jobstats_write_min_size=8820i 1556525847000000000
lustre2,host=mds1,jobid=42992017,name=wrk-MDT0000 jobstats_close=31798i,jobstats_crossdir_rename=0i,jobstats_getattr=34146i,jobstats_getxattr=15i,jobstats_link=0i,jobstats_mkdir=658i,jobstats_mknod=0i,jobstats_open=31797i,jobstats_rename=0i,jobstats_rmdir=0i,jobstats_samedir_rename=0i,jobstats_setattr=1788i,jobstats_setxattr=0i,jobstats_statfs=0i,jobstats_sync=0i,jobstats_unlink=0i 1556525828000000000
lustre2,host=oss2,name=lnet lnet_msgs_max=20i,lnet_rst_alloc=0i,lnet_send_count=1087679i,lnet_send_length=336104672i,lnet_recv_length=77316595816i,lnet_drop_length=4160i,lnet_msgs_alloc=0i,lnet_route_count=0i,lnet_drop_count=9i,lnet_route_length=0i,lnet_recv_count=1089012i 1556525828000000000
```

[lustre]: http://lustre.org/
[guide]: http://wiki.lustre.org/Lustre_Monitoring_and_Statistics_Guide
