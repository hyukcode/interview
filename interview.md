为什么是多agent，一个agent一直执行不行吗？
     
单agent长任务 goal-理解需求-改代码-测试-继续改-review 

问题是：1. Context 污染，2. 失败重试颗粒度太粗 ，3. 不同任务无法并行 4. 难以控制工具权限 5. 难以排查问题，很难知道问题到底失败在哪里

我的方案 goal - planner - dag (任务并行) - （t1 分析需求 t2 实现 t3 测试 t4 review）

把一个不可控的长上下文任务拆分成具有明确输入、输出、依赖和失败边界的工程任务

2. DAG 是怎么执行的

入度 indegree  

T1 和 T2 -> T3

手写dag

3. 既然同一层，为什么不全部并行？

只读 并发

读和写 串行

写和写 串行

全部并行 可能会导致 lost update - patch conflict - repository state 不确定

并发控制需要按照共享资源访问模型


4. 为什么不让LLM自己判断能不能并行

不能靠自然语言去猜

显式化 用schema表达控制信息，Runtime负责约束，LLM负责决策

5. Agent 执行失败怎么办？

完整链路是 agent failed -> failure classification  -> timeout / network / protocol / permission / user stop / execution error -> retry decision



6. 非幂等操作 不能无脑retry
	
idempotency_key   

 def execute_tool(tool_name, args, idemptency_key):
	result = db.get_execution(idempotency_key)
	if result:
		return result
	result = call_tool(tool_name, args)
	db.save_execution(idempotency_key, result)
	return result

 retry 区分错误类型 
429 / 503  -> retry
Timeout -> retry
400 参数错误 -> 不 retry
  permission denied -> 不 retry 


必须有限次数重试
1s 2s 4s 8s
 delay = min(max_delay, initial_delay * 2 ** retry_count)


7. 中断后为什么能继续
  runtime 状态  goal plan task_runs status

  event  events/t1.events.jsonl 


Tasker解决什么问题
Agent 任务执行框架
多agent
一个任务拆成多个可管理的小任务，控制这些任务如何执行
Planner 负责制定任务计划的部分
Task 一个具有明确目标和状态的子任务
DAG 描述任务之间先后依赖关系的有向无环图
Schduler 决定哪些任务现在可以执行
Executor 真正调用Claude Codex或其他执行器
Runtime 管理任务执行过程、状态和异常的运行时
Checkpoint 为中断后的恢复保存必要的执行状态
Memory 保存后续任务需要复用的信息或结果

Planner不等于Scheduler，Planner决定应该做什么，Scheduler决定现在能走什么

为什么需要DAG
显示保存依赖关系，说明哪个任务依赖哪个任务，哪些任务可以并行，某个任务失败后下游任务要暂停，是否有互相等待的循环依赖
 
代码 定义Task 
 from  dataclasses import dataclass, field

@dataclass 定义保存数据的类
 class Task:
	id: str
	description: str
	depends_on: list[str] = field(default_factory=list) 为每个实例创建自己的新列表
	status: str = “pending”

真实 Runtime 需要处理 中断、取消、重试中、状态未知等情况

DAG 校验函数
目标 检查任务ID是否重复，检查依赖的任务是否存在，检查是否有循环依赖，如果没有环返回一个合法的拓扑顺序

  def  topological_sort(tasks):
	task_map = {} # 1. 建立任务 ID 到任务对象的映射
	for task in tasks:
		if task.id in task_map:
			raise ValueError(f”duplicate task id: {task.id}")
		task_map[task.id] = task
	
	# 2. 初始化每个任务的入度
	
	indegree = {
		task_id: 0
		for task_id in task_map
	}

	# 3. 建立依赖图
	
	graph = defaultdict(list)
	for task in tasks:
		for dep_id in task.depends_on:
			if dep_id not in task_map:
				raise ValueError(f"unknown dependency: {dep_id}”)
	
	
	

	# 4. 把所有入度为 0 的任务放进队列
	
	queue = deque(
		task_id
		for task_id, degree in indegree.items()
		if degree == 0
	)
	
	order = []
	
	# 5. 依次处理可以执行的任务
	
	while queue:
		current = queue.popleft()
		order.append(current)
#graph的数据结构
		for next_task in graph[current]:
			indegree[next_task]-=1
			if indegree[next_task] == 0:
				queue.append(next_task)

	# 6. 如果有任务没有进入拓扑序列，就说明存在环
	
	if len(order) != len(tasks):
		raise ValueError(“cycle detected in task graph”)
	return order

任务是否可以实际启动，还要考虑前置任务是否成功、执行器是否可用、资源是否充足、任务是否需要审批



能用自己的话解释 Tasker 解决什么问题
能解释 Planner、Scheduler、Executor 和 Runtime 的区别
能独立写出 Task 数据类
能解释 default_factory=list 的作用
能写出拓扑排序与环检测代码
能解释拓扑排序为什么不等于实际执行
能说明当前实现与生产化目标之间的差距



面试时怎么介绍 Tasker？
可以按这个结构回答：
我做的 Tasker 定位是一个面向 Agent 任务的执行框架，核心目标是把 LLM 的任务规划和实际执行治理解耦。
在规划阶段，由模型把复杂需求拆解成任务及其依赖关系。调度层负责校验依赖图，并根据依赖关系决定任务的执行顺序和并发机会。
执行层通过统一的 Executor 接口接入不同的执行器，例如 Claude 和 Codex。运行时则需要管理任务状态、重试、幂等、上下文传递以及执行过程中的异常。
为了应对长任务和进程故障，还需要把执行状态与关键进度持久化，让系统能够判断哪些任务已经完成、哪些需要恢复，以及哪些必须先检查副作用再决定是否重试。

可能深挖
为什么不直接用一个 LLM 循环调用工具？ 简单任务可以。但复杂任务需要显式依赖、并发控制、状态持久化和恢复能力，不能只依赖对话历史。
Planner 和 Scheduler 有什么区别？ Planner 决定做哪些任务以及任务之间的逻辑关系；Scheduler 决定在当前状态下哪些任务可以执行。
DAG 排序是不是任务执行？ 不是。排序只解决依赖顺序，真正执行还需要状态管理、执行器调用、错误处理和结果持久化。
所有任务都必须交给 LLM 规划吗？ 不必。固定、稳定的业务流程可以使用确定性工作流；只有需要推理或动态拆解的部分才交给模型。




				