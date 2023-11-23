- Linux 提权
  - https://github.com/Getshell/LinuxTQ/

- Linux 取消历史命令记录

```bash
unset HISTORY HISTFILE HISTSAVE HISTZONE HISTORY HISTLOG; export HISTFILE=/dev/null; export HISTSIZE=0; export HISTFILESIZE=0
```
