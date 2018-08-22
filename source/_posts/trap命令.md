----
title: bash命令：trap
date: 2018-08-23 14:25:08
tags:
   - linux
   - bash
----
## 语法
`trap [COMMANDS] [SIGNALS]`

## 示例

### 防止脚本执行中断导致文件未清除
<!-- more -->
```sh
#!/bin/sh

trap cleanup 1 2 3 6

cleanup()
{
  echo "Caught Signal ... cleaning up."
  rm -rf /tmp/temp_*.$$
  echo "Done cleanup ... quitting."
  exit 1
}

### main script
for i in *
do
  sed s/FOO/BAR/g $i > /tmp/temp_${i}.$$ && mv /tmp/temp_${i}.$$ $i
done
```

上述代码将当前文件夹的文件中所有的`FOO`替换为`BAR`，写入到临时文件`/tmp/temp_${i}.$$`再写回到当前文件。如果在执行中途中断，可能仍然在`/tmp`中有临时文件。
`trap`语句表示在信号1、2、3、6时执行`cleanup()`函数。常用的`CTRL-C`是信号2（`SIGINT`）。

### 一个有趣的例子
```sh
#!/bin/sh
trap 'increment' 2

increment()
{
  echo "Caught SIGINT ..."
  X=`expr ${X} + 500`
  if [ "${X}" -gt "2000" ]
  then
    echo "Okay, I'll quit ..."
    exit 1
  fi
}

### main script
X=0
while :
do
  echo "X=$X"
  X=`expr ${X} + 1`
  sleep 1
done
```

示例中有一个变量`X`，每隔1s增加1，并打印出来。当捕获到`CTRL-C`不会立即退出，而是让变量增加500，当变量大于2000时脚本才会退出。如下：
![trapTest.sh](https://i.loli.net/2018/08/22/5b7d2cf2e787a.png)
当然所有脚本都能用`kill -9 <PID>`退出。

### 一些信号
|Number    |SIG        | 含义|
|-------    |-------    |----------    |
|0|	0|	On exit from shell|
|1|	SIGHUP|	Clean tidyup|
|2|	SIGINT	|Interrupt|
|3|	SIGQUIT|	Quit|
|6	|SIGABRT|	Abort|
|9	|SIGKILL	|Die Now (cannot be trapped)|
|14|	SIGALRM	|Alarm Clock|
|15|	SIGTERM	|Terminate|


注意如果脚本运行在忽略信号的环境下，比如通过`nohup`运行，那么脚本仍然会忽略这些信号。

----
**参考：**
- [Bash Guide for Beginners: 12.2 Traps](http://tldp.org/LDP/Bash-Beginners-Guide/html/sect_12_02.html)
- [man7.org: trap](http://man7.org/linux/man-pages/man1/trap.1p.html#top_of_page)
- [shellscript.sh: trap](https://www.shellscript.sh/trap.html)