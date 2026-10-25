# 算法与伪代码

> 全部用可读伪代码；变量名即含义。实现语言不限（Node/Python 均可照抄逻辑）。

## 1. 评分归一化

```text
# 提示词评估：写作 50 分，执行 60 分
if only_direct:      pct = round(direct / 50 * 100)
if both:             pct = round(((direct/50) + (execution/60)) / 2 * 100)
clamp(pct, 0, 100)

# 模拟问诊：四评委加权
weights = {doctor: 0.40, patient_sensitive: 0.25, patient_strong: 0.25, student: 0.10}
total = round(Σ role_score[i] * weights[i])
# 四位评委各自独立调用；第 5 次调用只生成评语，不改分
```

## 2. 同稿与查重

```text
# 提交时（跨人）：5 字 shingle 集合相似度
shingles(text, n=5) -> set of substrings
sim(a, b) = |A ∩ B| / max(|A|, |B|)
if min_len(a, b) < 120: skip
if sim >= 0.85 and not first_submitter: reject(409), log("rejected")
# 不调模型、不计数

# 排行榜"同稿归零"（仅提示词评估）：
# 完全同稿：归属最早提交者
# 长文本双阈值（6 字窗 Jaccard ≥0.90 且长度≥300）
#           或（40 字窗覆盖率≥0.50 且 Jaccard≥0.25 且长度≥500）
if same_work and first_submitter != me: score = 0
```

## 3. 图谱建造评分（核心）

```text
snapshot(state):
  items = normalize(nodes + edges)          # NFKC、去标点、统一空白
  refs  = sha256(each item's canonical text)
  return {items, fingerprint: sha256(sorted(refs))}

score(state):
  snap = snapshot(state)
  if no new ref (all in reviewed): return "unchanged"        # 0 调用
  if all new nodes isolated:       return "unchanged"        # 0 调用
  if fingerprint == last:          return "unchanged"        # 0 调用

  review = llm(build_prompt(snap), json=true, temp=0.2, timeout=120s)
  # review.awards: [{ref, kind, evidence}]  kind ∈ {concept, detail, mechanism, relation}

  base = 0
  for award in review.awards:
     item = items[award.ref]; if not exists: continue
     if award.kind == "relation" and not item.connected: continue      # 孤立不计
     if not fuzzy_contains(item.text, award.evidence, tol=0.06): continue
     if duplicate_in_same_kind(award): continue
     base += {concept:+2, detail:+1, mechanism:+2, relation:+2}[award.kind]

  edges, nodes_in_components = connectivity(state)
  K = min(0.7 + 0.6 * ((edges / max(nodes_in_components,1)) / 1.5), 1.3)
  Q = review.quality ? 0.7 + 0.15*(review.quality - 1)      # 1..5
                     : local_quality(state)                   # 0.7..1.15
  delta = round(base * K * Q)

  ledger.append(accepted items)      # 永久，含已删除内容
  reviewed.add(refs)
  history.append({delta, base, K, Q, structure})
  total += delta                     # 无上限

# 评分期间作品变化 → 409，不写入
# 并发：请求带 syncRequestId + baseRevision，重复请求幂等返回
```

## 4. 人体图谱作业计分

```text
for structure in submission.structures:
   entries = []
   for item in structure.items:
       if not citation_matches(item.text, item.evidence): continue   # 引用须可匹配
       entries.push({ score: 2*clinical + 3*mechanism + 1*evidence })
   entries = dedupe(entries, by="factKey + source")                   # 跨结构去重
   structure.score = Σ entries.score          # 替换旧分，不是累加
total = Σ structure.score
```

## 5. MDT 事实限分

```text
score = llm_six_dim_grade()          # 维度权重 30/20/20/15/10/5
if exam_order_wrong:   dims.check_decision = min(dims.check_decision, 12)
if diagnosis_wrong:    dims.clue = min(dims.clue, 10); dims.genetics = min(dims.genetics, 10)
if scribe_fix_ratio < 1: dims.reflection = min(dims.reflection, 3 + round(2*ratio))
total = clamp(Σ dims, 0, 100)
# 解锁自由模式：引导完成 + 提交齐全 + 正式分≥60 + ICD 正确 + 埋错全纠
# 幂等：gradingInputHash 命中直接返回旧记录
```

## 6. 更新记录（乐观并发）

```text
save(key, patch, baseRevision):
  lock(key)
  cur = read(key)
  if baseRevision != null and cur.revision != baseRevision: throw Conflict(409)
  next = merge(cur, patch); next.revision = cur.revision + 1
  write_atomic(path(key), next)      # tmp + rename
  unlock(key)
```

## 7. 个性化路径生成

```text
evidence = collect(recent records per module)
weak = items where practiced_attempts >= 2 and mastery < threshold
      # 不是最简单，也不是最难：选"练过但没成型"
cards = rule_based(weak) → {chapter, task, criteria, deadline}
if llm_enabled: cards = llm_refine(cards, evidence, json=true, temp=0)
render(cards) → 学生端；完成后回写 evidence（形成闭环）
# 不能用一段空泛评语代替卡片：必须有任务、时限、验收
```

## 8. 统计与快照

```text
# 排行榜/大屏用内存快照，避免每次全量读档
snapshot = build_at_boot(); refresh_every(2min); min_interval(10s)
if teacher_request_refresh: rebuild()

# 迭代轨迹：考试型模块取"历史最好"，累计型取累计值
trajectories[module] = type == "exam" ? max_by_attempt : cumulative
```

## 9. 限流

```text
checkRate(identity, kind, limit):
  key = course:role:id:kind:date(UTC+8)
  count[key] += 1
  if count > limit: 429
# 内存态，重启清零；与前端提示的剩余次数保持一致
```

## 10. 语音违规检测（可选模块）

```text
every 4s of student audio:
  cos = speaker_similarity(student_chunk, {patient_voice, assistant_voice})
  if cos > 0.5: consecutive += 1
  else: consecutive = 0
  if consecutive >= 2: violation = true
# 评分时若有违规：四评委全 0，total_source = "ai_voice_detected"
```
