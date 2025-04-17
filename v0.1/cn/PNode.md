##PNode

用于处理AGI和任务节点的交互等功能，可以理解维PNode是一种神经任务节点，他不具有Thought的能力，但是能够快速的接受AGI模型的任务模板高效的执行计算。  

1.1 PNode执行AGI模型规划的模板任务  

AGI模型将部分不需要Thought的任务下发到PNode去执行，PNode按照AGI输出的Prompt Program执行任务。  


1.2 PNode Protocal  

针对于AGI模型控制PNode执行状态的协议，例如中断处理，模板任务更新等。  


Ref.  

[PNode](github.com/prompt-lang/pnode)  
