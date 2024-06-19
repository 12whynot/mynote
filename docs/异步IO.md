# select
函数原型
```
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct\ timeval *timeout)
```

1、nfds:要操作的文件个数