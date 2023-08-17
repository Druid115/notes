### CPU 占用过高

1. top 或者 jps 查看 java 进程
2. top -Hp <pid> 查看进程详细信息，获取目标线程的 CPU 使用情况
3. 找到占用大的线程的 pid，转换成 16 进制   printf "%x\n" pid
4. jstack 进程 pid| grep -10 转换成 16 进制的线程 pid
5. 或者     jstack -l 进程 pid >> /usr/local/error.txt    导出查看



## 内存占用过高

1. jmap -heap 进程 pid：查看堆使用情况
2. jmap -histo 进程 pid：查看堆中对象数量和大小
3. jmap -histo:live 进程 pid：这个命名会触发一次 FUll GC，只统计存活对象