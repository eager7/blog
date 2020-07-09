# 查看磁盘的UUID并挂载

## 查看UUID

```text
pct@ubuntu-x86:~$ ls -l /dev/disk/by-uuid/ 
total 0
lrwxrwxrwx 1 root root 10 10月 27 11:25 000F9B8B0002C792 -> ../../sdc5
lrwxrwxrwx 1 root root 10 10月 27 11:21 57A0324AB1A8DD07 -> ../../sdb7
lrwxrwxrwx 1 root root 10 10月 27 11:16 5a9f2aee-cfb4-40f4-ba73-fd3b01d8611c -> ../../sda1
lrwxrwxrwx 1 root root 10 10月 27 11:22 8592F969687A7E01 -> ../../sdb5
lrwxrwxrwx 1 root root 10 10月 27 11:16 c4fb3ef0-aec4-4989-a304-88cd9f512d4a -> ../../sda5
lrwxrwxrwx 1 root root 10 10月 27 11:21 C5C2582287F5AA62 -> ../../sdb8
lrwxrwxrwx 1 root root 10 10月 27 11:16 dff6fa53-d6b8-4bdc-b07d-9ec414b0cffb -> ../../sda3
lrwxrwxrwx 1 root root 10 10月 27 11:23 E4F2EB7A219308D0 -> ../../sdb6
```

## 在fsatb中挂载：

```text
pct@ubuntu-x86:~$ cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/sda1 during installation
UUID=5a9f2aee-cfb4-40f4-ba73-fd3b01d8611c /               ext4    errors=remount-ro 0       1
# swap was on /dev/sda3 during installation
UUID=dff6fa53-d6b8-4bdc-b07d-9ec414b0cffb none            swap    sw              0       0
# swap was on /dev/sda5 during installation
UUID=c4fb3ef0-aec4-4989-a304-88cd9f512d4a none            swap    sw              0       0
UUID=000F9B8B0002C792 /media/pct/My     ntfs utf8,uid=1000,gid=1000,fmask=033 0 0
UUID=57A0324AB1A8DD07 /media/pct/Life     ntfs utf8,uid=1000,gid=1000,fmask=033 0 0
UUID=E4F2EB7A219308D0 /media/pct/Study     ntfs utf8,uid=1000,gid=1000,fmask=033 0 0
UUID=8592F969687A7E01 /media/pct/Work     ntfs utf8,uid=1000,gid=1000,fmask=033 0 0
UUID=C5C2582287F5AA62 /media/pct/Play     ntfs utf8,uid=1000,gid=1000,fmask=033 0 0
```

重启即可。

## 注意

NTFS格式的磁盘没有权限的管理，所以无法在里面修改文件的读写以及执行权限，如果经常用来开发的磁盘，最好是修改为Linux格式。

