# 安装firda
在Macos上安装Frida
> 这步比较简单  直接使用“pip install --user frida”

# 实验
## 实验一 setting up

---
参考 https://www.frida.re/docs/functions/
---
### 编译 hello ，简单hook
编译hello,获取函数f的地址，再用对应hook脚本进行hook，再运行hook会有权限提示，是否控制另外一个进程，授权后就能获取函数f hook对应的参数。



~~~~
hello 运行终端 
$ ./hello 

f() is at 0x1043c6ed0

Number: 0
Number: 1
Number: 3
Number: 4
Number: 5  


hook 脚本运行终端

$ python hook.py 0x1043c6ed0
{u'type': u'send', u'payload': 58}
{u'type': u'send', u'payload': 59}
{u'type': u'send', u'payload': 60}
{u'type': u'send', u'payload': 61}
{u'type': u'send', u'payload': 62}


~~~~

### 修改函数参数
修改python脚本，修改传入参数



~~~~
hello 运行终端 
./hello 
f() is at 0x10fc80ed0
Number: 0
Number: 1
Number: 2
...
Number: 44
Number: 1337
Number: 1337
Number: 1337
Number: 1337
Number: 1337
Number: 72
Number: 73
Number: 74 


hook 脚本运行终端

$ python modify.py 0x10fc80ed0


~~~~


###  调用函数 
修改python脚本，增加对应的调用

~~~~


hello 运行终端 
./hello 
f() is at 0x10d20ded0
Number: 0
Number: 1
Number: 2
Number: 3

...
Number: 21
Number: 1911
Number: 1911
Number: 1911
Number: 22
Number: 23


hook 脚本运行终端

$ python call.py 0x10d20ded0


~~~~


## 实验二 
### 注入字符串与函数调用

~~~~
hi 运行终端 
./hi
f() is at 0x10b719eb0
s is at 0x10b719f90
String: Testing!
String: Testing!
String: Testing!
String: Testing!
String: Testing!
String: Testing!
String: TESTMEPLZ!
String: Testing!
String: Testing!


hook 脚本运行终端

$ python stringhook.py 0x10b719eb0


~~~~

###  注入内存对象
需要3个终端 依次运行，等脚本运行后，在client终端按回车键

~~~~
client 运行终端 
./client 127.0.0.1
connect() is at: 0x7fff8b697528

Here's the serv_addr buffer:
00 02 13 88 7f 00 00 01 00 00 00 00 00 00 00 00

Press ENTER key to Continue


netcat 运行终端
nc -l 5001
Hello there!


hook 脚本运行终端
python struct_mod.py
[i] Allocating memory and writing bytes...
[i] Injecting malicious byte array:



~~~~


## 实验三 对目标进程 收发信息

---
参考 https://www.frida.re/docs/messages/
---
### 收发信息
~~~~
hello 运行终端
./hello 
f() is at 0x1024d4ed0
Number: 0
Number: 1
Number: 2
...
Number: 79
Number: 160 //从这个数开始 是rpc脚本 发给hello进程的
Number: 162
Number: 164
Number: 166
Number: 168
Number: 170
Number: 172
Number: 174
...
Number: 224
Number: 226
Number: 228 //后面rpc脚本停止了
Number: 115
Number: 116
Number: 117

脚本

 python rpc.py 0x1024d4ed0
{u'type': u'send', u'payload': u'0x50'}
{u'type': u'send', u'payload': u'0x51'}
{u'type': u'send', u'payload': u'0x52'}
{u'type': u'send', u'payload': u'0x53'}
{u'type': u'send', u'payload': u'0x54'}
{u'type': u'send', u'payload': u'0x55'}
{u'type': u'send', u'payload': u'0x56'}
{u'type': u'send', u'payload': u'0x57'}
{u'type': u'send', u'payload': u'0x58'}
...
{u'type': u'send', u'payload': u'0x70'}
{u'type': u'send', u'payload': u'0x71'}
{u'type': u'send', u'payload': u'0x72'}
~~~~


