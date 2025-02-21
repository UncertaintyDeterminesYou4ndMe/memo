# linux 命令行操作

## 三大重要命令
### sed
1. 举例
```
echo "a.\`曝光UV\`,
a.\`曝光PV\`,
a.\`点击UV\`,
a.\`点击PV\`,
a.\`完成UV\`,
a.\`完成PV\`,
a.\`领取UV\`,
a.\`领取PV\`," | sed "s/\(a\.`[^`]*`\)/SUM(\1) AS \1/g"
```
2. 解释
```bash
sed "s/\(a\.`[^`]*`\)/SUM(\1) AS \1/g"
```

- `sed`: 是一个流编辑器，可以对来自标准输入或文件的文本进行基本的文本转换。
- `"s/.../.../g"`: 这是 `sed` 的替换命令。它尝试将匹配的模式（第一个 `/.../` 之间的部分）替换为指定的文本（第二个 `/.../` 之间的部分）。`g` 表示全局替换，即在每一行中替换所有匹配的实例，而不只是第一个。
- `\(a\.`[^`]*`\)\`: 这是正则表达式的一部分，用于匹配文本。让我们分解这个表达式：
  - `\(` 和 `\)\`: 这些是捕获组的标记。它们告诉 `sed` 记住这些括号中匹配的文本，以便稍后可以在替换文本中引用。
  - `a\.`: 匹配字母 `a` 后跟一个点 `.`。反斜杠 `\` 是转义字符，用于匹配实际的点字符，因为在正则表达式中点有特殊含义（匹配任何单个字符）。
  - ``[^`]*``: 这是一个字符类，匹配不是反引号 `` ` `` 的任何字符。星号 `*` 表示匹配前面的字符类零次或多次。
  - `` ` ``: 匹配实际的反引号字符。
- `SUM(\1) AS \1`: 这是替换文本。`\1` 引用了之前由捕获组 `\( ... \)` 捕获的文本。因此，每个匹配的列名都被替换为 `SUM(列名) AS 列名` 的形式。

### 0.快捷命令

cat .zshrc

```
export PATH="$PATH:$HOME/.rvm/bin"
source ~/.bash_profile
source /etc/profile
```

cat .bash_profile 

```
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'
alias jumps='ssh jiaxiong.l@sa.shuli.com'
JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_341.jdk/Contents/Home/
PATH=$JAVA_HOME/bin:$PATH:.
CLASSPATH=$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib/dt.jar:.
export JAVA_HOME
export PATH
export CLASSPATH
```



### 1.shell awk 使用：

cat -A test.txt | awk -F '(\\^A)+' -v OFS=',' '{$1=$1;print $0}'



### 2.shell 特殊字符和控制字符：

**# 注释**

- 表示注释  #注释
- 在引号中间和\#等表示#本身
- echo ${PATH#*:} # 参数替换,不是一个注释
- echo $(( 2#101011 )) # 数制转换,不是一个注释

 

**; 分隔**

- 命令分隔，在一行中写多个命令  echo "aa" ; echo "bb"
- 在条件中的if和then如果放在同一行，也用;分隔

**;; case条件的结束**

 

**. 命令相当于source命令**

- 命令：source
- 文件名的前缀，隐藏文件
- 目录：.当前目录，..父目录
- 正则表达式：匹配任意单个字符

"" 部分引用 支持通配符扩展

 

'  ‘ 全引用，不进行通配符扩展

 

\ 转义

 

/ 目录分隔符

 

,  多个命令都被执行，但返回最后一个

 

` 后置引用

 

**: 操作符**

- 空操作

- 死循环：   while :

- 在if/then中表示什么都不做，引出分支

- 设置默认参数：  : ${username=`whoami`}

- 变量替换：   : ${HOSTNAME?} ${USER?} ${MAIL?}

- 在和 > (重定向操作符)结合使用时,把一个文件截断到0 长度,没有修改它的权限；如果文件在之前并不存在,那么就创建它.如:   

   : > data.xxx #文件"data.xxx"现在被清空了. 与 cat /dev/null >data.xxx 的作用相同 然而,这不会产生一个新的进程,因为":"是一个内建命令.

  在和>>重定向操作符结合使用时,将不会对想要附加的文件产生任何影响.

  如果文件不存在,将创建.

\* 匹配0个或多个字符；数学乘法；**幂运算

 

? 匹配任意一个字符；但在((a>b?a:b))表示c语言中的三目运算

 

**$** 

- 取变量的值 echo $PATH
- 正则表达式中表示行的结尾
- ${} 参数替换 ${PAHT}
- $* 所有参数
- $# 参数个数
- $$ 进程的ID
- $? 进程的返回状态

**( )**

- 命令组，在一个子Shell中运行  (a=3;echo $a) 其中定义的变量在后面不可用
- 数组初始化： array=(a,b,c)

{ } 代码块，即一个匿名函数，但其中定义的变量在后面依然可用

 

{ } \; 用在find的-exec中 $find -name *.txt -exec cat {} \;

 

**[ ]**

- 测试 [-z $1]
- 数组元素 a[1]='test'
- [[]]表示测试 使用**[[ ... ]]**条件判断结构, 而不是**[ ... ]**, 能够防止脚本中的许多逻辑错误. 比如, &&, ||, <, 和> 操作符能够正常存在于[[ ]]条件判断结构中, 但是如果出现在[ ]结构中的话, 会报错.
- (( ))数学运算
- 在正则表达式中表示范围 [a-z]

< <<  >  重定向和进程替换  ls -al > a.txt

 

\>  <  还用在ASCII比较 if [[ "$veg1" < "$veg2" ]]

 

\<,\> 正则表达式中的单词边界.如:bash$grep '\<the\>' textfile

 

| 管道

 

\>| 强制重定向(即使设置了noclobber 选项--就是-C 选项).这将强制的覆盖一个现存文件.

 

|| 逻辑或操作 ；用在两个命令之间的时候，表示在前一个命令结束时，若返回值为 false，继续执行下一个命令

 

&& 逻辑与；用在两个命令之间的时候，表示在前一个命令结束时，若返回值为 true，继续执行下一个命令

 

& 后台运行

 

**-**

- 参数选项
- 减号
- 重定向stdin和stdout：cd /source/directory && tar cf - . ) | (cd /dest/directory && tar xpvf -)
- 先前的工作目录 cd -
- 注：使用-开头的文件名和变量名可能会出现一些问题

\+  一个命令或者过滤器的选项标记.

 

~ home目录

~+ 当前工作目录

~- 先前工作目录

 

^ 正则表达式中表示行首

 

$IFS 用来做一些输入命令的分隔符, 默认情况下是空白.

 

 

### 3.控制字符


修改终端或文本显示的行为. . 控制字符以CONTROL + key这种方式进行组合(同时按下 ctr + k). 控制字符也可以使用8进制或16进制表示法来进行表示, 但是前边必须要加上转义符.

控制字符在脚本中不能正常使用. —— 需要 :dig 找到提示

Ctl-B退格(非破坏性的), 就是退格但是不删掉前面的字符.

Ctl-C终结一个前台作业.

Ctl-D  从一个shell中登出(与exit很相像).
      "EOF"(文件结束). 这也能从stdin中终止输入.
      在console或者在xterm窗口中输入的时候, Ctl-D将删除光标下字符. 当没有字符时, Ctl-D将退出当前会话, 在一个xterm窗口中, 则会产生关闭此窗口的效果.

Ctl-G "哔" (beep). 在一些老式的打字机终端上, 它会响一下铃.

Ctl-H "退格"(破坏性的), 就是在退格之后, 还要删掉前边的字符.

Ctl-I 水平制表符.

Ctl-J 重起一行(换一行并到行首). 在脚本中, 也可以使用8进制表示法 -- '\012' 或者16进制表示法 -- '\x0a' 来表示.

Ctl-K垂直制表符.

Ctl-L 清屏(清除终端的屏幕显示). 在终端中, 与clear命令的效果相同. 当发送到打印机上时, Ctl-L会让打印机将打印纸卷到最后.

Ctl-M 回车.

Ctl-Q 恢复(XON).在一个终端中恢复stdin.

Ctl-S 挂起(XOFF).
     在一个终端中冻结stdin. (使用Ctl-Q可以恢复输入.)

Ctl-U 删除光标到行首的所有字符. 在某些设置下, 不管光标的所在位置Ctl-U都将删除整行输入.

Ctl-V当输入字符时, Ctl-V允许插入控制字符. 

Ctl-V主要用于文本编辑.

Ctl-W 
当在控制台或一个xterm窗口敲入文本时, Ctl-W将会删除当前光标到左边最近一个空格间的全部字符. 在某些设置下, Ctl-W将会删除当前光标到左边第一个非字母或数字之间的全部字符.

##### 举例：

将控制字符 ^D 转换成空格并重定向到 tmp 文件

```bash
tr -s "[\004]" " " > tmp
```



### 4.常用命令

#### 1.lsof -i:某端口

#### 2.sar

用法：

#### 3.df

  df 可以查看一级文件夹大小、使用比例、档案系统及其挂入点，但对文件却无能为力。
  du可以查看文件及文件夹的大小。

```bash
df -h
```

#### 4.du

-c：显示所有已列出的文件的总大小

-h：易读格式

-s：显示每个输出参数的总计

```bash
du -h --max-depth=1 work/testing  

按文件大小排序 | sed 's/ //' | sort -hr

hdfs dfs -du -h /user/hive/warehouse/marketing.db/ | sed 's/ //' | sort -hr
```

#### 5.lsof

用法：

#### 6.scp

```bash
scp local_file remote_username@remote_ip:remote_folder
## eg:
scp -r conf/ sa_cluster@remote_ip:conf/
```

#### 9.ps -aux / jmap /jhat

操作记录/脚本 （***WIP***）

```bash
查看内存占比占用最多前十排名
ps auxw|head -1;ps -auxf|sort -nr -k4|head -10
ps aux --sort -rss | head -10

查看CPU占比占用最多前十排名
ps auxw|head -1;ps -auxf|sort -nr -k3|head -10
ps aux --sort=-pcpu | head -10

查看内存VSZ占用最多前十排名
ps auxw|head -1;ps -auxf|sort -nr -k5|head -10

查看内存RSS占用最多前十排名
ps auxw|head -1;ps -auxf|sort -nr -k6|head -10
```

#### 10.iftop

```
网卡实时流量监控
```



### 5.vim 使用经验

```bash
sudo chmod 账号:组 文件
##替换
#当前行
:s/foo/bar/g
#全文
:%s/foo/bar/g
#2-11行
:5,12s/foo/bar/g
```

### 6.时间函数使用

```bash
date "+%Y-%m-%d" -d ""-7 day"
```

```bash
echo $(($(date --date="$(date +%Y\/%m\/%d)" +%s)/86400 +1))
# 18950
```

```bash
echo $((($(date -d $(date +%Y%m%d) +%s)+28800)/86400))
# 18950

## 获取某日期的时间戳
date -d "2022-01-14 00:00:00" +%s 

## 将时间戳转为时间
date -d @1287331200

## 获取时间戳转为时间
date -d "1970-01-01 UTC 1287331200 seconds" "+%F %T"
```

```
Linux date命令可以用来显示或设定系统的日期与时间，在显示方面，使用者可以设定欲显示的格式，格式设定为一个加号后接数个标记，其中可用的标记列表如下：

时间方面：

% : 印出 %
%n : 下一行
%t : 跳格
%H : 小时(00..23)
%I : 小时(01..12)
%k : 小时(0..23)
%l : 小时(1..12)
%M : 分钟(00..59)
%p : 显示本地 AM 或 PM
%r : 直接显示时间 (12 小时制，格式为 hh:mm:ss [AP]M)
%s : 从 1970 年 1 月 1 日 00:00:00 UTC 到目前为止的秒数
%S : 秒(00..61)
%T : 直接显示时间 (24 小时制)
%X : 相当于 %H:%M:%S
%Z : 显示时区
日期方面：

%a : 星期几 (Sun..Sat)
%A : 星期几 (Sunday..Saturday)
%b : 月份 (Jan..Dec)
%B : 月份 (January..December)
%c : 直接显示日期与时间
%d : 日 (01..31)
%D : 直接显示日期 (mm/dd/yy)
%h : 同 %b
%j : 一年中的第几天 (001..366)
%m : 月份 (01..12)
%U : 一年中的第几周 (00..53) (以 Sunday 为一周的第一天的情形)
%w : 一周中的第几天 (0..6)
%W : 一年中的第几周 (00..53) (以 Monday 为一周的第一天的情形)
%x : 直接显示日期 (mm/dd/yy)
%y : 年份的最后两位数字 (00.99)
%Y : 完整年份 (0000..9999)
若是不以加号作为开头，则表示要设定时间，而时间格式为 MMDDhhmm[[CC]YY][.ss]，其中 MM 为月份，DD 为日，hh 为小时，mm 为分钟，CC 为年份前两位数字，YY 为年份后两位数字，ss 为秒数。


当您不希望出现无意义的 0 时(比如说 1999/03/07)，则可以在标记中插入 - 符号，比如说 date '+%-H:%-M:%-S' 会把时分秒中无意义的 0 给去掉，像是原本的 08:09:04 会变为 8:9:4。另外，只有取得权限者(比如说 root)才能设定系统时间。

当您以 root 身分更改了系统时间之后，请记得以 clock -w 来将系统时间写入 CMOS 中，这样下次重新开机时系统时间才会持续抱持最新的正确值。

语法
date [-u] [-d datestr] [-s datestr] [--utc] [--universal] [--date=datestr] [--set=datestr] [--help] [--version] [+FORMAT] [MMDDhhmm[[CC]YY][.ss]]
参数说明：

-d datestr : 显示 datestr 中所设定的时间 (非系统时间)
--help : 显示辅助讯息
-s datestr : 将系统时间设为 datestr 中所设定的时间
-u : 显示目前的格林威治时间
--version : 显示版本编号
实例
显示当前时间

# date
三 5月 12 14:08:12 CST 2010
# date '+%c' 
2010年05月12日 星期三 14时09分02秒
# date '+%D' //显示完整的时间
05/12/10
# date '+%x' //显示数字日期，年份两位数表示
2010年05月12日
# date '+%T' //显示日期，年份用四位数表示
14:09:31
# date '+%X' //显示24小时的格式
14时09分39秒
按自己的格式输出

# date '+usr_time: $1:%M %P -hey'
usr_time: $1:16 下午 -hey
显示时间后跳行，再显示目前日期

date '+%T%n%D'
显示月份与日数

date '+%B %d'
显示日期与设定时间(12:34:56)

date --date '12:34:56'
```



### 7.curl 使用示例

#### post:

```bash
curl localhost:9999/api/daizhige/article -X POST -H "Content-Type:application/json" -d '{"title":"comewords","content":"articleContent"}'

curl 'http://localhost:9104/sys/loginAction/login'   
-H 'Connection: keep-alive'   
-H 'Accept: application/json, text/plain, */*'  
-H 'Content-Type: application/json'   
-H 'Origin: http://dipc.data-pivot.com:81'  
-H 'Referer: http://dipc.data-pivot.com:81/'  
-H 'Accept-Language: zh-CN,zh;q=0.9'  
--data-raw '{"userCode":"zhangsanl","userPassword":"23","platform":1}'  -v
```

![image-20211209113114549](../Library/Application Support/typora-user-images/image-20211209113114549.png)

---

**curl命令**

不带有任何参数时，curl 就是发出 GET 请求。

$ curl https://www.example.com

上面命令向www.example.com发出 GET 请求，服务器返回的内容会在命令行输出。

**-A**

-A参数指定客户端的用户代理标头，即User-Agent。curl 的默认用户代理字符串是curl/[version]。

$ curl -A 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/76.0.3809.100 Safari/537.36' https://google.com

上面命令将User-Agent改成 Chrome 浏览器。

$ curl -A '' https://google.com

上面命令会移除User-Agent标头。

也可以通过-H参数直接指定标头，更改User-Agent。

$ curl -H 'User-Agent: php/1.0' https://google.com

**-b**

-b参数用来向服务器发送 Cookie。

$ curl -b 'foo=bar' https://google.com

上面命令会生成一个标头Cookie: foo=bar，向服务器发送一个名为foo、值为bar的 Cookie。

$ curl -b 'foo1=bar;foo2=bar2' https://google.com

上面命令发送两个 Cookie。

$ curl -b cookies.txt https://www.google.com

上面命令读取本地文件cookies.txt，里面是服务器设置的 Cookie（参见-c参数），将其发送到服务器。

**-c**

-c参数将服务器设置的 Cookie 写入一个文件。

$ curl -c cookies.txt https://www.google.com

上面命令将服务器的 HTTP 回应所设置 Cookie 写入文本文件cookies.txt。

**-d**

-d参数用于发送 POST 请求的数据体。

$ curl -d'login=emma＆password=123'-X POST https://google.com/login # 或者 $ curl -d 'login=emma' -d 'password=123' -X POST  https://google.com/login

使用-d参数以后，HTTP 请求会自动加上标头Content-Type : application/x-www-form-urlencoded。并且会自动将请求转为 POST 方法，因此可以省略-X POST。

-d参数可以读取本地文本文件的数据，向服务器发送。

$ curl -d '@data.txt' https://google.com/login

上面命令读取data.txt文件的内容，作为数据体向服务器发送。

**--data-urlencode**

--data-urlencode参数等同于-d，发送 POST 请求的数据体，区别在于会自动将发送的数据进行 URL 编码。

$ curl --data-urlencode 'comment=hello world' https://google.com/login

上面代码中，发送的数据hello world之间有一个空格，需要进行 URL 编码。

**-e**

-e参数用来设置 HTTP 的标头Referer，表示请求的来源。

curl -e '[https://google.com](https://google.com/)?q=example' https://www.example.com

上面命令将Referer标头设为https://google.com?q=example。

-H参数可以通过直接添加标头Referer，达到同样效果。

curl -H 'Referer: [https://google.com](https://google.com/)?q=example' https://www.example.com

**-F**

-F参数用来向服务器上传二进制文件。

$ curl -F '[file=@photo.png](mailto:file=@photo.png)' https://google.com/profile

上面命令会给 HTTP 请求加上标头Content-Type: multipart/form-data，然后将文件photo.png作为file字段上传。

-F参数可以指定 MIME 类型。

$ curl -F '[file=@photo.png](mailto:file=@photo.png);type=image/png' https://google.com/profile

上面命令指定 MIME 类型为image/png，否则 curl 会把 MIME 类型设为application/octet-stream。

-F参数也可以指定文件名。

$ curl -F '[file=@photo.png](mailto:file=@photo.png);filename=me.png' https://google.com/profile

上面命令中，原始文件名为photo.png，但是服务器接收到的文件名为me.png。

**-G**

-G参数用来构造 URL 的查询字符串。

$ curl -G -d 'q=kitties' -d 'count=20' https://google.com/search

上面命令会发出一个 GET 请求，实际请求的 URL 为https://google.com/search?q=kitties&count=20。如果省略--G，会发出一个 POST 请求。

如果数据需要 URL 编码，可以结合--data--urlencode参数。

$ curl -G --data-urlencode 'comment=hello world' https://www.example.com

**-H**

-H参数添加 HTTP 请求的标头。

$ curl -H 'Accept-Language: en-US' https://google.com

上面命令添加 HTTP 标头Accept-Language: en-US。

$ curl -H 'Accept-Language: en-US' -H 'Secret-Message: xyzzy' https://google.com

上面命令添加两个 HTTP 标头。

$ curl -d '{"login": "emma", "pass": "123"}' -H 'Content-Type: application/json' https://google.com/login

上面命令添加 HTTP 请求的标头是Content-Type: application/json，然后用-d参数发送 JSON 数据。

**-i**

-i参数打印出服务器回应的 HTTP 标头。

$ curl -i https://www.example.com

上面命令收到服务器回应后，先输出服务器回应的标头，然后空一行，再输出网页的源码。

**-I**

-I参数向服务器发出 HEAD 请求，然会将服务器返回的 HTTP 标头打印出来。

$ curl -I https://www.example.com

上面命令输出服务器对 HEAD 请求的回应。

--head参数等同于-I。

$ curl --head https://www.example.com

**-k**

-k参数指定跳过 SSL 检测。

$ curl -k https://www.example.com

上面命令不会检查服务器的 SSL 证书是否正确。

**-L**

-L参数会让 HTTP 请求跟随服务器的重定向。curl 默认不跟随重定向。

$ curl -L -d 'tweet=hi' https://api.twitter.com/tweet

**--limit-rate**

--limit-rate用来限制 HTTP 请求和回应的带宽，模拟慢网速的环境。

$ curl --limit-rate 200k https://google.com

上面命令将带宽限制在每秒 200K 字节。

**-o**

-o参数将服务器的回应保存成文件，等同于wget命令。

$ curl -o example.html https://www.example.com

上面命令将www.example.com保存成example.html。

**-O**

-O参数将服务器回应保存成文件，并将 URL 的最后部分当作文件名。

$ curl -O https://www.example.com/foo/bar.html

上面命令将服务器回应保存成文件，文件名为bar.html。

**-s**

-s参数将不输出错误和进度信息。

$ curl -s https://www.example.com

上面命令一旦发生错误，不会显示错误信息。不发生错误的话，会正常显示运行结果。

如果想让 curl 不产生任何输出，可以使用下面的命令。

$ curl -s -o /dev/null https://google.com

**-S**

-S参数指定只输出错误信息，通常与-s一起使用。

$ curl -s -o /dev/null https://google.com

上面命令没有任何输出，除非发生错误。

**-u**

-u参数用来设置服务器认证的用户名和密码。

$ curl -u 'bob:12345' https://google.com/login

上面命令设置用户名为bob，密码为12345，然后将其转为 HTTP 标头Authorization: Basic Ym9iOjEyMzQ1。

curl 能够识别 URL 里面的用户名和密码。

$ curl https://bob:12345@google.com/login

上面命令能够识别 URL 里面的用户名和密码，将其转为上个例子里面的 HTTP 标头。

$ curl -u 'bob' https://google.com/login

上面命令只设置了用户名，执行后，curl 会提示用户输入密码。

**-v**

-v参数输出通信的整个过程，用于调试。

$ curl -v https://www.example.com

--trace参数也可以用于调试，还会输出原始的二进制数据。

$ curl --trace - https://www.example.com

**-x**

-x参数指定 HTTP 请求的代理。

$ curl -x socks5://james:[cats@myproxy](mailto:cats@myproxy).com:8080 https://www.example.com

上面命令指定 HTTP 请求通过myproxy.com:8080的 socks5 代理发出。

如果没有指定代理协议，默认为 HTTP。

$ curl -x james:[cats@myproxy](mailto:cats@myproxy).com:8080 https://www.example.com

上面命令中，请求的代理使用 HTTP 协议。

**-X**

-X参数指定 HTTP 请求的方法。

$ curl -X POST https://www.example.com

上面命令对 https://www.example.com 发出 POST 请求。

### 8.shell 基础

#### 1. shell 统计文件中的条数

```bash
grep '"project":"SA HSHSAhjjdaviouau"' ua-2-event-sensor.log |grep '"type":"track"' |wc -l >21.ret 2>&1
```

```查找压缩文件中的关键字
zgrep "parallelPutToStoreThreadLimit" /mnt/disk1/log/hbase/http-request.log-2025-02-15.2.log.gz > /path/to/output/file
```

#### 2. shell 判断字符串包含关系的 case：

```
利用grep查找
strA="long string"
strB="string"
result=$(echo $strA | grep "${strB}")
if [[ "$result" != "" ]]
then
    echo "包含"
else
    echo "不包含"
fi
```

```
利用字符串运算符
strA="helloworld"
strB="low"
if [[ $strA =~ $strB ]]
then
    echo "包含"
else
    echo "不包含"
fi
```

```
利用通配符
A="helloworld"
B="low"
if [[ $A == *$B* ]]
then
    echo "包含"
else
    echo "不包含"
fi
```

```
利用case in 语句
thisString="1 2 3 4 5" # 源字符串
searchString="1 2" # 搜索字符串
case $thisString in 
    *"$searchString"*) echo Enemy Spot ;;
    *) echo nope ;;
esa
```

```
利用替换
STRING_A=$1
STRING_B=$2
if [[ ${STRING_A/${STRING_B}//} == $STRING_A ]]
    then
        ## is not substring.
        echo N
        return 0
    else
        ## is substring.
        echo Y
        return 1
    fi
```

```shell
## shell if 或的表达方式
if [ "$a" = 1 ] || [ "$a" = "2" ];then

　　echo $a

fi

## shell if 且的表达方式

if [ "$a" = 1 ] && [ "$a" = "2" ];then

　　echo $a

fi
```



#### 3. if：

```
str1 = str2　　　　　　当两个串有相同内容、长度时为真 
str1 != str2　　　　　 当串str1和str2不等时为真 
-n str1　　　　　　　 当串的长度大于0时为真(串非空) 
-z str1　　　　　　　 当串的长度为0时为真(空串) 
str1　　　　　　　　   当串str1为非空时为真

[ "2006.01.23" \> "2005.03.01" ] && echo dayu || echo budayu

int1 -eq int2　　　　两数相等为真 
int1 -ne int2　　　　两数不等为真 
int1 -gt int2　　　　int1大于int2为真 
int1 -ge int2　　　　int1大于等于int2为真 
int1 -lt int2　　　　int1小于int2为真 
int1 -le int2　　　　int1小于等于int2为真

-r file　　　　　用户可读为真 
-w file　　　　　用户可写为真 
-x file　　　　　用户可执行为真 
-f file　　　　　文件为正规文件为真 
-d file　　　　　文件为目录为真 
-c file　　　　　文件为字符特殊文件为真 
-b file　　　　　文件为块特殊文件为真 
-s file　　　　　文件大小非0时为真 
-t file　　　　　当文件描述符(默认为1)指定的设备为终端时为真

-a 　 　　　　　 与 
-o　　　　　　　 或 
!　　　　　　　　非


上面的三种写在括号内，对应的 && || 写在中括号之间。例如，if  [   "$a"  eq   1  -o  "$b" eq 2 ]  &&  [   "$c"  eq  3 ]
4字符串匹配
if [  `echo $str | grep -e regexp`  ];then .
 

转自：http://hi.baidu.com/ryouaki/item/0689dcb8a467b5a7eaba9319

二 具体使用
比较两个字符串是否相等的办法是：

    if [ "$test"x = "test"x ]; then

    这里的关键有几点：

    1 使用单个等号

    2 注意到等号两边各有一个空格：这是unix shell的要求

    3 注意到"$test"x最后的x，这是特意安排的，因为当$test为空的时候，上面的表达式就变成了x = testx，显然是不相等的。而如果没有这个x，表达式就会报错：[: =: unary operator expected

    

    二元比较操作符,比较变量或者比较数字.注意数字与字符串的区别.

    整数比较 需要注意的是 要么使用[]和gt组合 要么使用大于号和双括号组合

    -eq 等于,如:if [ "$a" -eq "$b" ]

    -ne 不等于,如:if [ "$a" -ne "$b" ]

    -gt 大于,如:if [ "$a" -gt "$b" ]

    -ge 大于等于,如:if [ "$a" -ge "$b" ]

    -lt 小于,如:if [ "$a" -lt "$b" ]

    -le 小于等于,如:if [ "$a" -le "$b" ]

     大于(需要双括号),如:(("$a" > "$b"))

    >= 大于等于(需要双括号),如:(("$a" >= "$b"))

    小数据比较可使用AWK

    字符串比较

    = 等于,如:if [ "$a" = "$b" ]

    == 等于,如:if [ "$a" == "$b" ],与=等价

     注意:==的功能在[[]]和[]中的行为是不同的,如下:

     1 [[ $a == z* ]] # 如果$a以"z"开头(模式匹配)那么将为true

     2 [[ $a == "z*" ]] # 如果$a等于z*(字符匹配),那么结果为true

     3

     4 [ $a == z* ] # File globbing 和word splitting将会发生

     5 [ "$a" == "z*" ] # 如果$a等于z*(字符匹配),那么结果为true

     一点解释,关于File globbing是一种关于文件的速记法,比如"*.c"就是,再如~也是.

     但是file globbing并不是严格的正则表达式,虽然绝大多数情况下结构比较像.

    != 不等于,如:if [ "$a" != "$b" ]

     这个操作符将在[[]]结构中使用模式匹配.

     大于,在ASCII字母顺序下.如:

     if [[ "$a" > "$b" ]]

     if [ "$a" \> "$b" ]

     注意:在[]结构中">"需要被转义.

     具体参考Example 26-11来查看这个操作符应用的例子.

    -z 字符串为"null".就是长度为0.

    -n 字符串不为"null"

     注意:

     使用-n在[]结构中测试必须要用""把变量引起来.使用一个未被""的字符串来使用! -z

     或者就是未用""引用的字符串本身,放到[]结构中。虽然一般情况下可

     以工作,但这是不安全的.习惯于使用""来测试字符串是一种好习惯.

if判断式
if [ 条件判断一 ] && (||) [ 条件判断二 ]; then
elif [ 条件判断三 ] && (||) [ 条件判断四 ]; then
else
   执行第三段內容程式
fi

例如：

 

root@Bizbox:~# a=0
root@Bizbox:~# b=0
root@Bizbox:~# c=5         
root@Bizbox:~# if [ $a = 0 -a $b = 0 ]&&[ $c != 0 ]; then
> echo success
> fi
success
if 使用的表达式


http://www.cnblogs.com/276815076/archive/2011/10/30/2229286.html





if 语句格式
if  条件
then
 Command
else
 Command
fi                              别忘了这个结尾 If语句忘了结尾fi
test.sh: line 14: syntax error: unexpected end of fi
if 的三种条件表达式
if
command
then

if
 函数
then  命令执行成功，等于返回0 （比如grep ,找到匹配）
执行失败，返回非0 （grep,没找到匹配） if [ expression_r_r_r  ]
then   表达式结果为真，则返回0，if把0值引向then if test expression_r_r_r
then  表达式结果为假，则返回非0，if把非0值引向then
       [ ] &&  ——快捷if
[ -f "/etc/shadow" ] && echo "This computer uses shadow passwors"    && 可以理解为then
    如果左边的表达式为真则执行右边的语句
shell的if与c语言if的功能上的区别
 shell if     c语言if 0为真，走then  正好相反，非0走then  不支持整数变量直接if
必须:if [ i –ne 0 ]

但支持字符串变量直接if
if [ str ] 如果字符串非0  支持变量直接if
if (i )
 
     echo –n “input:”
read user

if
多条指令,这些命令之间相当于“and”（与）
grep $user /etc/passwd >/tmp/null      
who -u | grep $user
then             上边的指令都执行成功,返回值$?为0，0为真，运行then
 echo "$user has logged"
else     指令执行失败，$?为1，运行else                            
 echo "$user has not logged"
fi   
# sh test.sh
input : macg
macg     pts/0        May 15 15:55   .          2075 (192.168.1.100)
macg has logged
   
# sh test.sh
input : ddd
ddd has not logged  

以函数作为if条件  (函数就相当于command,函数的优点是其return值可以自定义)
if
以函数作为if条件，
getyn
then   函数reture值0为真，走then
echo " your answer is yes"
else  函数return值非0为假，走else
echo "your anser is no"
fi  
if command  等价于 command+if $?
$ vi testsh.sh
#!/bin/sh

if
cat 111-tmp.txt | grep ting1
then
echo found
else
echo "no found"
fi  $ vi testsh.sh
#!/bin/sh

cat 111-tmp.txt | grep ting1

if [ $? -eq 0 ]
then
echo $?
echo found
else
echo $?
echo "no found"
fi $ sh testsh.sh
no found   $ sh testsh.sh
1
no found $ vi 111-tmp.txt
that is 222file
thisting1 is 111file

$ sh testsh.sh
thisting1 is 111file
found $ vi 111-tmp.txt
that is 222file
thisting1 is 111file

$ sh testsh.sh
thisting1 is 111file
0
found
   
    
   条件表达式
文件表达式
 ]    如果文件存在且可写
if [ -x file  ]    如果文件存在且可执行   
整数变量表达式
 
   字符串变量表达式
 [ $a = $b ]                 如果string1等于string2
                                  [ $string1 !=  $string2 ]   如果string1不等于string2        if  [ -n $string  ]             如果string 非空(非0），返回0(true)
if  [ -z $string  ]             如果string 为空
if  [ $sting ]                  如果string 非空，返回0 (和-n类似)  

     if [ a = b ] ;then    
echo equal
else
echo no equal
fi
[macg@machome ~]$ sh test.sh
input a:
5
input b:
5
no equal  （等于表达式没比较$a和$b,而是比较和a和b,自然a!=b)
if [ $a = $b ] ;then       
echo equal
else
echo no equal
fi
[macg@machome ~]$ sh test.sh
input a:
5
input b:
5
equal

                                                                                    [macg@machome ~]$ vi test.sh
echo -n "input your choice:"
read var
if  [ $var -eq "yes" ]
then
echo $var
fi
[macg@machome ~]$ sh -x test.sh
input your choice:
y
test.sh: line 3: test: y: integer expression_r_r_r expected
                       期望整数形式，即-eq不支持字符串

    =放在别的地方是赋值,放在if [ ] 里就是字符串等于,shell里面没有==的,那是c语言的等于

    [macg@machome ~]$ vi test.sh
echo "input a:"
read a
echo "input is $a"
if [ $a = 123 ] ; then
echo equal123
fi
[macg@machome ~]$ sh test.sh
input a:
123
input is 123
equal123 

   = 作为等于时，其两边都必须加空格，否则失效
等号也是操作符，必须和其他变量，关键字，用空格格开 (等号做赋值号时正好相反，两边不能有空格）
[macg@machome ~]$ vi test.sh

echo "input your choice:"
read var
if [ $var="yes" ]
then
echo $var
echo "input is correct"
else
echo $var
echo "input error"
fi [macg@machome ~]$ vi test.sh

echo "input your choice:"
read var
if [ $var = "yes" ]   在等号两边加空格
then
echo $var
echo "input is correct"
else
echo $var
echo "input error"
fi [macg@machome ~]$ sh test.sh
input your choice:
y
y
input is correct
[macg@machome ~]$ sh test.sh
input your choice:
n    
n
input is correct 
输错了也走then,都走then,为什么?
因为if把$var="yes"连读成一个变量，而此变量为空，返回1，则走else  [macg@machome ~]$ sh test.sh
input your choice:
y
y
input error
[macg@machome ~]$ sh test.sh
input your choice:
no                       
no
input error
一切正常
If  [  $ANS  ]     等价于  if [ -n $ANS ]
      如果字符串变量非空（then） , 空(else)
echo "input your choice:"
read ANS

if [ $ANS ]
then
echo no empty
else
echo empth
fi  [macg@machome ~]$ sh test.sh
input your choice:                       回车
                                                
empth                                    说明“回车”就是空串
[macg@machome ~]$ sh test.sh
input your choice:
34
no empty 
  整数条件表达式，大于，小于hell里没有> 和< ,会被当作尖括号，只有-ge,-gt,-le,lt
[macg@machome ~]$ vi test.sh

echo "input a:"
read a
if  [ $a -ge 100 ] ; then
echo 3bit
else
echo 2bit
fi [macg@machome ~]$ sh test.sh
input a:
123
3bit
[macg@machome ~]$ sh test.sh
input a:
20
2bit
if  test $a  ge 100 ; then

[macg@machome ~]$ sh test.sh
test.sh: line 4: test: ge: binary operator expected
if  test $a -ge 100 ; then

[macg@machome ~]$ sh test.sh
input a:
123
3bit

    逻辑非 !                   条件表达式的相反
if [ ! 表达式 ]
if [ ! -d $num ]                        如果不存在目录$num

    逻辑与 –a条件表达式的并列
if [ 表达式1  –a  表达式2 ]

                –o 表达式2 ]

   逻辑表达式
    表达式与前面的=  != -d –f –x -ne -eq -lt等合用
    逻辑符号就正常的接其他表达式，没有任何括号（ ），就是并列
    注意逻辑与-a与逻辑或-o很容易和其他字符串或文件的运算符号搞混了

 
[macg@mac-home ~]$ vi test.sh
:
echo "input the num:"
read num
echo "input is $num"

if [ -z "$JHHOME" -a -d $HOME/$num ]   如果变量$JHHOME为空，且$HOME/$num目录存在
then
JHHOME=$HOME/$num                      则赋值
fi

echo "JHHOME is $JHHOME"  
-----------------------
[macg@mac-home ~]$ sh test.sh
input the num:
ppp
input is ppp
JHHOME is

目录-d $HOME/$num   不存在，所以$JHHOME没被then赋值
[macg@mac-home ~]$ mkdir ppp
[macg@mac-home ~]$ sh test.sh
input the num:
ppp
input is ppp
JHHOME is /home/macg/ppp

echo "input your choice:"
read ANS

if [ $ANS="Yes" -o $ANS="yes" -o $ANS="y" -o $ANS="Y" ]
then
ANS="y"
else
ANS="n"
fi

echo $ANS
[macg@machome ~]$ sh test.sh
input your choice:
n
y
[macg@machome ~]$ sh test.sh
input your choice:
no
y
为什么输入不是yes,结果仍是y(走then）
因为=被连读了，成了变量$ANS="Yes"，而变量又为空，所以走else了

[macg@machome ~]$ vi test.sh

echo "input your choice:"
read ANS    echo "input your choice:"
read ANS

if [ $ANS = "Yes" -o $ANS = "yes" -o $ANS = "y" -o $ANS = "Y" ]
then
ANS="y"
else
ANS="n"
fi

echo $ANS [macg@machome ~]$ sh test.sh
input your choice:
no
n
[macg@machome ~]$ sh test.sh
input your choice:
yes
y
[macg@machome ~]$ sh test.sh
input your choice:
y
y
 test 条件表达式 作为if条件===================================
if test $num -eq 0      等价于   if [ $num –eq 0 ]
  test  表达式,没有 [  ]
if test $num -eq 0                
    man test
[macg@machome ~]$ man test
[(1)                             User Commands                            [(1)

SYNOPSIS
       test EXPRESSION
       [ EXPRESSION ]


       [-n] STRING
              the length of STRING is nonzero          -n和直接$str都是非0条件

       -z STRING
              the length of STRING is zero

       STRING1 = STRING2
              the strings are equal

       STRING1 != STRING2
              the strings are not equal

       INTEGER1 -eq INTEGER2
              INTEGER1 is equal to INTEGER2

       INTEGER1 -ge INTEGER2
              INTEGER1 is greater than or equal to INTEGER2

       INTEGER1 -gt INTEGER2
              INTEGER1 is greater than INTEGER2

       INTEGER1 -le INTEGER2
              INTEGER1 is less than or equal to INTEGER2

       INTEGER1 -lt INTEGER2
              INTEGER1 is less than INTEGER2

       INTEGER1 -ne INTEGER2
              INTEGER1 is not equal to INTEGER2

       FILE1 -nt FILE2
              FILE1 is newer (modification date) than FILE2

       FILE1 -ot FILE2
              FILE1 is older than FILE2

       -b FILE
              FILE exists and is block special

       -c FILE
              FILE exists and is character special

       -d FILE
              FILE exists and is a directory

       -e FILE
              FILE exists                                 文件存在

       -f FILE
              FILE exists and is a regular file     文件存在且是普通文件

       -h FILE
              FILE exists and is a symbolic link (same as -L)

       -L FILE
              FILE exists and is a symbolic link (same as -h)

       -G FILE
              FILE exists and is owned by the effective group ID

       -O FILE
              FILE exists and is owned by the effective user ID

       -p FILE
              FILE exists and is a named pipe


       -s FILE
              FILE exists and has a size greater than zero

       -S FILE
              FILE exists and is a socket

       -w FILE
              FILE exists and is writable

       -x FILE
FILE exists and is executable
 

        && 如果是“前面”，则“后面”
[ -f /var/run/dhcpd.pid ] && rm /var/run/dhcpd.pid    检查 文件是否存在，如果存在就删掉
   ||   如果不是“前面”，则后面 [ -f /usr/sbin/dhcpd ] || exit 0    检验文件是否存在，如果存在就退出

     [ -z "$1" ] && help                 如果第一个参数不存在（-z  字符串长度为0 ）
[ "$1" = "-h" ] && help                        如果第一个参数是-h,就显示help

例子
#!/bin/sh
[ -f "/etc/sysconfig/network-scripts/ifcfg-eth1" ] && rm -f /etc/sysconfig/network-scripts/ifcfg-eth1cp ifcfg-eth1.bridge /etc/sysconfig/network-scripts/ifcfg-eth1 [ -f "/etc/sysconfig/network-scripts/ifcfg-eth0:1" ] && rm -f /etc/sysconfig/network-scripts/ifcfg-eth0:1
```







#### 4. **基本命令**

\1. 	-z ：可判断参数是否为空

2.	nohup ： 后台执行脚本

3.	$? ：  函数退出码的标准变量

4.	which sa_mysql   		vim ~/bin/sa_mysql    		hosts

5、cmd+d :多窗口，cmd+shift+d:关闭一个窗口

6、打开一个窗口光标移动到终端，命令+ N代开一个新的窗口

7、在一个新窗口中建立多个终端窗口，命令+ T，即可实现

#### **5. find 命令使用方法**

```
Linux find 命令用来在指定目录下查找文件。任何位于参数之前的字符串都将被视为欲查找的目录名。如果使用该命令时，不设置任何参数，则 find 命令将在当前目录下查找子目录与文件。并且将查找到的子目录和文件全部进行显示。

语法为：

1

find   path   -option   [   -print ]   [ -exec   -ok   command ]   {} \;

其中

find 根据下列规则判断 path 和 expression，在命令列上第一个 - ( ) , ! 之前的部份为 path，之后的是 expression。如果 path 是空字串则使用目前路径，如果 expression 是空字串则使用 -print 为预设 expression。

expression 中可使用的选项有二三十个之多，在此只介绍最常用的部份。

-mount, -xdev : 只检查和指定目录在同一个文件系统下的文件，避免列出其它文件系统中的文件

-amin n : 在过去 n 分钟内被读取过

-anewer file : 比文件 file 更晚被读取过的文件

-atime n : 在过去n天内被读取过的文件

-cmin n : 在过去 n 分钟内被修改过

-cnewer file :比文件 file 更新的文件

-ctime n : 在过去n天内被修改过的文件

-empty : 空的文件-gid n or -group name : gid 是 n 或是 group 名称是 name

-ipath p, -path p : 路径名称符合 p 的文件，ipath 会忽略大小写

-name name, -iname name : 文件名称符合 name 的文件。iname 会忽略大小写

-size n : 文件大小 是 n 单位，b 代表 512 位元组的区块，c 表示字元数，k 表示 kilo bytes，w 是二个位元组。

-type c : 文件类型是 c 的文件。

d: 目录

c: 字型装置文件

b: 区块装置文件

p: 具名贮列

f: 一般文件

l: 符号连结

s: socket

-pid n : process id 是 n 的文件

你可以使用 ( ) 将运算式分隔，并使用下列运算。

exp1 -and exp2

! expr

-not expr

exp1 -or exp2

exp1, exp2

示例如下：

将当前目录及其子目录下所有文件后缀为 .c 的文件列出来

1

# find . -name "*.c"
```

```bash
## 定时删除日志任务
find /data/log -type -mtime +3 | xargs rm -rf
```



#### 6. **scp使用方法**

```bash
sudo scp -r sa_cluster@data01:./{file} sa_cluster@data01:./{file}
scp local_file remote_username@remote_ip:remote_folder
## eg:
scp -r conf/ sa_cluster@remote_ip:conf/
```



#### 7.crontab 

```shell
## 将Crontab中的命令输出按照当前日期进行存储
0 2 * * * /usr/bin/php /home/wwwroot/default/monkey/sync_product.php > /home/wwwroot/default/log/monkey_sync_product_$(date +\%Y\%m\%d).log 2>&1

```

#### 8.将脚本输出到控制台和日志文件中

```
exec 3>&1 1>>${LOG_FILE} 2>&1

会将 stdout 和 stderr 输出发送到日志文件中，但也会让您将 fd 3 连接到控制台，因此您可以这样做
echo "Some console message" 1>&3

只向控制台写一条消息，或
echo "Some console and log file message" | tee /dev/fd/3

将消息写入控制台和日志文件 - tee将其输出发送到它自己的 fd 1(这里是 LOG_FILE )和你告诉它写入的文件(这里是 fd 3，即控制台)。

例子:
exec 3>&1 1>>${LOG_FILE} 2>&1

echo "This is stdout"
echo "This is stderr" 1>&2
echo "This is the console (fd 3)" 1>&3
echo "This is both the log and the console" | tee /dev/fd/3

会打印
This is the console (fd 3)
This is both the log and the console

在控制台上并把
This is stdout
This is stderr
This is both the log and the console

进入日志文件。
关于bash - 将输出写入日志文件和控制台，我们在Stack Overflow上找到一个类似的问题： https://stackoverflow.com/questions/18460186/

```

### 9.zip/unzip

```
```



### 10.命令行常用快捷键

```
Tab 自动补全
Ctrl+a 光标移动到开始位置
Ctrl+e 光标移动到最末尾
Ctrl+k 删除此处至末尾的所有内容
Ctrl+u 删除此处至开始的所有内容
Ctrl+d 删除当前字符
Ctrl+h 删除当前字符前一个字符
Ctrl+w 删除此处到左边的单词
Ctrl+y 粘贴由 Ctrl+u ， Ctrl+d ， Ctrl+w 删除的单词
Ctrl+l 相当于clear，即清屏
Ctrl+r 查找历史命令
Ctrl+b 向回移动光标
Ctrl+f 向前移动光标
Ctrl+t 将光标位置的字符和前一个字符进行位置交换
Ctrl+& 恢复 ctrl+h 或者 ctrl+d 或者 ctrl+w 删除的内容
Ctrl+S 暂停屏幕输出
Ctrl+Q 继续屏幕输出
Ctrl+Left-Arrow 光标移动到上一个单词的词首
Ctrl+Right-Arrow 光标移动到下一个单词的词尾
Ctrl+p 向上显示缓存命令
Ctrl+n 向下显示缓存命令
Ctrl+d 关闭终端
Ctrl+xx 在EOL和当前光标位置移动
Ctrl+x@ 显示可能hostname补全
Ctrl+c 终止进程/命令
Shift +上或下 终端上下滚动
Shift+PgUp/PgDn 终端上下翻页滚动
Ctrl+Shift+n 新终端
alt+F2 输入gnome-terminal打开终端
Shift+Ctrl+T 打开新的标签页
Shift+Ctrl+W 关闭标签页
Shift+Ctrl+C 复制
Shift+Ctrl+V 粘贴
Alt+数字 切换至对应的标签页
Shift+Ctrl+N 打开新的终端窗口
Shift+Ctrl+Q 管壁终端窗口
Shift+Ctrl+PgUp/PgDn 左移右移标签页
Ctrl+PgUp/PgDn 切换标签页
F1 打开帮助指南
F10 激活菜单栏
F11 全屏切换
Alt+F 打开 “文件” 菜单（file）
Alt+E 打开 “编辑” 菜单（edit）
Alt+V 打开 “查看” 菜单（view）
Alt+S 打开 “搜索” 菜单（search）
Alt+T 打开 “终端” 菜单（terminal）
Alt+H 打开 “帮助” 菜单（help）

另外一些小技巧包括：在终端窗口命令提示符下，连续按两次 Tab 键、或者连续按三次 Esc 键、或者按 Ctrl+I 组合键，将显示所有的命令及工具名称。Application 键即位置在键盘上右 Ctrl 键左边的那个键，作用相当于单击鼠标右键。
```

### 11.hadoop 修改权限操作

```bash
sudo -u hdfs hadoop fs -chown hduser8009:supergroup /user/hduser8009/history/userlabel
sudo -u hdfs hadoop fs -chmod 777 /user/hduser8009/history/userlabel
```
