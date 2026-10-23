Repository -> Scanner -> Tree-sitter -> Symbol

PostgreSQL + Qdrant -> Hybrid Retrieval -> MCP  -> Coding Agent


为什么代码RAG不直接按 500 token chunk？
有限保证代码语义边界，再考虑embedding模型的长度限制
思路：Tree-sitter -> Function / Class / Method / ... -> Symbol -> Symbol内部再chunk


为什么要用 PostgreSQL + Qdrant 两套存储？
PostgreSQL 负责 source of truth, Qdrant 负责 retrieval index
Qdrant找到symbol_id，PostgreSQL用symbol_id找到完整Symbol,
事务、更新、删除、关系查询、完整代码读取 由 PostgreSQL完成更合适
企业内部RAG：业务数据库 = truth Vector DB = secondary index


Dense 和 Sparse 为什么一起用
用户搜索 UserService 精确词，用Sparse
搜索 找到负责用户登录鉴权的代码，用Dense
Query - Dense / Symbol BM25 - Fusion - TopK
RRF 的rank fusion
def rrf(rank_lists, k=60):
    scores = {}
    for weight, results in rank_lists:
        for rank, item_id in enumrate(results, start=1):
            scores[item_id] = (
                scores.get(item_id, 0)
                + weight / (k+rank)
            )
    return sorted(
        scores.items(),
        key=lambda x: x[1],
        reverse=True
    )
为什么不用原始 similarity score 相加？
因为不同的检索系统的score distribution不一致，RRF只依赖rank，相对更稳定，
也避免额外做score calibration


大Symbol内存问题
表现：huge symbol -> embedding -> 内存急剧增加
解决方向：symbol -> symbol chunker -> 8k 8k 8k ... -> batch = 2 -> embed -> upsert -> 释放 batch


MCP在系统里是什么？
Claude Code/Codex -> Repo Retriever MCP Server (search_code / get_symbol / get_code_map) -> PostgreSQL + Qdrant
MCP 是 Agent Runtime 和 领域能力之间的一层稳定协议。Retriever怎么存、检索可以怎么变化，但Agent看到的是稳定的tool contract


为什么拆成 search_code / get_symbol / get_code_map
search_code 定位候选
get_symbol 读取完整实现
get_code_map 理解上下游关系
Tool设计应该是窄接口、结构化、可组合