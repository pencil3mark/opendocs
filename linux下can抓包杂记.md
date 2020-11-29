# 使用python分析can总线

暂时先不考虑物理can，操作vcan设备试试。

can物理设备支持：https://zhuanlan.zhihu.com/p/135841135

```
$ pip3 install python-ics
```

python can库操作示例：https://pypi.org/project/python-can/

python can库更加详细的api介绍：https://python-can.readthedocs.io/en/stable/

1. 根据以上python的can库官网案例，试试：

   ```
   $ cat vcan0_python.py
   #!/bin/python3
   # -*- coding: UTF-8 -*-
   
   # import the library
   import can
   
   # create a bus instance
   # many other interfaces are supported as well (see below)
   bus = can.Bus(interface='socketcan',
                 channel='vcan0',
                 receive_own_messages=True)
   
   # send a message
   message = can.Message(arbitration_id=123, is_extended_id=True,
                         data=[0x11, 0x22, 0x33])
   bus.send(message, timeout=0.2)
   
   # iterate over received messages
   for msg in bus:
       # print("{X}: {}".format(msg.arbitration_id, msg.data))
       print("{:#X}: {}".format(msg.arbitration_id, msg.data))
   
   # or use an asynchronous notifier
   notifier = can.Notifier(bus, [can.Logger("recorded.log"), can.Printer()])
   ```

   分析：

   * create a bus instance: 初始化can接口实例，支持很多接口类型，目前用vcan
   * send a message: can报文发送功能，构造can包，然后发送数据
   * iterate over received messages：can接口上报文抓包和输出，显示大量报文
   * or use an asynchronous notifier：can数据本地化log文件或数据库存储

   以上是python can库的基本用法！

   效果：( 输出的格式基本是：  canid ：can数据报文 )

   ```
   # root @ archlinux in ~/test/ICSim [11:34:51] C:130
   $ ./vcan0_python.py 
   0X7B: bytearray(b'\x11"3')
   0X158: bytearray(b'\x00\x00\x00\x00\x00\x00\x00(')
   0X161: bytearray(b'\x00\x00\x05P\x01\x08\x00+')
   0X191: bytearray(b'\x01\x00\x90\xa1A\x00\x12')
   0X133: bytearray(b'\x00\x00\x00\x00\xb6')
   0X136: bytearray(b'\x00\x02\x00\x00\x00\x00\x009')
   0X13A: bytearray(b'\x00\x00\x00\x00\x00\x00\x007')
   0X13F: bytearray(b'\x00\x00\x00\x05\x00\x00\x00=')
   0X164: bytearray(b'\x00\x00\xc0\x1a\xa8\x00\x00\x13')
   0X17C: bytearray(b'\x00\x00\x00\x00\x10\x00\x000')
   0X18E: bytearray(b'\x00\x00z')
   0X294: bytearray(b'\x04\x0b\x00\x02\xcfZ\x00;')
   0X21E: bytearray(b'\x03\xe87E"\x06>')
   0X183: bytearray(b'\x00\x00\x00\x05\x00\x00\x103')
   0X143: bytearray(b'kk\x00\xff')
   0X95: bytearray(b'\x80\x00\x07\xf4\x00\x00\x00&')
   0X166: bytearray(b'\xd02\x006')
   0X158: bytearray(b'\x00\x00\x00\x00\x00\x00\x007')
   0X161: bytearray(b'\x00\x00\x05P\x01\x08\x00:')
   0X191: bytearray(b'\x01\x00\x10\xa1A\x00)')
   ```

   在数据包的基本格式上，跟candump还是存在一些差距的。需要拿一个简单包去对比下模式，目前使用最简单报文是左转向灯，can id ：188。

2. python  can 分析研究，数据抓包并输出到屏幕：

   ```
#!/bin/python3
   # -*- coding: UTF-8 -*-
   
   # import the library
   import can
   
   # create a bus instance
   # many other interfaces are supported as well (see below)
   bus = can.Bus(interface='socketcan',
                 channel='vcan0',
                 receive_own_messages=True)
   
   
   # iterate over received messages
   for msg in bus:
       # print("{X}: {}".format(msg.arbitration_id, msg.data))
       # 原示例的string.format()有些问题，应该是markdown问题，修改如下：
       print("{:#X}: {}".format(msg.arbitration_id, msg.data))
       # 输出示例：0X244: bytearray(b'\x00\x00\x00\x01\xbb')
       # 研究下string.format()的使用，让输出更好看些,老套路，先分析变量数据类型
   ```
   
   以上可以直接输出can数据包的内容，但是有个问题：

   **msg是啥？一个for  in，到底是拿的bus里边的什么东西？**

   * msg.arbitration_id ： 是什么数据类型？
* msg.data: 是什么数据类型？
  
   **参考：** doc_201125_python_can使用python抓取can报文.md  （ 优秀can抓包分析实践 ）
   
   使用python的type（）找到了数据类型：
   
   * for in 的msg是  bus.recv ( ) 返回的对象：<class 'can.message.Message'>
   * msg.arbitration_id 是个 int：<class 'int'>
   * msg.data是个 二进制字节数组：<class 'bytearray'>
   
   ```
   $ python3
   Python 3.8.6 (default, Sep 30 2020, 04:00:38) 
   [GCC 10.2.0] on linux
   Type "help", "copyright", "credits" or "license" for more information.
   
   >>> import can
   >>> bus = can.Bus(interface='socketcan',channel='vcan0',receive_own_messages=True)
   >>> type(bus.recv())
   <class 'can.message.Message'>
   >>> type(bus.recv().arbitration_id)
   <class 'int'>
   >>> type(bus.recv().data)
   <class 'bytearray'>
   ```
   
   知道数据类型了，接下来可以考虑格式化字串显示：
   
   **参考：** doc_201125_python中的bytes和hexstring之间转化.md
   
   最后找到，完全适配 cansend 工具的数据包 字串 输出：
   
   ```
   #!/bin/python3
   # -*- coding: UTF-8 -*-
   
   # import the library
   import can
   import binascii
   
   # create a bus instance
   # many other interfaces are supported as well (see below)
   bus = can.Bus(interface='socketcan',
                 channel='vcan0',
                 receive_own_messages=True)
   
   # 查看抓包数据输出方法：
   # msg = bus.recv()
   
   # iterate over received messages
   for msg in bus:
       # print("{X}: {}".format(msg.arbitration_id, msg.data))
       # 原示例的string.format()有些问题，应该是markdown问题，修改如下：
       # ":#X" 这些格式问题，只应用于数字！
       # print("{:#X}: {}".format(msg.arbitration_id, msg.data))
       # 输出示例：0X244: bytearray(b'\x00\x00\x00\x01\xbb')
       # 研究下string.format()的使用，让输出更好看些,老套路，先分析变量数据类型
       # print("{:#X}: {}".format(msg.arbitration_id, str(msg.data)))
       # 数据的输出没有变化：0X166: bytearray(b'\xd02\x006')
       # print("{:#X}: {}".format(msg.arbitration_id, binascii.b2a_hex(msg.data)))
       # 输出：0X17C: b'0000000010000012' ; 怎么去掉b
       # print("{:#X}: {}".format(msg.arbitration_id, binascii.b2a_hex(msg.data).decode("UTF-8")))
       # 输出：0X17C: 0000000010000012 可以，有candump的log样子
       # print("{:#X}#{}".format(msg.arbitration_id, binascii.b2a_hex(msg.data).decode("UTF-8")))
       # 输出：0X244#0000000188
       
       
       print(("{:#X}#{}".format(msg.arbitration_id, binascii.b2a_hex(msg.data).decode("UTF-8"))).replace('0X',''))
       # 之前不用decode("utf-8")时候，数据段会出现乱码“=”什么的。转utf-8之后就好了，哈哈
       # 输出，漂亮的cansend 模式：183#0000000600001014
       # 总结，上边的数据类型转换好绕，不过落数据库。最后还是用字串最好！！
   ```
   
3. python can 发送数据包：

   ```
   #!/bin/python3
   # -*- coding: UTF-8 -*-
   
   # import the library
   import can
   
   # create a bus instance
   # many other interfaces are supported as well (see below)
   bus = can.Bus(interface='socketcan',
                 channel='vcan0',
                 receive_own_messages=True)
   
   # send a message
   # 构造一个can的Message类
   message = can.Message(arbitration_id=188, is_extended_id=True,
                         data=[01, 00, 00, 00])
   bus.send(message, timeout=0.2)
   
   # 循环一定要的
   while 1:
       bus.send(message, timeout=0.2)
   ```

   结果：灯不亮，抓包找不到id: 188:

   ![image-20201125165522143](sqimg/note_200701_%E9%BB%91%E6%B1%BD%E8%BD%A6%E6%A8%A1%E6%8B%9F%E7%8E%AF%E5%A2%83-ICSim_img/image-20201125165522143.png)

   分析，确认代码问题如下：

   * arbitration_id=188，这是int变量，直接给188应该默认处理成10进制数据；应该用0x188。

     ```
     >>> print(188)
     188
     >>> print(0x188)
     392
     ```

   * is_extended_id=True, 这个意思使用扩展can id，29位can id；而一般使用的标准can id，11位can id；模拟器应该使用标准can id

     （使用true，cansniffer 会抓到一个包00000000188，具体几个0忘了，反正长....)

     参考：doc_201125_一张图解释标准CanID和扩展CanID.md

   * data=[01, 00, 00, 00]，同样道理，这个也不好，会被误会成十进制数字！！

   成功代码：

   ```
   #!/bin/python3
   # -*- coding: UTF-8 -*-
   
   # import the library
   import can
   
   # create a bus instance
   # many other interfaces are supported as well (see below)
   bus = can.Bus(interface='socketcan',
                 channel='vcan0',
                 receive_own_messages=True)
   
   # send a message
   # 构造一个can的Message类
   # message = can.Message(arbitration_id=188, is_extended_id=True,
   #                         data=[01, 00, 00, 00])
   
   message = can.Message(arbitration_id=0x188, is_extended_id=False,
                        data=[0x01, 0x00, 0x00, 0x00])
   # 查看下数据设定：
   print(message.arbitration_id)
   print(message.data)
   
   # 发一个数据包测试下：
   bus.send(message, timeout=0.2)
   
   # 循环一定要的，效果马上来
   while 1:
       bus.send(message, timeout=0.2)
       print ("can sended")
   ```

   效果不上了。

4. 抓包数据本地化：（ 实际上，依靠上边两个功能可以开始日can设备了。**思路： 抓包-数据库sqlite存储-分析-发包**）

   ```
   #!/bin/python3
   # -*- coding: UTF-8 -*-
   
   # import the library
   import can
   
   # create a bus instance
   # many other interfaces are supported as well (see below)
   bus = can.Bus(interface='socketcan',
                 channel='vcan0',
                 receive_own_messages=True)
   
   # or use an asynchronous notifier
   notifier = can.Notifier(bus, [can.Logger("recorded.log"), can.Printer()])
   ```

   又是直接按网上搞的，结果，生成一个log文件居然是空的：

   ```
   # root @ archlinux in ~/test/ICSim [17:29:52] 
   $ ls recorded.log
   recorded.log
   # root @ archlinux in ~/test/ICSim [17:51:39] 
   $ cat recorded.log
   空的。。。。。
   # root @ archlinux in ~/test/ICSim [17:51:43] 
   ```

   暂时不想折腾它。一个软件开发的逻辑，notifier是通知消息，定义一个bus接口，然后[]里边是listener，监听器！can.Logger("recorded.log")，can.Printer()

5. python抓包存到数据库：sqlite3 > .exit ( sqlite的sql命令格式，前边要加. )

   **sqlite3 数据库表查看步骤：** sqlite 命令参考

   ```
   1          sqlite3 local.db
   2         .mode column
   3         .headers on
   4         .tables
   5          select * from tablename;
   6          ;(执行完语句必须执行;号才能查看出表中数据)
   
   7         .exit 退出
   
   注意： sqlite 设置前面要加一点，如：.mode column
   
              执行sql语句最后要有分号. 如：select * from table;
   ```

   python3 cmdline操作：

   ```
   >>> import sqlite3
   >>> conn = sqlite3.connect("test.db")
   执行后在当前文件夹，会生成一个test.db的文件，就是数据库文件
   ```

   脚本吧：

   ```
   $ cat vcan0_sqlite.py
   #!/bin/python3
   # -*- coding: UTF-8 -*-
   
   # import the library
   import can
   import binascii
   import sqlite3
   
   # create a bus instance
   # many other interfaces are supported as well (see below)
   bus = can.Bus(interface='socketcan',
                 channel='vcan0',
                 receive_own_messages=True)
   
   # 操作sqlite
   # 初始化数据库
   conn = sqlite3.connect("test.db")
   # 如果没有test.db,执行后在当前文件夹，会生成一个test.db的文件，就是数据库文件
   # 数据库连接
   c = conn.cursor()
   # 成功连接数据库
   print ("Connect database successfully")
   # 创建数据库表，执行一次
   # id的 AUTOINCREMENT 只能写  INTEGER PRIMARY KEY， int不行！
   # c.execute('''
   #          CREATE TABLE CANDATA
   #          (ID INTEGER PRIMARY KEY AUTOINCREMENT,
   #          CANID CHAR(60),
   #          CANDA CHAR(60));
   #          ''')
   # print ("Table created successfully")
   
   # 测试数据插入
   c.execute('''
             INSERT INTO CANDATA
             (CANID,CANDA) VALUES
             ('0X188','01000000');
             ''')
   # 测试查询数据
   cursor = c.execute('''
                      SELECT ID,CANID,CANDA from CANDATA;
                      ''')
   for row in cursor:
       print('ID = ',row[0],'CANID = ',row[1],'CANDA = ',row[2])
   conn.commit()
   conn.close()
   ```

   参考：doc_201126_sqlite3_python基本操作.md

6. 把can抓到的数据包存到数据库：（ 实现 ： 指定can包数量，抓包）

   python实现，重点python sql字符串拼接：‘ ‘.join('a','b','c')

   ```
   #!/bin/python3
   # -*- coding: UTF-8 -*-
   
   # import the library
   import can
   import binascii
   import sqlite3
   
   # create a bus instance
   # many other interfaces are supported as well (see below)
   bus = can.Bus(interface='socketcan',
                 channel='vcan0',
                 receive_own_messages=True)
   
   # 操作sqlite
   # 初始化数据库
   conn = sqlite3.connect("test.db")
   # 如果没有test.db,执行后在当前文件夹，会生成一个test.db的文件，就是数据库文件
   # 数据库连接
   c = conn.cursor()
   # 成功连接数据库
   print ("Connect database successfully")
   # 创建数据库表，执行一次
   # id的 AUTOINCREMENT 只能写  INTEGER PRIMARY KEY， int不行！
   # c.execute('''
   #          CREATE TABLE CANDATA
   #          (ID INTEGER PRIMARY KEY AUTOINCREMENT,
   #          CANID CHAR(60),
   #          CANDA CHAR(60));
   #          ''')
   # print ("Table created successfully")
   
   # 测试数据插入
   # 抓包数据插入：
   # messages = bus.recv(100)
   # print(type(messages))
   count = 0
   nums = 100
   for msg in bus:
       # 以下使用会报错，没个“,“ 会认为是一个参数，用一个str变量封装下
       # c.execute("INSERT INTO CANDATA (CANID,CANDA) VALUES ('",str(msg.arbitration_id),"','",binascii.b2a_hex(msg.data).decode("UTF-8").replace('0X',''),"');")
       # sql_exec = "INSERT INTO CANDATA (CANID,CANDA) VALUES ('",str(msg.arbitration_id),"','",binascii.b2a_hex(msg.data).decode("UTF-8").replace('0X',''),"');"
       # 打印看一下,参数
       # print(type(sql_exec))     # 是个tuple元组
       # print(str(sql_exec))
       # c.execute(sql_exec)
       # c.execute(str(sql_exec))
       if  count < nums :
           # sql_exec = "INSERT INTO CANDATA (CANID,CANDA) VALUES ('",str(msg.arbitration_id),"','",binascii.b2a_hex(msg.data).decode("UTF-8").replace('0X',''),"');"
           # sql_exec = "INSERT INTO CANDATA (CANID,CANDA) VALUES ('",str(msg.arbitration_id),"','",str(binascii.b2a_hex(msg.data).decode("UTF-8").replace('0X','')),"');"
           # 不行，上边str()是10进制数，要格式化成16进制数字
           sql_exec = "INSERT INTO CANDATA (CANID,CANDA) VALUES ('","{:#x}".format(msg.arbitration_id),"','",str(binascii.b2a_hex(msg.data).decode("UTF-8").replace('0X','')),"');"
   
           print(str(sql_exec))    # sql_exec使用"," 和 str()不能达到拼接字串效果，结果是个元组
           print(''.join(sql_exec))    # 这才是真正的python3字串拼接！
           # print(str(msg.arbitration_id))
           # print(str(binascii.b2a_hex(msg.data).decode("UTF-8").replace('0X','')))
           # str(sql_exec) 输出带括号，是元组的字串("INSERT INTO CANDATA (CANID,CANDA) VALUES ('", '314', "','", '0000000000000037', "');"), 带括号
           # c.execute(str(sql_exec))
           c.execute(''.join(sql_exec))
       else:
           break
   
       count = count + 1
   
   # 测试查询数据
   cursor = c.execute('''
                      SELECT ID,CANID,CANDA from CANDATA;
                      ''')
   for row in cursor:
       # 格式化输出数据
       print('ID = ',row[0],'CANID = ',row[1],'CANDA = ',row[2])
   
   conn.commit()
   conn.close()
   ```

   使用sqlite3命令行，查询统计数据包：

   ```
   先清空数据库
   sqlite> delete from candata;
   sqlite> select * from candata;
   
   统计canid出现次数：
   sqlite> select canid,count(canid)  from candata group by canid;
   0x133|5
   0x136|5
   0x13a|5
   0x13f|5
   0x143|5
   0x158|6
   0x161|5
   0x164|5
   0x166|5
   0x17c|5
   0x183|5
   0x18e|5
   0x191|5
   0x1a4|3
   0x1aa|3
   0x1b0|3
   ```

   **注意：现在可以设置一个很高的nums，让抓包时间长一点，保持一直抓包。但是，直接ctrl + c 结束python是不行的， 在while循环里，没有执行commit操作，数据不落实到数据库里边。所以，ctrl + c之后，直接去select数据库，查询到是空的。**
   
   **解决：**
   
   * 第一种方式，是在每个循环的结束使用commit
   * 第二种方式，是用ui的设计思路，开启多线程，一个用来维护抓包和监听线程间共享变量；一个用来监听用户的输入和改变循环监听变量
   
   最简单，改变commit（）位置：
   
   ```
   while循环的insert执行之后，马上执行commit（）
           c.execute(''.join(sql_exec))
           conn.commit()
   ```
   
   修改之后的代码，应该可以使用ctrl + c了:
   
   待续。。试试，hackrf去了

***

# archlinux下使用ValueCAN3工具（can转usb）

刚收到公司的ValueCAN3，看样子像是苦寻已久的can转usb设备，于是研究一下怎么用。希望能早日结合can-utils和python-can分析下我的08款福美来战车。

一个小小转接头，淘宝一看居然有报价9900的，看来把我的车卖了刚刚够买个valuecan3。郁闷

![image-20201129214203368](sqimg/note_200701_%E9%BB%91%E6%B1%BD%E8%BD%A6%E6%A8%A1%E6%8B%9F%E7%8E%AF%E5%A2%83-ICSim_img/image-20201129214203368.png)

1. 查看设备：dmesg

   ```
   $dmesg
   前usb转ttl设备拔出
   [77957.822825] ch341 1-3:1.0: device disconnected
   valuecan3 设备的识别如下
   [77962.237891] usb 1-3: new full-speed USB device number 18 using xhci_hcd
   [77962.384765] usb 1-3: New USB device found, idVendor=093c, idProduct=0601, bcdDevice= 6.00
   [77962.384772] usb 1-3: New USB device strings: Mfr=1, Product=2, SerialNumber=3
   [77962.384776] usb 1-3: Product: USB to CAN Converter
   [77962.384779] usb 1-3: Manufacturer: ICS
   [77962.384782] usb 1-3: SerialNumber: 132070
   [77962.389495] ftdi_sio 1-3:1.0: FTDI USB Serial Device converter detected
   [77962.389613] usb 1-3: Detected FT232RL
   [77962.390507] usb 1-3: FTDI USB Serial Device converter now attached to ttyUSB1
   ```

   重点：

   * Product: USB to CAN Converter ： 大致声明是usb 转  can设备
   * Manufacturer: ICS ： 厂家是著名ICS
   * FTDI USB Serial Device converter detected ： 重要，注意类型，是个usb 串口转can，貌似还不是像USB2CAN那种直接识别成can0的设备，有点麻烦
   * FTDI USB Serial Device converter now attached to ttyUSB1： 表示usb串口映射到 /dev/ttyUSB1

   以上是分析获得的设备信息，但是直接查google和youtube，没发现valuecan3在linux下的使用指导。

   直到google  “valuecan3  candump" 发现些东西，直到看到这篇文章，关于linux开启can设备介绍：

   doc_201129_linux下打开can设备接口方法.md

   https://elinux.org/Bringing_CAN_interface_up

   文章把linux的can设备分三类：

   SocketCAN provides several CAN interface types:

   - virtual interfaces like vcan0
   - native (real hardware) interfaces like can0
   - SLCAN based interfaces like slcan0

   valuecan3的设备，应该属于slcan

2. 创建串口can设备：

   根据/dev/ttyUSB1创建slcan接口：（启动slcand服务，把ttyUSB串口映射成虚拟can口）

   ```
   slcand -o -s8 -t hw -S 3000000 /dev/ttyUSB1
   ```

   * 注意-s8，配置can速率
   * -S，配置串口的速率

   -s8的can速率对照：

   | ASCII Command | CAN Bitrate |
   | :-----------: | :---------: |
   |      s0       |  10 Kbit/s  |
   |      s1       |  20 Kbit/s  |
   |      s2       |  50 Kbit/s  |
   |      s3       | 100 Kbit/s  |
   |      s4       | 125 Kbit/s  |
   |      s5       | 250 Kbit/s  |
   |      s6       | 500 Kbit/s  |
   |      s7       | 800 Kbit/s  |
   |      s8       | 1000 Kbit/s |

   可以看到接口了：

   ```
   $ ip link show 
   
   7: slcan0: <NOARP,UP,LOWER_UP> mtu 16 qdisc pfifo_fast state UNKNOWN mode DEFAULT group default qlen 10
       link/can 
   ```

   启动接口：

   ```
   ip link set up slcan0
   ```

   可以抓包：

   ```
   candump slcan0
   ```

   明天可以去试试了，哈哈



