### JSON类库：

JSON（JavaScript Object Notation， JavaScript对象表示法）是一种轻量级的数据交换格式。

JSON有两种结构：

1. 多个key-value串在一起，组成一个集合。
2. 值的有序列表。如array，vector。

```C++
Json : :Value json_ temp;
json_temp [ ” name ” ] = Json: :Value( "sharexu " ) ;
json_ temp [” age ” l = Json : :Value(l8) ;
Json : :Value root ;
root [” key_ string ” l = Json: :Value ( ” value_string ” );
root [ ” key_number ” ] = Json : :Value(12345) ;
root [ ” key_ boolean ” ] = Json: :Value(false);
root [” key_double ” l = Json :: Value(l2 . 345) ;
root ［ ” key一object ” ］ = json_temp ;
root [ ” key_array ” l . append ( ” array_string ”) ;
root [” key_ array ” l . append(1234) ; 
```

![JSON](/resources/JSON.jpg)

JsonCpp主要包含3个类：Value，Reader，Writer

JSON::Value是最基本最重要的类，用来表示各种类型的对象。

Jsoncpp 的 Json::Writer 类是一个纯虚类，并不能直接使用。在此使用 Json::Writer 的子
类： Json: :FastWriter 、 Json::StyledWriter 、Json::StyledStreamWriter。其中 Json::FastWriter 来处理 JSON 是最快的 。 

```C++
Json:: FastWriter fast_writer;
std :: cout << styled_writer.write(root); 
```

Json::Reader 是用于读取的，确切地说，是用于将**字符串转换为Json::Value 对象**的 。 

```C++
//将字符转义，如/"
string str_test ＝ ”{＼”id＼”：1,＼”name＼”：＼”pacozhong＼”｝”；
Json::Reader reader;
Json::Value value ;
if (!reader.parse(str test, value))
	return O; 
```



##### JSON用途：

​	由于很多页面都是用 Java Script 写的，而使用 Java Script 解析 JSON 又非常方便，所以很多 CGI 都是用 JSON 与页面进行通信的 。
​	JSON 可以以字符串的形式存储，要使用时再进行序列化 。 





### **这里要提到两个概念，序列化和反序列化。** 

- ### **序列化是将对象状态转换为可保持或传输的格式的过程。** 

- ### **与序列化相对的是反序列化，它将流转换为对象。**  

