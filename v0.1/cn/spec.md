1 UMI交付模型  

参考 [UMI交互模型](./umi.md)  

2 Prompt片段  
2.1 ppl和ippl protocol  

![PPL Chunk](https://github.com/prompt-lang/arch/blob/master/assets/ppl.jpg#pic_center)  

prompt-lang人类视角的一个片段内容持久化后缀.ppl，模型视角的内容格式持久化后缀.ippl，pplc提供ppl与ippl文件的转换功能。 

规则：.ppl不能被模型直接加载运行  
规则：.ippl可以被模型加载运行  
规则：.ppl需要通过tokenizer工具与词表完成到.ippl的转换  
规则：.ippl需要通过工具程序与词表实现到.ppl的转换  

 
prompt输入内容模版定义在开始标签<SOI>和结束标签<EOI>  
<SOI><EOI>之间的内容表示model的一次内容输入  

Header  
用于指定  
- 安全规则  
- 加密规则  
- 数据压缩协议  
- 输出数据模态  
- 输出数据模态  
- 模态数据时间格式  
- 数据特殊解码方式  
- 推理模式  

规则PPL：
Header内容需要进行进行转换，如果这个安全规则等协议发生在模型输入前  
Header内容使用xml结构描述，接受到标签<EOH>结束Header内容块  

Content  
Content内容从<SOI>或者<EOH>标签的数据开始，到<EOT>结束  

Ex  
非文本模态内容编码，数据从<EOT>标签开始，在<EOE>结束  

片段结束  
使用<EOI>结束输入内容片段  


2.2 Mapping List  
Mapping List用于ppl与ippl的转换  
规则：相同Agent的UMI具有相同的Mapping规则。  
规则：ppl中<content></content>标签中的内容片段可以与ippl进行Mapping  
规则：ppl中<content></content>中的所有单词或者标签需要在Mapping List中进行定义  

3 Pipelined Prompts  

3.1 2ppl和2ippl protocol  
 
3.2 2ppl的顺序规则  

2ppl的chunk构成安排不一定按照一定顺序构建，按照一定的规则进行Pipeline，遵循的规则为Pipeline语序规则。  

4 Conditioned Prompts  

在2PPL管线排布的PPL可以对于输入prompt设置条件。

5 External Toolbox  
5.1 Sandbox  

对于模型计算使用AGI与外部计算器的效率值IOE-1<1 (IOE参考)[./thoughtbench.md],Sandbox中实现了系列的可以执行代码或执行代码工程环境。