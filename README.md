# Notes
- 随笔，学习记录
-------------
目录
-------------
* __ attribute __
  - GNU C 中关键字__attribute__可以设置函数属性，变量属性和类型属性。使用方法__attribute__((__参数__));
也可以对struct或union进行属性设置，使用方法 struct xxx{int x;}__ attribute __((__参数__));
参数可以为：aligned，packed，transparent_union，unused，may_alias.
    - aligned: 设定一个指定大小的对齐格式，而且必须是2的指数倍，如"
    ```cpp
    struct xxx {
      int x[3];
    };__attribute__((__aligned(8)));
    ```
    强制设定struct的变量在分配空间时采用8字节对齐方式，但如果超过连接器最大支持字节数是无济于事的,小于struct内最大类型也无效。测试：
    ```cpp
    #include<iostream>
    using namespace std;

    struct xxx {
        int a;
        char b;
        short c;
    }__attribute__((aligned(4))) xx;

    struct yyy {
        char a;
        int b;
        short c;
    }__attribute__((aligned(4))) yy;

    struct zzz {
        char a;
        short b;
        short c;
    }__attribute__((aligned(4))) zz;

    int main(){
        cout<<sizeof(xx)<<endl;//得到8，因为4+2+1按4对齐变成8
        cout<<sizeof(yy)<<endl;//得到12，因为(1->4)+4+2按4对齐变成12
        cout<<sizeof(zz)<<endl;//得到8，因为(1+2=3->4)+2按4对齐变成8
        return 0;
    }
    ```
    - packed: aligned属性使被设置的对象占用更多的空间，相反的，packed可以减小对象占用的空间。测试：
    ```cpp
    struct aaa {
      char a;
      short b;
      int c;
    }__attribute__((__packed__)) aa;
    cout<<sizeof(aa)<<endl;//得到7，因为1+2+4=7
    ```
    - __attribute__改变函数属性，如：
    ```cpp
    #include<iostream>
    using namespace std;

    static __attribute__((constructor)) void before(){
      cout<<"hello ";//在main函数之前执行
    }

    static __attribute__((destructor)) void after(){
      cout<<"world!"<<endl;//在main函数之后执行
    }

    int main(){
      return 0;
    }//编译运行，显示hello world
    ```
* Socket：
  - 什么是socket？
    - socket是使用标准Unix文件描述符和其他程序通讯的方式。Unix程序在执行任何形式的I/O的时候，程序都是在读或者写一个文件描述符，而一个文件描述符是和一个打开的文件相关联的整数。而这个文件可能是一个网络连接，FIFO，管道等。所以，你想和Internet上别的程序通讯的时候，你将要使用到文件描述符。如何获得网络通讯的文件描述符呢?利用系统调用socket(),它返回套接字描述符，然后再通过send()和recv()调用。
  - 需要了解的两种internet套接字类型： 
    - 流格式套接字： 是可靠的双向通讯的数据流，如果你向流套接字按顺序输出“1,2”，那么他们将按顺序“1,2”到达另一边，他们是无错误传递的，有自己的错误控制，使用传输控制协议 TCP。(SOCK_STREAM)
    - 数据包格式套接字： 也叫无连接套接字，如果你发送一个数据报，他可能会到达，可能会次序颠倒；如果他到达了，那么在这个包的内部是无错误的，使用用户数据报协议 UDP。(SOCK_DGRAM)
    - 为什么数据报格式套接字是无连接的？主要是因为他不像流格式套接字那样维持一个连接，你只要建立一个包，构造一个有目标信息的IP头，然后发出去，无需连接。收到者必须发回一个ACK包，如果在一定时间发送方没有收到ACK，将重新发送，直到得到ACK。
  - 七层协议：应用层(telnet,ftp等),表示层，会话层，传输层(TCP,UDP)，网络层(IP和路由)，数据链路层，物理层。
  - 结构体sockaddr，in_addr和sockaddr_in：
    - socket描述符的类型是int
    - 在头文件 sys/socket.h，netinet/in.h和arpa/inet.h下
    - sockaddr: sockaddr是通用的socket地址，此数据结构用做bind，connect，recvfrom，sendto等函数的参数，指明地址信息。但一般编程中并不直接对此数据结构操作，而是使用另一个与sockaddr等价的数据结构sockaddr_in。
    ```cpp
    struct sockaddr{
      unsigned short sa_family;
      char sa_data[14];
    };
    ```
       --其中sa_family是地址家族，一般都是AF_xxx的形式，代表tcp/ip协议族。
       --sa_data是14字节协议地址。
    - sockaddr_in: 
    ```cpp
    struct sockaddr_in{
      short int sin_family;//地址族，AF_xxx，在socket编程中只能是AF_INET
      unsigned short int sin_port;//端口号，2字节
      struct in_addr sin_addr;//存储IP地址，4字节
      unsigned char sin_zero[8];//8字节，实际上没什么用，是为了和sockaddr保持一样的长度
    };
    ```
       --使用sockaddr_in可以轻松处理套接字地址的基本元素。sin_zero应该使用函数bzero()或menset()来全部置零。sockaddr和sockaddr_in可以互相转换，而且指向这两个结构体的指针也可以互相代替。同时，sin_family和sa_family需一致。最后，sin_port和sin_addr必须是网络字节顺序
     - in_addr: 就是32位IP地址,4个字节(怎么让sin_addr按照网络字节顺序？)
     ```cpp
     struct in_addr {
      unsigned long s_addr;//存储4字节的IP地址(网络字节顺序)
     };
     ```
      --存储4字节的IP地址(网络字节顺序)
  - 本机转换：
    - htons()--"Host to Network Short"
    - htonl()--"Host to Network Long"
    - ntohs()--"Network to Host Short"
    - ntohl()--"Network to Host Long"
    - 为什么在sockaddr_in中，sin_addr和sin_port需要转换成网络字节顺序，而sin_family不需要呢？因为sin_addr和sin_port封装在包的IP和UDP层。因此他们必须是网络字节顺序。但是sin_family域只是被内核使用来决定在数据结构中包含什么类型的地址，所以他必须是本机字节顺序。同时，sin_family没有发送到网络上，所以是本机字节顺序。
  - 处理IP地址：
    - 用函数inet_addr()可以将IP地址从点格式转换成无符号长整数型。使用方法：\
      struct sockaddr_ina; \
      ina.sin_addr = inet_addr("192.168.0.1");\
      inet_addr()返回的地址已经是网络字节格式，所以没必要再使用htonl()
    - 函数inet_ntoa()可以将一个sin_addr转换成点数格式
  - socket()函数：
    - 需要头文件sys/types.h和sys/socket.h
    - 根据地址族，数据类型和协议来分配一个套接字及其所用资源
    - int socket(int domain,int type,int protocol);
    - domain应该设置成AF_INET，可以传inta.sin_family
    - type告诉内核是SOCK_STREAM还是SOCK_DGRAM类型
    - protocol用来指定socket所使用的传输协议编号，其中IPPROTO_TCP为6，IPPROTO_IP为0，IPPROTO_UDP为17
    - socket返回以后在系统调用中可能用到的socket描述符，在错误时返回-1.全局变量errno中存储返回的错误值。
   - bind()函数：
     - 需要头文件sys/types.h和sys/socket.h
     - 一旦有了一个套接字，可能就需要将套接字和设备上的端口关联起来，如果你想用listen()来监听一定端口的数据这步是必须的，如果只想用connect()，那么这个步骤没有必要。
     - int bind(int sockfd,struct sockaddr *my_addr,int addrlen);
     - sockfd是调用socket()返回的文件描述符
     - my_addr是指向sockaddr的指针，一般我们需要传ina,也就是sockaddr_in,所以一般传(const struct sockaddr*)&ina，他保存自己的地址(端口和IP信息)
     - addrlen设置为sizeof(struct sockaddr_in)也就是16字节
     - bind成功返回0；出错返回－1，相应地设定全局变量errno。
   - socket() + bind()实例:
     ```cpp
     int main(){
       int serverfd = -1;
       struct sockaddr_in ina;

       ina.sin_family = AF_INET;
       ina.sin_port = htons(8183);//将整型变量转换成网络字节顺序，TCP中规定好的一种数据表示格式，大端
       ina.sin_addr.s_addr = inet_addr("172.27.138.8");//转换成32位IP地址，也就是8位十六进制数
    
       serverfd = socket(ina.sin_family,SOCK_STREAM,IPPROTO_TCP); 
       if(serverfd < 0){//socket错误检查
        cout<<"创建server的socket套接字失败"<<endl;
        return -1;
       }
 
       bzero(&(ina.sin_zero), 8);

       int ret = bind(serverfd,(const struct sockaddr*)&ina,sizeof(ina));
       if(ret < 0){
        cout<<"绑定本地地址、端口失败"<<endl;
        return -1;
       }
       return 0;
      }
     ```
   --其中bzero在头文件string.h中，置零
    
