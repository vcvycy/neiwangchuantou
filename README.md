程序约定：
(1)程序热部署：即可以随时添加数据转发端口(修改conf.ini,每次请求都会读取conf.ini文件)
(2)服务主端口默认为5010
(3)函数以大写开头，变量以小写开头
(4) Server: 中继服务器(如阿里云)
    Client: User发送到Server的数据将转发到Client
    User  : 任何想连接到Client，但由于Client在内网或无静态IP，从而连接Server中继的用户。
(5) Server上的几种TCP连接：
   (a) MainPort(默认5010): Server 接受Client连接，身份认证等等
   (b) IntermediatePort: 中间端口，监听并接受user的连接，数据将转发到conf.ini配置的对应的ClientName。
                         在此端口Accept的每一个TCP都有一个对应的TempPort连接对应。
   (c) TempPort        : 临时端口，在IntermediatePort接受一个TCP连接后，便会随机选取一个端口(TempPort),
                         并通过MainPort告诉对应此端口数值，从而建立一条新的连接。与上述端口共同作为数据中继节点。
                         此端口只Accept一条连接。
Server：作为TCP连接的中继
一、配置信息(conf.ini)(热部署)
  IntermediatePort1  ClientName1 ClientPort1
  IntermediatePort2  ClientName2 ClientPort2
  ...
  IntermediatePortN  ClientNameN ClientPortN
  表示：所有连接Server IntermediatePort的连接都会转发到ClientName客户端中的ClientPort端口。
二、程序功能
  1、与client交互
     监听端口(5010),接受Client的连接
     Attribute：
         (1) clientConn  记录所有连接到服务器的client。#dict类型,client["ClientName"]=ClientSocket，将ClientName映射到Socket
         (2) 
     Method:
         (1) ConnVerify(socket) #TCP连接Socket身份验证，即询问ClientName(处理冲突情况),查询是否在conf.ini出现。
             若符合要求将ClientName -> ClientSocket加入clientConn
         (2) Listen(port=5010)  #监听端口5010，将TCP交给ConnVerify处理
         (3) EstablishConnOnClient(clientName,serTmpPort)
             #有数据经服务器,
         (4) 
  2、数据转发类(DataTrans)
     Attribute:
         (1)establishedConn(数组：记录所有数据转发连接)
            数组项记录dict："connUser"  -> IntermediatePort与User建立的连接
                            "connClient"-> TempPort与Client建立的连接。
                            "clientName"-> 即IntermeiatePort对应ClientName,
                            "thread"    -> 二元组(两个线程)
                            "bytesSent" -> 转发的字节数(bytesClient,bytesUser)分别表示Client和User发送的字节数。
                            "status"    -> 当前连接状态，1表示active,0表示inactive.
     Method:
         (1)AddConn(connUser,connClient,clientName) 
           #添加数据转发连接：加入establishedConn,并新建两条线程分别用于两个方向(connUser <-> connClient)数据转发
         (2)inactiveConn(eConnItem) #eConnItem为establishedConn的一项。将其status设置为0。(若对应的thread未结束，结束之)
         (3)ThreadFun(conn_1,conn_2,eConnItem): #从conn_1读取信息，发送给conn_2。若某条连接关闭，调用RemoveConn。
Client
   配置信息：本机需要中转的端口
  连接服务器
  Attribute: