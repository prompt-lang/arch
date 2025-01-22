## Unified Machine Interface

#### umi-1 交互模型

交互模型定义AGI智能体与环境接入的数据接口及接入方法，包含程序性的交互。


1.1 A-A-Human  


    Agent(Task) \<-\> Agent(Delegate) \<-\> Human  

    Delegate Agent主要的功能实现和Human人机交互，或者和外部进行交付，处理交互数据等，Task Agent更关注于任务相关的信息及策略执行  


1.2 self-loop  

    Agent(time i) -> Agent(time i+n)  

    在time i时刻Agent保留了prompt信息发送给在time i+n时刻使用  
    
    1.2.1 使用external processor去记忆并触发这个prompt  

    1.2.2 使用step-by-step的方式处理，Agent的每一个处理步骤需要理解这个prompt是否需要被处理  

1.3 multi-loop task

    Agent通过多次输出执行完成task

    1.1 Task是复杂的不能解决，经过多轮次来完成任务

    1.2 Task是伴随的，例如游戏，在这一个周期内不断的产生动作输出来完成这个周期内的任务

1.4 seek-for-prompt

    Agent通过获取更多的Prompt来获取信息

1.5 Prompt Rank

    收到多个prompt，在处理时prompt的可以设定排序，或者按照优先级计算

1.6 Realtime Prompt


    接入Prompt信号为实时的交互反馈

1.7 Session



1.8 Prompt Modeling


1.9 Prompt Canvas

    提示视图，与视图提示


#### AGI输入输出建模 Input and Output Modeling

   2.1 针对输入的的可以使用统一的建模技术，例如通过不同的tokenizer实现不同的输入建模的处理，AGI主模型具备对于全量输入的处理，对于输入的建模为输入建模

   2.2 在非文本输入，如进行动作控制领域的输入，语音生成等的非文本输出的建模为输出建模


Reference 

[Prompt Chain](https://www.promptingguide.ai/zh/techniques/prompt_chaining)  

[RAG](https://ai.meta.com/blog/retrieval-augmented-generation-streamlining-the-creation-of-intelligent-natural-language-processing-models/)  

[Anthropic MCP](https://www.anthropic.com/news/model-context-protocol)  

[OpenAI Swarm](https://github.com/openai/swarm)   

[APPL](https://github.com/appl-team/appl)    

[PromptWizard](https://arxiv.org/abs/2405.18369)  
