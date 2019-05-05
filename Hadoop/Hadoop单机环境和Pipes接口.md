我没亲手搭过，根据书本简单总结一下。



***

###### 单机伪分布式系统下的守护进程：

- HDFS的守护进程：NameNode，DataNode和Secondary-NameNode
- MapReduce的守护进程：JobTracker和TaskTraker





###### Hadoop自己实现了Web接口方便查看集群状态：

- NameNode为http://localhost:50070/
- JobTraker为http://localhost:50030/





###### Pipes接口是Mapper和Reducer的C++接口：

stream是Java的接口，使用标准输入输出使用户的Map和Reduce节点之间相互交流。

Pipes则使用socket作为TaskTraker与用户MapReduce进程之间的通道。



Mapper，Reducer，Partitioner使用C++编写，其他组件采用Hadoop的Java内置实现。

```C++
class WordCountMap:public HadoopPiper::Mapper{
    public:
    	HadoopPipes::TaskContext::Counter* inputWords;
    
    	void map(HadoopPipes::MapContext& context){
        	std::vector<std::string> words = 
            	HadoopUtils::splitString(context.getInputValue," ");
        	for(int i=0;i<words.size();i++){
            	context.emit(words[i], "1");
        	}
        context.increamentCounter(inputWords, words.size());
    	}
}

class WordCountReducer:public HadoopPipes::Reducer{
    public:
        HadoopPipes::TaskContext::Counter* outputWords;
    	
    	void reduce(HadoopPiper::ReduceContext& context){
        	int sum = 0;
            while(contex.nextValue()){
                sum += HadoopUtils::toInt(context.getInputValue());
            }
            context.emit(context.getInputKey(), 	HadoopHtils::toString(sum));
            context.increamentCounter(outputWord, 1);
    	}
}

int main(int argc, char *argv[]){
//模板工厂，传两个类进来，执行任务
    return HadoopPipes::runTask
        (HadoopPipes::TemplateFactroy
         <WordCountMap, WordCountReduce>());
}
```

