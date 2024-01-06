# 阅读方法
1. tutorial了解项目的目的. 
2. doc里的[example](https://deap.readthedocs.io/en/master/examples/index.html)了解各个算法的应用. 
3. [library reference](https://deap.readthedocs.io/en/master/api/index.html)里面比代码中的文档更多, 解释目的更多. 
# 总体结构
核心代码包含五个文件和两个文件夹:
	creator.py & base.py: 包含Fitness类, creator函数, 和Toolbox类. fitness类存储适应度值, creator存储种群的参数和对应的Fitness值. creator存储所有数据. Toolbox类存储所有算子, 进化算子, 评估算子, 种群初始化算子等等. 
	algoithm.py: 六个常用的顶层算法, 包括var变化算法, 和最简单的进化算法框架. 
	tools: 统计和记录工具, 和选择, 交叉, 变异, 迁徙, 等算子的实现
		emo.py是多目标优化相关选择算子. indicator/constraint是CMA算子. 
	benchmarks: 常用的评价指标. 
	gp.py: genetic Programming相关数据结构和算法算子.  
	cma.py: Covariance Matrix Adaptation Evolution Strategy相关
# base.py & create.py
## Fitness类
代码目的:
	Fitness只储存score, weight, valid和定义score之间的比较关系. 不负责score的计算方式. Fitness是个数据类. 对于多维fit, 还定义了dominate算子. 
	Constrained Fitness是由外界在算fit的时候, 额外赋值一个constraint项, 判断individual是否满足约束. 然后根据constrained项, 实现各种比较. 
实现方式:
	Fitness是一个抽象基类, 有两个class attributes, weights和wvalues, 都是tuple(**why?**). weights必须在init之前定义. 通常是1或-1. class attributes并不是所有class共享同一个数值..相比instance attributes只不过是初始化时间不同, 且可以由class直接调用罢了. 
	使用propety保护了内部的wvalues变量. 给外界的是没有加权的values. 还有valid property, 表示当前计算的fitness是否有效. 
	定义了dominates函数. 当一个个体的fitness的每个维度都大于等于另一个个体, 且不全等于的时候, 是dominate. dominate没法用>=来表示. 
	还定义了大于/小于等常见操作. 通过内部的wvalues的比较来实现. 
## create()函数
代码目的: 
	适用场景: 
		通过create函数注册fitness和model(individual). 注册fitness是给Fitness基类加上一个weights属性初始化. 注册individual的方法是把一个已有的类eg. list/array绑定上一个属性(通常就是fitness), 这样individual依旧是个数据类(individual数据+fit数据)
	这样写的好处是:
		1. 用户在自定义model与loss时不必手动写新类, 但也支持手写新类. 
		2. 不提供基类, 自由度极高, 用户想咋写咋写. 
		3. 代码样例给individual暗示了两个通用接口: 1). 所有individual默认都是可索引的sequence-方便种群相加/遗传变异操作 2) individual都是纯数据, 包含model和fitness; individual和打分逻辑分离. `这似乎和当下的DL架构是不一样的. `
	该方法的劣势:
		1. 类型是动态生成的, 似乎不方便写yaml配置. 似乎让用户手写[dataclass](https://realpython.com/python-data-classes/)更方便. 
		2. 代码不强制individual一定有fitness的接口. 
实现方式: 
	代码内部通过metaclass生成新的类. MetaCreator继承自type, 应用了[python元编程](https://realpython.com/python-metaclasses/). 相比于类装饰器/继承, 用户不用重写新的class定义, 而是调用一个函数. 
	creator需要处理新类型的复制, 原生python array和numpy array复制的时候不会复制attribute, creator添加了这部分的代码, 但只在输入类就是array的时候才执行, 如果输入的类是个复杂定义的类型, creator没有递归处理的能力. 
## Toolbox类
代码目的
	注册(储存+初始化)进化算法用到的各种算子. toolbox内存储partial后的函数, 而非函数调用的结果. 把不同算子注册到toolbox的同一个名字上去, 方面后续算法调用. 除了储存算子, 也储存初始化population的函数. 
	相对于用基类
		好处: 代码自由程度最高. 可以任意实现toolbox中的一部分功能, 也可以添加任意功能. 
		坏处: 没有统一接口, 没有必须实现的虚函数做sanity check. 
实现方式
	通过partial给函数传参数, 相当于做了config. 通过alias给函数添加别名, 并且setattr到Toolbox的实例上去. 
	默认有注册了两个函数, 一个map, 一个clone. 算是基函数? 
# algorithms.py
总体介绍: 
	1. 打包了六个常用函数. 六个函数共用一些toolbox的接口, 比如mate, evaluate等. 
	2. varAnd和varOr实现了一个进化算法的变化的部分(crossover, mutate, reproduction). 变化后改变的个体的fitness会invalid. 然后返回变化后的population. varAnd是两个操作都可能做, varOr是赌轮盘, crossover/mutate/repro中的一个操作. 
## eaSimple
一个最简化的进化算法的框架逻辑. 除了下述的伪代码之外, 还会记录每代的最优个体halloffame; 记录中间的统计信息logbook. 最终输出最后一代population和logbook. halloffame不用输出. 
```python
evaluate(population)
for g in range(ngen):
	population = select(population, len(population))
	offspring = varAnd(population, toolbox, cxpb, mutpb)
	evaluate(offspring)
	population = offspring
```
## eaMuPlusLambda
进化算法  (𝜇+𝜆) . 主要区别是select和var的顺序变了. 从所有的父母和子代中选择. 
```python
evaluate(population)
for g in range(ngen):
    offspring = varOr(population, toolbox, lambda_, cxpb, mutpb)
    evaluate(offspring)
    population = select(population + offspring, mu)
```
## eaMuCommaLambda
进化算法ask-tell模型. 文档中没有对generate和update的说明. 
```python
for g in range(ngen):
    population = toolbox.generate()
    evaluate(population)
    toolbox.update(population)
```
# Tools文件夹
## init
都是对list comprehension做的一个简单包装; 第一个输入项均是container. 
后面参数可以是函数和item数(initRepeat). 也可以是个自动停止的generator(initIterate); 还可以是把输入序列函数重复n次initCycle, 这个函数其实是方便用list直接初始化individual, 一般并不循环. 
## Statistics
该类负责计算统计量. 通过register注册reduce算子, 运行时用compile计算. 
具体实现: register就是partial, compile就是先用key函数(默认是identity)拿到value, 再for循环调用不同的算子. 
### MultiStatistics
该类可以注册多个Statistics类, 这些类的区别是key不同! 比如gp里面需要同时记录个体的len和fit. 而这两都需要min/max/avg. 
## Logbook
代码目的: 
	该类负责记录/显示统计量. 核心是header, 是每个column的名字, header可以自动推导出来. 每次record应记录header中的所有内容. 通过stream打印所有record中尚未打印过的内容. 除了自己的header, 还通过chapter嵌套保存其他Logbook. chapter实际是multistats的接口, 用于存储multistats的结果. 
实现方法:
	继承自list. record函数时把info append进自身. select函数只显示某个特定的header的结果. property stream()是最常用接口, 根据bufindex调用\_\_str__函数. 
	\_\_str__函数根据给定的startindex来print. 逻辑就是把logbook中的值转换为一个str matrix, 最后再给str matrix的每一行插入'\t'控制对齐, 最后再每行之间插入'\n'. 
	float转str单独处理但没有在现实的时候控制位数. 
	columns_len是header的条目数, 似乎不需要存在? 
	也支持del item? 什么应用场景呢? 
## Hall-Of-Fame & ParetoFront
代码目的: 
	通过update函数, 记录历史上最优的若干个indvidual+fitness. 需要排除完全相同的individual. ParetoFront继承自Halloffame, 但是无法限制maxsize. 
实现方法:
	halloffame对一个排序数组不停insert和delete. 实际用priority queue即可? 
	paretofront是如果individual不被所有hof里的个体dominate, 即可以加入. 如果个体被新添加的个体dominate, 就去除. 具体的实现就是嵌套循环. 
## History
代码目的:
	显示每个个体的系谱学历史. 
## constraint.py
似乎是CMA算法专用的算子
## emo.py
存储多目标优化相关的算子. 主要是选择和排序算子. 

