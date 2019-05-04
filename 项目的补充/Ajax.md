### Ajax总结：



##### **1、Ajax全称：Asynchronous JavaScript and XML**



##### 2、关于同步和异步：　

- 异步传输是面向字符的传输，它的单位是字符，异步传输是将比特分成小组来进行传送。一般每个小组是一个8位字符，在每个小组的头部和尾部都有一个开始位和一个停止位，它在传送过程中接收方和发送方的时钟不要求一致。
- 而同步传输是面向比特的传输，它的单位是帧，它传输的时候要求接受方和发送方的时钟是保持一致的。



##### 3、Ajax所包含的技术：

- 　ajax并非一种新的技术，而是几种原有技术的结合体。它由下列技术组合而成。
- 　使用CSS和XHTML来表示。
-    使用DOM模型来交互和动态显示。
-    使用XMLHttpRequest来和服务器进行异步通信。
-    使用javascript来绑定和调用。



##### 4、优缺点：

###### 优点：

- 使用异步方式与服务器通信，页面不需要重新加载，页面无刷新
- 按需取数据，减少服务器的负担
- 使得Web应用程序更为迅捷地响应用户交互
- AJAX基于标准化的并被广泛支持的技术，不需要下载浏览器插件或者小程序，但需要客户允许JavaScript在浏览器上执行
- 浏览器的内容和服务端代码进行分离。页面的内容全部由JAVAScript来控制，服务端负责逻辑的校验和从数据库中拿数据。

###### 缺点：

- 安全问题：将服务端的方法暴露出来，黑客可利用这一点进行攻击
- 大量JS代码，容易出错
- Ajax的无刷新重载，由于页面的变化没有刷新重载那么明显，所以容易给用户带来困扰——用户不太清楚现在的数据是新的还是已经更新过的；现有的解决有：在相关位置提示、数据更新的区域设计得比较明显、数据更新后给用户提示等
- 可能破坏浏览器后退按钮的正常行为； 
- 一些手持设备（如手机、PAD等）自带的浏览器现在还不能很好的支持Ajax

![img](/resources/Ajax.png)





***

##### **5、jQuery的Ajax代码：**

jquery中的ajax方法参数总是记不住，这里记录一下。

- 1.url: 

要求为String类型的参数，（默认为当前页地址）发送请求的地址。

- 2.type: 

要求为String类型的参数，请求方式（post或get）默认为get。注意其他http请求方法，例如put和delete也可以使用，但仅部分浏览器支持。

- 3.timeout: 

要求为Number类型的参数，设置请求超时时间（毫秒）。此设置将覆盖$.ajaxSetup()方法的全局设置。

- 4.async: 

要求为Boolean类型的参数，默认设置为true，所有请求均为异步请求。如果需要发送同步请求，请将此选项设置为false。注意，同步请求将锁住浏览器，用户其他操作必须等待请求完成才可以执行。

- 5.cache: 

要求为Boolean类型的参数，默认为true（当dataType为script时，默认为false），设置为false将不会从浏览器缓存中加载请求信息。

- 6.data: 

要求为Object或String类型的参数，发送到服务器的数据。如果已经不是字符串，将自动转换为字符串格式。get请求中将附加在url后。防止这种自动转换，可以查看　　processData选项。对象必须为key/value格式，例如{foo1:"bar1",foo2:"bar2"}转换为&foo1=bar1&foo2=bar2。如果是数组，JQuery将自动为不同值对应同一个名称。例如{foo:["bar1","bar2"]}转换为&foo=bar1&foo=bar2。

- 7.dataType: 

要求为String类型的参数，预期服务器返回的数据类型。如果不指定，JQuery将自动根据http包mime信息返回responseXML或responseText，并作为回调函数参数传递。可用的类型如下：

xml：返回XML文档，可用JQuery处理。

html：返回纯文本HTML信息；包含的script标签会在插入DOM时执行。

script：返回纯文本JavaScript代码。不会自动缓存结果。除非设置了cache参数。注意在远程请求时（不在同一个域下），所有post请求都将转为get请求。

json：返回JSON数据。

jsonp：JSONP格式。使用SONP形式调用函数时，例如myurl?callback=?，JQuery将自动替换后一个“?”为正确的函数名，以执行回调函数。

text：返回纯文本字符串。

- 8.beforeSend：

要求为Function类型的参数，发送请求前可以修改XMLHttpRequest对象的函数，例如添加自定义HTTP头。在beforeSend中如果返回false可以取消本次ajax请求。XMLHttpRequest对象是惟一的参数。

​            function(XMLHttpRequest){

​               this;   //调用本次ajax请求时传递的options参数

​            }

- 9.complete：

要求为Function类型的参数，请求完成后调用的回调函数（请求成功或失败时均调用）。参数：XMLHttpRequest对象和一个描述成功请求类型的字符串。

​          function(XMLHttpRequest, textStatus){

​             this;    //调用本次ajax请求时传递的options参数

​          }

- 10.success：要求为Function类型的参数，请求成功后调用的回调函数，有两个参数。

(1)由服务器返回，并根据dataType参数进行处理后的数据。

(2)描述状态的字符串。

function(data, textStatus){

​            //data可能是xmlDoc、jsonObj、html、text等等

​            this;  //调用本次ajax请求时传递的options参数

​         }

- 11.error:

要求为Function类型的参数，请求失败时被调用的函数。该函数有3个参数，即XMLHttpRequest对象、错误信息、捕获的错误对象(可选)。ajax事件函数如下：

​       function(XMLHttpRequest, textStatus, errorThrown){

​          //通常情况下textStatus和errorThrown只有其中一个包含信息

​          this;   //调用本次ajax请求时传递的options参数

​       }

- 12.contentType：

要求为String类型的参数，当发送信息至服务器时，内容编码类型默认为"application/x-www-form-urlencoded"。该默认值适合大多数应用场合。

- 13.dataFilter：

要求为Function类型的参数，给Ajax返回的原始数据进行预处理的函数。提供data和type两个参数。data是Ajax返回的原始数据，type是调用jQuery.ajax时提供的dataType参数。函数返回的值将由jQuery进一步处理。

​            function(data, type){

​                //返回处理后的数据

​                return data;

​            }

- 14.dataFilter：

要求为Function类型的参数，给Ajax返回的原始数据进行预处理的函数。提供data和type两个参数。data是Ajax返回的原始数据，type是调用jQuery.ajax时提供的dataType参数。函数返回的值将由jQuery进一步处理。

​            function(data, type){

​                //返回处理后的数据

​                return data;

​            }

- 15.global：

要求为Boolean类型的参数，默认为true。表示是否触发全局ajax事件。设置为false将不会触发全局ajax事件，ajaxStart或ajaxStop可用于控制各种ajax事件。

- 16.ifModified：

要求为Boolean类型的参数，默认为false。仅在服务器数据改变时获取新数据。服务器数据改变判断的依据是Last-Modified头信息。默认值是false，即忽略头信息。

- 17.jsonp：

要求为String类型的参数，在一个jsonp请求中重写回调函数的名字。该值用来替代在"callback=?"这种GET或POST请求中URL参数里的"callback"部分，例如{jsonp:'onJsonPLoad'}会导致将"onJsonPLoad=?"传给服务器。

- 18.username：

要求为String类型的参数，用于响应HTTP访问认证请求的用户名。

- 19.password：

要求为String类型的参数，用于响应HTTP访问认证请求的密码。	

- 20.processData：

要求为Boolean类型的参数，默认为true。默认情况下，发送的数据将被转换为对象（从技术角度来讲并非字符串）以配合默认内容类型"application/x-www-form-urlencoded"。如果要发送DOM树信息或者其他不希望转换的信息，请设置为false。	

- 21.scriptCharset：

要求为String类型的参数，只有当请求时dataType为"jsonp"或者"script"，并且type是GET时才会用于强制修改字符集(charset)。通常在本地和远程的内容编码不同时使用。