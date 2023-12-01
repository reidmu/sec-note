- Linux 提权
  - https://github.com/Getshell/LinuxTQ/

- Linux 取消历史命令记录

```bash
unset HISTORY HISTFILE HISTSAVE HISTZONE HISTORY HISTLOG; export HISTFILE=/dev/null; export HISTSIZE=0; export HISTFILESIZE=0
```

- Linux ping 查询网段存活主机

```bash
for k in $( seq 1 255);do ping -c 1 192.168.50.$k|grep "ttl"|awk -F "[ :]+" '{print $4}'; done
```
