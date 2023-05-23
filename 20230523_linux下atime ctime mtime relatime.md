# atime

access time，访问时间，文件最后一次被访问的时间，或者叫文件最后被读取的时间，因此阅读一个文件会更新它的访问时间.

atime 读取文件或者执行文件时会更改

# mtime

modify time，修改时间，最后一次修改文件的时间，指的是文件内容最后被修改的时间。
mtime 是在写入文件时随文件内容的更改而更改。

# ctime

change time（也有人说是 status time），改变时间，变更时间，最后一次改变文件的时间。指的是文件的元数据，状态最后被变更的时间。改变文件的元数据，即:属性。例如，更改文件的 inode、权限、所属组等。

ctime 文件的权限、拥有者、所属的组、链接数发生改变时的时间。当然当内容改变时也会随之改变（即inode内容发生改变和Block内容发生改变时）

ctime 是在写入文件、更改所有者、权限或链接设置时随 Inode 的内容更改而更改的

>
> atime、mtime、ctime 都放在 inode 中，若更改文件的内容就一定会更改 mtime 和 ctime，mtime 改变，ctime 一定会改变，但是文件的 ctime 可能会在 mtime 未发生变化时更改，也就是说 ctime 改变，mtime 未必改变。
>
> 例如，文件的权限更改，但是文件内容没有变化的情况下，ctime 改变，mtime 不改变。

# 如何禁用atime

```shell
vi /etc/fstab
/dev/sdd1 / xfs noatime,errors=remount-ro 0 0
```

# relatime

如果每次访问文件都会更新 atime，这是很耗时的，尤其在 web 服务器上，大量用户只是访问 html 页面，完全没有必要修改 atime。从kernel 2.6.29 开始，文件系统默认集成了一个 relatime 的属性，这样就不会每次访问文件时都改变 atime，而是满足一定条件才会更改 atime。

可能是因为在文件读操作很频繁的系统中，atime 更新所带来的开销很大，所以很多 SA（系统管理员）都在挂装文件系统的时候使用noatime 属性来停止更新 atime。但是有些程序需要根据 atime 进行一些判断和操作，所以 Linux 就推出了一个relatime特性。使用这个特性来挂装文件系统后，只有当 mtime 比 atime 更新或者上次访问时间是在 1 天前才会更新 atime。使用方法就是通过 mount -o relatime /dir 来挂装目录。
