通过深度分析 xai-org/x-algorithm 代码库（包含 home-mixer 和 phoenix 核心推荐引擎），我为你整理了这份关于 X（原 Twitter）算法机制的详细评估与博主实操指南。

  > 核心发现： 开源版本中，具体的权重数值（如 FAVORITE_WEIGHT = 30）被封装在名为 params
  的模块中并被刻意剔除（Redacted），因此无法给出确切的数字。但是，我们可以从代码逻辑、计分公式和模型架构中推导出哪些行为最重要。
  1. 互动权重与排名逻辑 (Engagement Scoring)
  算法的核心是 Phoenix（基于 Grok 的 Transformer 模型）预测你对推文产生各种行为的概率，然后由 WeightedScorer（加权打分器）计算总分。
   * 公式： 最终得分 = Σ (权重 × 行为概率)
   * 计分项（源码中明确列出的加分因子）：
       * 基础三项： 点赞 (favorite)、回复 (reply)、转推 (retweet)。
       * 强互动项：
           * 分享（Share）： 细分为 share_via_dm（私信分享）和 share_via_copy_link（复制链接）。源码证实“复制链接”是一个独立的加分指标！
           * 引用（Quote）： 引用推文 (quote) 及其带来的点击 (quoted_click)。
           * 停留与点击： 图片点开 (photo_expand)、链接点击 (click)、主页点击 (profile_click)、停留时长 (dwell_time)。
       * 视频专属： vqv_score (Video Quality View)，只有当视频时长超过一定阈值（MIN_VIDEO_DURATION_MS）时才生效。
   * 关于“书签（Bookmark）”的真相：
       * 在源码的 WeightedScorer 和 Phoenix 预测目标中，并没有出现 `bookmark` 或 `save` 的字眼。
       * 解读：
         书签可能被归类为“点赞”的一种强形式，或者作为训练数据进入了模型底层（Embedding），但在最终排序公式中，它并没有像“复制链接”那样拥有独立的权重项。这与市面上“书签
         权重最高”的传闻有出入，或者该特性尚未同步到此开源版本。
   * 负分项（明确的降权/压制信号）：
       * 点击“不感兴趣” (not_interested)
       * 拉黑作者 (block_author)
       * 静音作者 (mute_author)
       * 举报 (report)
       * 源码中有专门的 NEGATIVE_SCORES_OFFSET 和 NEGATIVE_WEIGHTS_SUM 来处理这些负反馈，直接扣分。

  2. 账号权重与内容分发 (Reputation & Pipeline)

   * TweepCred 与 蓝标（Blue Check）：
       * 在此开源代码中，未发现显式的 is_verified 或 blue_check 加权逻辑。这说明蓝标的流量扶持可能发生在更底层的检索阶段（Candidate
         Generation）或通过闭源参数控制，而非简单的“乘以系数”。
   * 垂直度（SimClusters/Embeddings）：
       * Phoenix 模型使用了 user_embeddings（用户向量）和 history_post_embeddings（历史推文向量）。
       * 机制：
         算法不通过打标签（如“科技博主”）来分类，而是通过向量相似度。如果你过去发的推文和你现在的推文在向量空间距离近，且与你的粉丝群体的兴趣向量匹配，得分就高。跨领域
         发帖会导致向量距离变远，推荐精准度下降。
   * 多样性惩罚 (Diversity Scorer)：
       * 源码中有 AuthorDiversityScorer。如果你连续发多条推文，且都进入了候选池，算法会通过 decay_factor（衰减因子）降低你后续推文的权重，防止同一个人刷屏。

  3. 安全与过滤 (Safety & Filtering)

   * 可见性过滤 (Visibility Filtering, VF)：
       * 代码中存在 VFFilter 和 VFCandidateHydrator，它们调用外部的“可见性服务”。
       * 双重标准：
           * 关注流 (In-Network)： 使用 TimelineHome 安全级别（较宽松）。
           * 推荐流 (Out-of-Network)： 使用 TimelineHomeRecommendations 安全级别（更严格）。
       * 实操意义： 一些打擦边球的内容可能粉丝能看到，但绝不会进入“为你推荐”。
   * 去重机制：
       * RetweetDeduplicationFilter：防止同一条推文（无论是原创还是转推）多次出现。
       * DedupConversationFilter：在对话串中，只保留得分最高的那一条回复，避免评论区刷屏。

  ---

  🏆 博主 SOP：基于源码的实操指南

  根据上述代码分析，针对算法机制优化你的运营策略：

  1. 互动引导：主攻“复制链接”与“停留”
   * 不再盲目求“书签”： 既然源码中没有明确的“书签权重”，不如引导用户做源码中明确的高分动作。
   * 新的金句： 结尾话术改为 “转给需要的朋友（Share via DM）” 或 “复制链接分享（Share via Copy Link）”。这两个动作在代码中有独立的计分项 p::SHARE_VIA_DM_WEIGHT 和
     p::SHARE_VIA_COPY_LINK_WEIGHT。
   * 增加停留时长（Dwell Time）： dwell_score 是明确的加分项。发长文（Longform）或做成 Thread（串推），让用户手指停止滑动。

  2. 视频为王：利用 VQV 机制
   * 必须发视频： 代码中有一个专门的 vqv_score（视频质量观看得分）。
   * 注意时长： 只有当时长超过 MIN_VIDEO_DURATION_MS 时，这个加分项才生效。这意味着极短的动图或几秒的视频可能吃不到这个红利，建议视频长度适中（例如 15秒+）。

  3. 破解“链接降权”迷思
   * 策略： 源码中没有发现针对 http 文本的直接“正则表达式过滤器”。相反，click_score（链接点击）是加分项。
   * 但是： 如果带链接导致用户跳出 App 且不再回来，可能会影响后续的会话时长。
   * 建议： 如果必须放链接，确保它是高价值链接（用户会点击），因为 click 本身能回血。如果只是引流且点击率低，确实不如放在评论区（避免占用正文的点击预测模型资源）。

  4. 避免“连发”与“刷屏”
   * 利用多样性衰减： AuthorDiversityScorer 告诉我们，短时间内连续发推，后续推文权重会指数级下降。
   * SOP： 每天发帖要有间隔（例如间隔 2-4 小时），不要一次性把准备好的素材全发出去，否则会被算法认为是“同作者刷屏”而自我降权。

  5. 评论区运营
   * 回复权重： reply_score 很高。
   * 对话树去重： 算法会折叠低分回复。确保你的回复（或你在评论区的抢楼）是高质量的，能引发二级回复，这样才能在 DedupConversationFilter
     中胜出，成为“神评”被展示给更多人。

  一句话总结：
  不要迷信“书签”，算法更看重传播（复制链接/私信）和留存（视频完播/停留时长）。做“值得被转发给别人看”的内容，而不是“值得自己偷偷收藏”的内容。
