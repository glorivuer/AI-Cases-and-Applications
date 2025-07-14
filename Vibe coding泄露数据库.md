Vibe coding 大瓜来了，外网都转疯了，Supebase CEO 都亲自叫大家紧急断掉你的 cursor 等开发工具直接连接生产环境数据库的做法。 

https://www.generalanalysis.com/blog/supabase-mcp-blog

https://x.com/kiwicopple/status/1944393307850768424

Supebase CEO：MCP 用于开发。请勿将其连接到生产数据库。请勿将其连接到任何包含生产数据的数据库。  这不是超级建议。该建议适用于任何 MCP。

Supebase CEO：我之所以使用黑白分明的解释，是因为人们显然对攻击媒介的了解还不够，无法保护自己  如果你的 MCP 上启用了其他可以访问互联网的工具，那么“只读”选项就毫无用处了。在这种情况下，存在一个攻击向量可以读取数据并将其 POST 到外部数据存储。

疯传的是gen_analysis  的文章，揭示了攻击过程：

开发人员使用Cursor 等工具，会给代理很高的权限，会调用 Supabase MCP 服务器来查询数据库， 访问权限操作 Supabase 数据库service_role，从而绕过所有行级安全 (RLS) 保护。同时，它会读取客户提交的消息作为其输入的一部分。如果其中一条消息包含精心设计的指令，助手可能会将其解释为命令并无意中执行 SQL。

Supabase、Postgres、MySQL 都一样，只要 agent 拥有查询权限，攻击者就能“借刀杀人”。更糟的是，工单、评论、聊天窗口都能成为隐形载体，WAF 和 RBAC 根本感知不到。

没有越权、没有报警，开发者甚至以为自己在“正常检索工单”。结果？Slack、GitHub、Gmail 等 OAuth access token / refresh token 全部泄露，连过期时间都一清二楚。

