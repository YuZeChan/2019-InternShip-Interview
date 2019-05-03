### CGI：

CGI（Common Gateway Interface）通用网关接口，首先CGI是一个Web服务器提供信息服务的标准接口。

通过CGI，Web服务器就是能获取客户端提交的信息，并转交给服务端的CGI程序进行处理，最后将结果返回给客户端。

其次，CGI可以理解为一个通信系统，主要由两部分组成：

- HTML页面。
- 服务器上的CGI程序。

![CGI通信](/resources/CGI通信.jpg)

在这里，Web服务器即Http服务器，CGI程序为运行在Http服务器上的脚本（与Application服务器运行servlet不同）。

CGI是一个标准化协议，能够使应用程序（通常称CGI程序或CGI脚本）同Web服务器和客户端进行交互。



服务器与CGI程序之间用过**标准输入输出**来进行数据传递，这个过程需要**环境变量**的协作。

CGI运行模式为**多进程**，每个CGI进程**只能处理一个**用户请求，激活一个CGI进程时创建连了属于该进程的环境变量。



***

#### CGI程序：

**CGI作为脚本，需事先编译完成后，放在Apache的cgi-bin目录下。**

可直接cout输出为标准输出流：

```C++
#include<iostream>
using namespace std;
int main (){
	cout << ” Content-type : text/html\r＼n＼r\n ”;
    //第一行指明浏览器显示的文本类型
	cout << ” <html>\n”;
	cout << ” <head>\n ”;
	cout << ”<title>Hello World - First CGI Program</title>\n”;
	cout << ”</head>\n ”;
	cout << ”<body>\n”;
	cout << ”<h2>Hello World! This is my first CGI program</h2>\n”;
	cout << ”＜／ body ＞飞 n ”;
	cout << ”</html>\n ” ;
	return 0;
}
```





***

#### CGI的环境变量：

CGI 是一个进程 ，且在处理完一个请求后就退出， 在下一个请求到来时再创建一个新进程。 

CGI继承了系统的环境变量。CGI环境变量在CGI程序启动时初始化，在结束时销毁。

- 当一个CGI程序未被调用时，其环境变量几乎是系统的环境变量。
- 当被HTTP服务器调用时，其环境变量会增加HTTP服务器、客户端、CGI传输过程等项目。
- **获取环境变量使用char* getenv()函数，依赖头文件stdlib.h**

![CGI环境变量](/resources/CGI环境变量.jpg)

详见P390



##### GET方法：

在此方法下，服务器将从标准输入接收到的数据，**编码到环境变量QUERY_STRING**(或PATH_INFO)，需要从其中分离出数据。

```C++
char *value = getenv(”QUERY_STRING”)；
```



##### POST方法：

在此方法下，CGI程序可以直接从服务器的标准输入中获取数据，但要先从**CONTENT_LENGTH这个环境变量中得到参数的长度**。

```C++
char *lenstr = getenv(”CONTENT_LENGTH”)；
if(lenstr != NULL){
	int len=atoi(lenstr);
	//然后用 fgets 函数从标准输入 stdin 中获得 len+l 长度的内容，代码如下：
	fgets(poststr , len+l , stdin);
	//再利用 sscanf 函数获取m、n变量的值，(正则表达式)
	sscanf(poststr ，” m＝%[^&]&n＝%s”， m, n);
}

```



***

##### COOKIE：

用作：会话跟踪。

HTTP协议无状态，引入COOKIE机制，弥补HTTP协议无状态的不足。

**Cookie 实际上是一小段的文本信息 ，是客户端请求服务器，服务器response给客户端的用来记录用户状态的。**

客户端把COOKIE保存起来，再次请求该网站时，会随request一起穿过去。

**<u>*Cookie 具有不可跨域名性 。</u>*** 

```C++
const char* cookie = getenv(”HTTP_ COOKIE”)； 
```



保存登录信息多种方案：

- **直接保存用户名密码**（危险）
- **密码加密**后保存，或保存**时间戳**（淘汰）
- 只在登录时查询一次数据库，以后访问验证登录信息时不再查询数据库 。 **实现方式是把账号按照一定的规则加密后，连同账号一块保存到 Cookie 中** 。 
  下次访问时只需要判断账号的**加密规则**是否正确即可 。 





***



### FastCGI：

CGI多进程，适合并发量小，访问量小的情况。

FastCGI是一个额“常驻型”CGI，一直执行，不用每次都fork一个去执行。

1. Web Server 启动时载入 **FastCGI 进程管理器**（IIS ISAPI 或 Apache Module）。
2. FastCGI进程管理器自身初始化，**启动多个CGI进程**并等待来自Web服务器的连接 。
3. 当客户端请求到达 Web Server 时， FastCGI 进程管理器选择并连接到一个 FastCGI进程 。 Web 服务器将 CGI 环境变量和标准输入发送到 FastCGI 进程 。
4. FastCGI 子进程完成处理后将标准输出和错误信息从同一连接返回 Web Server。FastCGI 子进程关闭连接时，请求便被告知处理完成 。 **FastCGI 进程接着等待并处理来自FastCGI 进程管理器（运行在 Web 服务器中）的下一个连接 。** 

又感觉是类似多路复用IO的感觉。







贴一个完整的例子，get方法。

```C++
#include<iostream>
#include<stdlib.h>
#include<vector>
#include<string>
using namespace std;
vector<string> StringSplit(const string& sData, const string& sDelim) {
	vector<string> vItems ;
	vItems.clear();
	string::size_ type bpos = 0;
	string::size_ type epos = 0;
	string::size_ type nlen = sDelim.size();
    //查找字符串a是否包含子串b,不是用strA.find(strB) > 0 而是 strA.find(strB) != string:npos
	while ( (epos=sData . find(sDelim, epos)) != string::npos){
		vitems . push_ back(sData . substr(bpos , epos-bpos));
		epos += nlen;
		bpos = epos;
    }
	vItems.push_back(sData.substr(bpos, sData.size()-bpos));
	return vItems;
}

int main() {
cout << " Content-type : text/html\r\n\r\n "; 
cout << " <html>\n";
cout << " <head>\n";
cout << " <title>Hello World - First CGI Program</title>\n" ;
cout << " </head>\n "；
cout << " <body>\n ";
cout << " get parameter: <br/> ";
char *value = getenv (”QUERY_STRING”) ;
if(value != 0)(
	/* ” a=l&b=2&c=3 ” */
	vector<string> paras = StringSplit((const string)value, "&");
	vector<string>::iterator iter = paras.begin() ;
	for(;iter != paras.end();iter++){
		vector<string> singlepara = StringSplit(*iter,”= ” );
		cout<< singlepara[0] << " " << singlepara[1] << "<br/>";
	}
	cout << "</body>\n"；
	cout << "</html>\n";
	return 0;
}
```

