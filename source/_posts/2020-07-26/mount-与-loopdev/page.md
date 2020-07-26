---
title: mount 与 loopdev
date: 2020-07-26 07:28:29
tags: linux
---

&emsp;&emsp;在 linux 中我们可以像一个块设备一样挂载一个文件，这是通过 loopdev 实现的。它可以将一个文件虚拟为一个块设备。

```shell
losetup `losetup -f` 镜像文件
```

- losetup -f 找到第一个空闲的 loop 设备

&emsp;&emsp;然后 mount 便能令其挂载。但是我们平时使用 mount 命令时却并不需要这样做，因为现代的 mount 可以自动帮我们做这件事。具体可以看 mount man 的 LOOP-DEVICE SUPPORT 章节。而且当它 umount 时也会自动释放占用的 loop 设备。

&emsp;&emsp;问题来了，那 umount 怎么知道要释放 loop 设备而且不会误释放我们手动设置的 loop 设备呢？
通过查看 util-linux 的代码可以知道，是通过 user_mountflags 的 MNT_MS_LOOP 标志实现的。

```c
/**
 * mnt_context_set_user_mflags:
 * @cxt: mount context
 * @flags: mount(2) flags (MNT_MS_* flags, e.g. MNT_MS_LOOP)
 *
 * Sets userspace mount flags.
 *
 * See also notes for mnt_context_set_mflags().
 *
 * Returns: 0 on success, negative number in case of error.
 */
int mnt_context_set_user_mflags(struct libmnt_context *cxt, unsigned long flags)
{
	if (!cxt)
		return -EINVAL;
	cxt->user_mountflags = flags;
	return 0;
}
```
