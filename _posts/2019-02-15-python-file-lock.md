---
layout: post
title: "Python 文件锁使用"
date: 2019-02-15
categories: Python
tags: [Python]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png?raw=true
---

在Windows 和 linux 下 读写文件枷锁

<!-- more -->

```python
#!/usr/bin/python
# -*- coding:utf-8 -*-

import os

if os.name == 'nt':
    import win32con, win32file, pywintypes

    LOCK_EX = win32con.LOCKFILE_EXCLUSIVE_LOCK
    LOCK_SH = 0  # The default value
    LOCK_NB = win32con.LOCKFILE_FAIL_IMMEDIATELY
    __overlapped = pywintypes.OVERLAPPED()


    def lock(f, flags=1):
        """
        flags=1, 阻塞其他进程进行文件读写
        flags=2, 非阻塞，立即抛出异常
        """
        lock_flag = 1
        if flags == 1:
            lock_flag = LOCK_EX
        elif flags == 2:
            lock_flag = LOCK_EX | LOCK_NB
        hfile = win32file._get_osfhandle(f.fileno())
        win32file.LockFileEx(hfile, lock_flag, 0, 0xffff0000, __overlapped)


    def unlock(file):
        hfile = win32file._get_osfhandle(file.fileno())
        win32file.UnlockFileEx(hfile, 0, 0xffff0000, __overlapped)
elif os.name == 'posix':
    import fcntl
    from fcntl import LOCK_EX, LOCK_NB


    def lock(f, flags):
        """
        flags=1, 阻塞其他进程进行文件读写
        flags=2, 非阻塞，立即抛出异常
        """
        lock_flag = 1
        if flags == 1:
            lock_flag = LOCK_EX
        elif flags == 2:
            lock_flag = LOCK_EX | LOCK_NB
        fcntl.flock(f.fileno(), lock_flag)


    def unlock(f):
        fcntl.flock(f.fileno(), fcntl.LOCK_UN)
else:
    raise RuntimeError("File Locker only support NT and Posix platforms!")


def write_file(file_path, content):
    try:
        with open(file_path, 'ab') as f:
            lock(f, 2)
            f.write(content)
            unlock(f)
    except Exception as e:
        raise e


def read_file(file_path):
    result = None
    try:
        with open(file_path, 'rb+') as f:
            lock(f, 2)
            result = f.read()
            f.seek(0)  # clean file
            f.truncate()
            unlock(f)
        return result
    except EOFError:
        pass
    except Exception as e:
        raise e


if __name__ == "__main__":
    pass
```