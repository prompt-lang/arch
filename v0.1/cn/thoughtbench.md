### Thoughtbench for Prompt-Lang



## Efficiency Items

#### IOE: inside-outside-efficiency

TODO:   

[ ] 定义不同的IOE的samples的复杂度组成 

[ ] thoughtqa-ioe-1/samples  



QA:   


可以通过ThoughtQA定义一系列测试样例,例如一段python执行代码  


```
def add(x, y):
    return x+y*2

add(3,5)
```

度量指标
 
直接的度量指标$ioe_1$:

AGI模型正确执行一个运算任务的时间为$T1(i)=agi_computer_exec(QA_i)$,如果执行失败$T1(i)=false$   
外部计算无智能计算器的正确执行相同任务的时间为$T2(i)=external_computer_exec(QA_i)$,如果执行失败$T2(i)=false$    
$$ioe_1=\mathbb{E}_{i=1}^{n}[\frac{T1(i)}{T2(i)}, i if  T1(i) and T2(i) ]$$  


#### Response with Memory，Response without memory




## Intelligence Level Items

ICS: Intelligent's Capacity and tiered Scope
