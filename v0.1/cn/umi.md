## Unified Machine Interface

#### umi-1 交互模型

交互模型定义AGI智能体与环境接入的数据接口及接入方法，包含程序性的交互。


1.1 A-A-Human  


    Agent(Task) \<-\> Agent(Delegate) \<-\> Human  

    Delegate Agent主要的功能实现和Human人机交互，或者和外部进行交付，处理交互数据等，Task Agent更关注于任务相关的信息及策略执行  


1.2 Self Loop  

    Agent(time i) -> Agent(time i+n)  

    在time i时刻Agent保留了prompt信息发送给在time i+n时刻使用  
    
    1.2.1 使用external processor去记忆并触发这个prompt  

    1.2.2 使用step-by-step的方式处理，Agent的每一个处理步骤需要理解这个prompt是否需要被处理  


Reference 

[Prompt Chain](https://www.promptingguide.ai/zh/techniques/prompt_chaining)  

[RAG](https://ai.meta.com/blog/retrieval-augmented-generation-streamlining-the-creation-of-intelligent-natural-language-processing-models/)  

