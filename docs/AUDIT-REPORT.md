# 「涨潮归 · 个人办公工作台」源代码审计与 v31 修复报告

- 审计对象：`https://www.workbuddy.cn/space/d/EssoxITW3BtDiOywnDSARJ`（kind=`web`，编辑态 version 40，单文件 `index.html`，8224 行 / 537 KB）
- 基线：v30（源码内联版本标记最高到 `【v30】`）
- 交付版本：v31 审计修复版（8328 行 / 546 KB，37 处定点替换，116 个 `// modified` 标记）
- 审计原则：稳定性优先 > 可读性 > 适度新增功能；不做架构级重构；零新增依赖；不改动 `KEY` / `dbId` / 9 字段契约 / 默认参数
- 静态校验：`node --check` 通过；平台质量工具 PQ001 / PQ005 见第 7 节说明

---

## 1. 同类方案调研小结

### 1.1 检索的同类项目与资料（假想检索：GitHub / 技术论坛 / 开源社区）

| 来源 | 类型 | 与本项目相关的核心结论 |
|---|---|---|
| TodoMVC（todomvc.com，长期维护的标杆示例集） | 高星开源项目 | 单一真源的不可变 CRUD：`load → 内存数组 → 生成新数组 → 整体写回`；任务 ID 全局唯一；视图只从 state 派生 |
| `CRUD with localStorage`（softwarejutsu 示例） | 公认最佳实践 | `JSON.parse` 必须包 try/catch；存储键带版本号（`tasks-v1`）；`crypto.randomUUID()` 生成 ID；两标签并发会相互覆盖 |
| MDN / Client-Side Storage 指南、Zustand `persist` 中间件文档 | 官方文档 / 主流状态库 | 读写都要 try/catch（隐私模式、配额）；版本化 + `migrate()`；rehydrate 时校验数据形状；写入去抖；`StorageEvent` 做跨标签同步 |
| Local-First Software（Ink & Switch 系 / 实践综述） | 行业成熟方案 | 墓碑（tombstone）表达删除；LWW + 每条记录版本戳；全局唯一 ID（UUIDv4/ULID）；schema 演进用「读时兼容 + 写时升级」 |
| Syncing localStorage to a Server（dev.to 实战复盘） | 工程复盘 | **union merge 无法表达删除**，删除会「复活」；需要 tombstone 或 `removedAt`；「查询失败 ≠ 空表」；一次只允许一个写入在飞；revision 认账避免误回推 |
| 美团技术团队《前端安全系列：XSS》 | 公认安全实践 | 不同上下文用不同转义；属性内必须转义引号；`href`/`javascript:` 白名单；优先用成熟转义函数而非自造 |
| wln.wiki《Version and Minimize localStorage Data》 | 工程规范 | 键加版本前缀、只存最小字段、迁移后清理旧键、迁移失败不得阻塞启动、整体 try/catch |

### 1.2 可借鉴的优秀做法 / 功能改进方向

1. **数据损坏不静默清空**：解析或归一化失败时先把原始串留档，再回退到空态；否则一次异常就是不可逆数据丢失。
2. **读时校验形状**：`Array.isArray` + 元素类型校验 + 非法值兜底，避免云端/手工导入的脏数据让整页渲染抛错白屏。
3. **`StorageEvent` 跨标签即时同步**：比只在 `visibilitychange`（切回标签才发现）更及时。
4. **源头净化 + 出口转义**：云端列可作为 DOM 属性值的，在入口做白名单清洗；所有进入 HTML 的用户文本统一走 `esc()`。
5. **`crypto.randomUUID()`**：避免同毫秒批量创建时的 ID 碰撞。
6. **写入失败分级提示**：`QuotaExceededError` 要告诉用户「数据还在内存里，先导出备份」。
7. **后台合并不打断输入**：数据合并与整页重绘解耦，重绘延后到空闲。
8. **损坏数据可下载自救**：本地留档 + 页面入口，形成恢复闭环。

### 1.3 成熟实现思路（本项目已具备，本次予以保留）

- 7 张云端表共用一条「拉取 → LWW 合并 → 推送」流水线，表只填字段映射（v30 已完成重构，架构合理，**本次不做架构级重构**）。
- 墓碑软删除 + `本地ID` 认身份 + `更新时间` 决胜 + legacy 业务键认领迁移。
- 快照比对（`snap` / `sig`）避免无效请求；单表并发锁 `pulling/pushing`；翻页 `MAXP` 硬上限 + 游标前进校验；查询失败回传 `null` 而非空表。
- 落后标签页不得覆盖较新存储（`storageNewerThanMine`）。
- 220ms 写入去抖、番茄钟墙钟计时且绝不调 `render()`。

### 1.4 需要规避的坑（本次据此定策）

| 坑 | 本项目现状 | 本次对策 |
|---|---|---|
| 删除被 union merge 复活 | 已有 7 张墓碑表 | 补齐遗漏的 `tombStudy`（导入 / 清空两处） |
| 旧标签覆盖新数据 | 已有 visibilitychange + beforeunload 护栏 | 新增 StorageEvent + `resig` 认账 |
| 云同步打断输入导致丢字 | `merge()` 无条件 `render()` | 延后重绘 |
| 存储配额 / 隐私模式抛错 | 统一 `tSaveFail` | 分级提示 |
| 模板字符串里属性未转义 | 多处 `value="${...}"` / `data-d="${...}"` | 出口 `esc()` + 入口白名单净化 |
| 归一化自愈把用户新数据「吃掉」 | v30 已改为「留最新」，但写成了 `Set.get()` | 修为 `Set.has()` |

### 1.5 第三方代码使用说明

**本次未直接摘抄任何第三方源码片段**，仅复用上述项目的成熟模式（墓碑、LWW、版本化键、StorageEvent、出口转义、UUID）。新增代码的行内注释已用 `[v31-xx]` 标注来源场景与用途（如 `// modified: [v31-N1] StorageEvent ...`），符合「摘抄复用需标注来源场景与用途」的约定。

---

## 2. 项目理解小结

- **定位**：个人自用的单文件离线工作台（`涨潮归 · 个人办公工作台`），纯前端 SPA（内联 CSS + 6416 行内联 JS，无构建、无外部依赖、无网络字典），界面语言 `?lang=zh|en`，暗黑/明亮主题，移动端横竖屏切换。
- **数据层**：主存 `localStorage['pd_workbench_v2']`（`S` 对象），7 张业务数组 `tasks / notes / projects / inspects / schedules / personal / study` + 7 张墓碑表 + `meta`；云端通过页面平台注入的 `__SMART_PAGE__.database` SDK 做双向同步（`query / addRecord / updateRecord / deleteRecord / onUpdated`）。
- **同步契约**：云端 9 列契约 `标题 / 优先级 / 截止日期 / 是否完成 / 分类 / 备注 / 本地ID / 更新时间 / 已删除`（个人事项表 `wJKetf6NORpSEY4Q8fgHj6`，v9/v10/v12 绑定历史以注释保留）；各表另有自己的列映射，7 个 `dbId` 硬编码。
- **身份与冲突**：`本地ID` 认身份、`更新时间` LWW、墓碑软删传播删除、legacy 行按业务键认领；`save()` 被同步内核包装为「写本地 + 排一次调度」，用快照比对决定哪张表真变了。
- **模块**：今日焦点、今天待办、灵感速记、专项跟进、检查台账（含批量导入/导出 Excel）、日程（月/周视图 + 系列日程）、个人事项（多选分类标签）、学习打卡（八维评分 + 小时时间块 + 番茄钟）、每周复盘、全局搜索、数据说明（导出 JSON 备份 / 导入恢复 / 云端自检 / 清空）。
- **原有设计约束（本次严格不破坏）**：不删可用逻辑、接口与调用方式兼容、不改 `KEY` 与 `dbId`、不改默认参数、不引入依赖、单文件可离线打开（`__SMART_PAGE__` 缺失时 `CloudSync` 返回 `null`，页面降级为纯本地模式）。
- **预期输入**：用户表单、批量导入的 Excel/文本、JSON 备份、云端表记录（可能被其他设备或表格视图手工编辑）。**预期输出**：渲染后的 DOM、localStorage 存档、云端记录、Excel/JSON 导出文件。

> 结论：这是一个「本地优先 + 云端最终一致」的个人数据应用。最高风险不在 UI，而在**启动期状态归一化**和**同步合并**——任何一处抛错都可能让整页白屏或用空库覆盖真实数据。本次优化全部集中在这两条链路，未改动业务玩法与视觉。

---

## 3. 审计问题汇总

> 行号为 **v30 基线** 行号；「级别」P0 致命 / P1 高 / P2 中 / P3 观察项。

| # | 级别 | 代码位置 | 问题 | 后果 |
|---|---|---|---|---|
| A01 | **P0** | 3580 `normalize()` 学习打卡去重 | `const keep = new Set(...)`，却调用 `keep.get(x.date)`；`Set` 没有 `.get` | 只要 `S.study` 有任意一条带日期的记录，`normalize()` 抛 `TypeError` → `load()` catch → `S = blank()` → 启动流程 `rollover()` 调 `save(true)` **把空库写回 localStorage，真实数据不可逆丢失** |
| A02 | **P1** | 3480-3493 `load()` | 解析/归一化失败直接 `S = blank()`，既不备份原始串，也不区分失败阶段 | 任何一次脏数据 = 静默清空 + 下次保存覆盖，无自救通道 |
| A03 | **P1** | 3495-3596 `normalize()` | `idBackfill` 跳过了 `null`/非对象元素，但后续 `S.tasks.forEach(k => { if (k.pid === undefined)... })`、`S.inspects.forEach(x => x.seq)` 等直接解引用 | 云端脏行 / 手工导入 `[null]` → 启动抛错白屏 |
| A04 | **P1** | 5893-5909 `importBackupFile()` | 导入后未走 `normalize()`；`meta` 未校验；墓碑合并漏 `tombStudy` | 导入旧备份后白屏；刚删的学习打卡记录被云端旧行「复活」 |
| A05 | **P1** | 6430-6435 `case 'wipe'` | 清空时未对 `S.study` 登记 `tombStudy` 墓碑，`tk` 也漏 `tombStudy` | 「清空全部」后学习打卡从云端复活，删除不彻底 |
| A06 | **P1** | 7621 `queryAll()` | `db.query(...)` 同步抛错未捕获（SDK 包装层会 rethrow） | 异常冒泡到 `boot()` 的串行 `step()`，启动链中断，后续 6 张表**永远不再拉取** |
| A07 | **P1** | 4725/4895/4901/4902/6452/6541/6542/6700/6701/5409/5509/5510、3104、4754、4176/4196/4646/5702 | 用户/云端可控值未经 `esc()` 直接进 `value=`、`data-d=` 属性或 HTML 文本 | 属性注入 / 存储型 XSS（云端表格被手工编辑即可触发） |
| A08 | **P1** | 7649-7652 `rowOf()` | `本地ID` 列原样作为 `x.id`，最终写进 54 处 `data-id="..."` 属性 | 云端列被填 `"` 即可闭合属性注入事件处理器 |
| A09 | **P1** | 7771 `merge()` | 云端合并后无条件 `render()`；`render()` 会整体替换 `#view.innerHTML` | 用户正在输入 / 弹层打开时被整页重绘，输入框失焦丢字（v17 番茄钟注释已明确记录过这个坑） |
| A10 | **P2** | 3152 `uid()` | `Date.now().toString(36) + Math.random().toString(36).slice(2,7)` | 同毫秒批量创建（批量导入、系列日程）存在 ID 碰撞概率，碰撞后 LWW 互顶 |
| A11 | **P2** | 7661 `propsOf()` | `更新时间: { number: x.updatedAt || Date.now() }` 未强制数值 | 云端脏值把字符串写进 number 列，upsert 失败且难定位 |
| A12 | **P2** | 7855-7859 `push()` catch | 异常分支只置 `t.pushing = false`，未置 `finished = true` | 已发出的异步 `done()` 迟到最后仍会执行 `finish()`，把 `t.sig` 改写成当前快照，**吞掉本应推送的改动** |
| A13 | **P2** | 3605-3606 `save()` | 所有写入失败统一提示 `tSaveFail` | 配额满/隐私模式时用户不知道数据还在内存、该先导出备份 |
| A14 | **P2** | 3613-3620 | 跨标签只靠 `visibilitychange`，要切回标签才同步 | 多标签并行编辑时 stale 视图持续存在 |
| A15 | **P2** | 3618 `visibilitychange` | `load()` 后 CloudSync 各表的 `t.sig` 仍是旧快照 | 另一标签的改动被误判为「本地新改动」，触发全表回推（写放大） |
| A16 | **P2** | 3094/3096 `dayDiff()` / `human()`、日期字段 | 非 `YYYY-MM-DD` 的脏日期产生 `NaN`，比较与日历渲染错位 | 逾期判断、排序、提醒异常（不崩溃但结果错） |
| A17 | **P3** | 7688-7690 `buildIndex()` | 空 `legacyKey` 会让多条缺键云端行互认副本，进而被 `ix.dup` 硬删 | 极端脏数据下误删云端行（各表 `valid()` 已要求业务键非空，实际概率低） |
| A18 | **P3** | 5868 `runCloudDiag()` | 只查第一页 200 行，自检「云端行数」用 `200+` 表示 | 自检数字不是真实总量（已有 `+` 标识，属展示口径） |
| A19 | **P3** | 平台质量工具 | `PQ001` 报括号不配平；`PQ005` 报 innerHTML 动态拼接 | 见第 7 节：PQ001 为误报（`node --check` 通过）；PQ005 是模板渲染架构的粗粒度告警 |

---

## 4. 变更总览

### 4.1 修复项（P0/P1/P2 共 15 类，37 处定点替换）

| 组 | 修复内容 | 对应问题 | 替换编号 |
|---|---|---|---|
| F01 | `Set.get` → `Set.has` | A01 | R21 |
| F02 | `load()` 重写 + `quarantineRaw()` 损坏数据隔离 | A02 | R19 |
| F03 | `normalize()` 过滤非法元素 + 非法日期兜底 | A03 / A16 | R20（含 R38 内联） |
| F04 | 导入恢复：补 `tombStudy` / `meta` 校验 / `normalize()` | A04 | R25、R26 |
| F05 | 清空补齐学习打卡墓碑 | A05 | R27 |
| F06 | `queryAll()` 同步异常捕获 | A06 | R31 |
| F07 | 18 处属性/文本出口 `esc()` | A07 | R02-R18、R06 |
| F08 | `rowOf()` 云端 `本地ID` 入口白名单净化 | A08 | R37 |
| F09 | `merge()` 输入/弹层期间延后重绘 | A09 | R32、R34 |
| F10 | `uid()` 改用 `crypto.randomUUID`（带降级） | A10 | R01 |
| F11 | `propsOf()` 更新时间数值化 | A11 | R33 |
| F12 | `push()` 异常分支 `finished` 封口 | A12 | R35 |
| F13 | `save()` 写入失败分级提示 | A13 | R22 |
| F14 | `visibilitychange` 载入后 `resig` 认账 | A15 | R23、R36 |

### 4.2 新增功能（贴合稳定性场景，克制，共 2 项）

| 组 | 功能 | 替换编号 |
|---|---|---|
| N01 | `StorageEvent` 跨标签即时同步（400ms 去抖；输入/弹层/后台标签不打断） | R24 |
| N02 | 损坏数据隔离留档 + 「数据说明」页「下载损坏备份」入口（最多保留 3 份） | R19、R28、R29、R30 |

### 4.3 重构项

**无架构级重构**。v30 的「7 表共用一条流水线」架构合理，本次仅做局部加固（把跨标签载入后的认账能力以 `CloudSync.resig()` 暴露，属接口新增而非改造）。按约束规则第 3 条，未触发「先单独输出重构方案待确认」的条件。

### 4.4 明确未改动的内容

- `KEY = 'pd_workbench_v2'`、7 个 `dbId`、9 列字段契约、DELAY/POLL/PAGE/MAXP 等默认参数、`blank()` 结构、`esc()` 原实现、全部 UI 结构与 CSS、i18n 字典、业务玩法。
- A17（`buildIndex` 空业务键）：各表 `valid()` 已保证业务键非空；若贸然 `continue` 跳过空键行，推送时 `ix.legacy['']` 匹配不到反而会 `addRecord` 重复插入，**风险大于收益**，本次不改。
- A18（自检只查首页）：已有 `200+` 标识，属展示口径，不改。
- **零新增依赖**：全部使用原生 `crypto` / `StorageEvent` / `localStorage` API，无体积与稳定性负担。

---

## 5. 分点变更详情

> 【代码位置】为 v30 基线行号；【改动代码片段】为 v31 实际代码，`// modified` 为交付标注。完整上下文见 `个人办公工作台_v31_变更对照.diff`。

### F01 【代码位置】3580 `normalize()`
- **原始问题**：`keep` 是 `new Set(best.values())`，却调用 `keep.get(x.date)`；`Set` 无 `.get`，抛 `TypeError: keep.get is not a function`（已用 Node 复现）。只要存在学习打卡记录即触发，且异常沿 `load()` → catch → `blank()` → `rollover().save(true)` 演变成**全库覆盖**。
- **修改思路**：改为 `keep.has(x)`，与相邻 3555 行日程去重的正确写法一致；保留 v30「同一天留 updatedAt 最新」的语义。
- **改动代码片段**：
```js
const keep = new Set(best.values());
S.study = S.study.filter(x => x && x.date && keep.has(x)); // modified
```

### F02 【代码位置】3480-3493 `load()` / 新增 `quarantineRaw()`
- **原始问题**：`JSON.parse` 或 `normalize()` 任一失败都直接 `S = blank()`，无原始串留档；启动阶段 `rollover()` 会 `save(true)` 把空库写回，数据不可逆丢失。
- **修改思路**：新增隔离函数把原始串写到 `pd_workbench_v2_corrupt_latest` 与带时间戳的键（最多留 3 份，防占满配额）；`normalize()` 单独包一层 try/catch——即使归一化失败，也**保留已解析的原始数据继续运行**而不是清空；只有解析本身失败才回退空态，且先留档。
- **改动代码片段**：
```js
// modified: [v31-P0] 解析/归一化失败时先把原始串隔离留档，绝不静默清空；最多保留最近 3 份
function quarantineRaw(raw, why) {
  if (!raw) return;
  try {
    const stamp = Date.now();
    const item = JSON.stringify({ at: stamp, why: why, raw: raw }); // modified
    localStorage.setItem(KEY + '_corrupt_latest', item); // modified
    localStorage.setItem(KEY + '_corrupt_' + stamp, item); // modified
    /* ...遍历 key、排序、只留最近 3 份... */
  } catch (_) {}
}
function load() {
  let raw = null;
  try {
    raw = localStorage.getItem(KEY);
    if (!raw) { S = blank(); S.meta.sample = false; save(); return; }
    const d = JSON.parse(raw);
    if (!d || typeof d !== 'object' || Array.isArray(d)) throw new Error('bad shape'); // modified
    S = Object.assign(blank(), d);
    S.meta = Object.assign(blank().meta, (d.meta && typeof d.meta === 'object') ? d.meta : {}); // modified
    ['tasks','notes','projects','inspects','schedules','personal','study'].forEach(k => { if (!Array.isArray(S[k])) S[k] = []; });
    try { normalize(); } // modified
    catch (ne) {
      quarantineRaw(raw, 'normalize: ' + (ne && ne.message)); // modified
      try { console.warn('[load] normalize failed; raw data kept as-is', ne); } catch (_) {} // modified
    }
  } catch (e) {
    quarantineRaw(raw, 'parse: ' + (e && e.message)); // modified
    S = blank(); S.meta.sample = false;
    toast(T('tLoadFail'), true);
  }
}
```

### F03 【代码位置】3495 `normalize()` 开头
- **原始问题**：`idBackfill` 会跳过 `null` 元素，但后续多处直接解引用（`k.pid`、`x.seq`、`x.target`），云端/导入脏行导致启动抛错白屏；非法日期串让 `dayDiff` 得 `NaN`。
- **修改思路**：先统一过滤 7 张数组的非法元素；再对**不参与身份/去重**的日期字段做格式兜底。日程 `date`（参与 legacy 业务键）与学习 `date`（参与按日去重）**刻意不清空**，避免引发重复插入或误删数据。
- **改动代码片段**：
```js
function normalize() {
  // modified: [v31-P1] 云端/手工导入的脏数据可能含 null 或非对象元素，先剔除再取 id，避免整页白屏
  ['tasks','notes','projects','inspects','schedules','personal','study'].forEach(k => { // modified
    S[k] = (Array.isArray(S[k]) ? S[k] : []).filter(x => x && typeof x === 'object' && !Array.isArray(x)); // modified
  }); // modified
  // modified: 日程 date 参与业务键、学习 date 参与去重，二者不动，避免清空后误删数据
  const DATE_RE = /^\d{4}-\d{2}-\d{2}$/; // modified
  const cleanD = v => (typeof v === 'string' && DATE_RE.test(v)) ? v : ''; // modified
  S.tasks.forEach(x => { if ('due' in x) x.due = cleanD(x.due); }); // modified
  S.personal.forEach(x => { if ('due' in x) x.due = cleanD(x.due); }); // modified
  S.inspects.forEach(x => { ['date','recheck'].forEach(f => { if (f in x) x[f] = cleanD(x[f]); }); }); // modified
  const ids = new Set(S.projects.map(p => p.id));
```

### F04 【代码位置】5893-5909 `importBackupFile()`
- **原始问题**：`looksOk` 只认 6 张数组；导入后直接 `save(true); render()`，未走 `normalize()`；`meta` 可能为 `null`；墓碑合并漏 `tombStudy`。
- **修改思路**：把 `study` 纳入校验；补 `meta` 与数组校验；墓碑列表补 `tombStudy`；导入后调用 `normalize()`（包 try/catch，失败不阻塞恢复）。
- **改动代码片段**：
```js
['tasks', 'personal', 'schedules', 'notes', 'projects', 'inspects', 'study'].some(k => Array.isArray(data[k])); // modified
// modified: [v31-P1] 补 tombStudy；导入后补 meta/数组校验并走 normalize，旧备份缺字段不再白屏
const tk = { tombSched: S.tombSched, /* ... */, tombPers: S.tombPers, tombStudy: S.tombStudy }; // modified
S = Object.assign(blank(), data);
S.meta = Object.assign(blank().meta, (data.meta && typeof data.meta === 'object') ? data.meta : {}); // modified
['tasks','notes','projects','inspects','schedules','personal','study'].forEach(k => { if (!Array.isArray(S[k])) S[k] = []; }); // modified
['tombSched','tombTask','tombProj','tombNote','tombIns','tombPers','tombStudy'].forEach(k => { // modified
  S[k] = Object.assign({}, S[k] || {}, tk[k] || {});
});
try { normalize(); } catch (ne) { try { console.warn('[import] normalize', ne); } catch (_) {} } // modified
save(true); render();
```

### F05 【代码位置】6430-6435 `case 'wipe'`
- **原始问题**：清空时 7 张表只登记了 6 张墓碑，学习打卡未登记，云端旧行下次拉取会复活。
- **修改思路**：补一行 `tombItems('tombStudy', S.study)`，并把 `tombStudy` 加进 `tk`。
- **改动代码片段**：
```js
tombItems('tombStudy', S.study); // modified: [v31-P1] 清空时学习打卡也要登记墓碑，否则云端旧行会复活
const tk = { tombSched: S.tombSched, /* ... */, tombStudy: S.tombStudy }; // modified
```

### F06 【代码位置】7621 `queryAll()`
- **原始问题**：`db.query()` 若同步抛错（SDK 包装层 `catch → throw e`），异常会冒泡到 `boot()` 的 `step()`，串行启动链在该表中止，后续表永不拉取。
- **修改思路**：同步调用包 try/catch，失败走与 Promise reject 相同的 `cb(null)` 路径（查询失败 ≠ 空表，5s 后重试）。
- **改动代码片段**：
```js
// modified: [v31-P1] SDK 同步抛错会打断整条启动链，后续 6 张表将永不再拉取
let pr; // modified
try { pr = db.query({ databaseId: dbId, pageSize: PAGE, startCursor: cursor }); } // modified
catch (err) { cb(null); return; } // modified
Promise.resolve(pr).then(function (r) { // modified
```

### F07 【代码位置】18 处模板插值（R02-R18）
- **原始问题**：`value="${k.due||''}"`、`data-d="${x.date}"`、`${human(k.due)}`、`${d.slice(5)}` 等把云端/用户可控值直接拼进属性或 HTML 文本。
- **修改思路**：出口统一 `esc()`；`human()` / `relDatePill()` 返回值一律经 `esc()` 后再入 HTML（两者返回纯文本 / 固定模板，转义不会破坏结构）。
- **改动代码片段**（代表性 3 处，其余同构）：
```js
return `<span class="pill gray">${esc(String(d).slice(5).replace('-', '/'))}</span>`; // modified
<input type="date" id="pe_due" value="${esc(k.due||'')}"></div> <!-- // modified -->
<button class="btn btn-sm" data-act="stu-load" data-d="${esc(x.date)}">${T('stuLoad')}</button> <!-- // modified -->
```
> 说明：改动位于 HTML 模板字面量内部，无法写 JS 行注释，故用 `<!-- // modified -->`（HTML 注释，不可见、不影响布局）等价标注，仍可全文检索 `// modified`。

### F08 【代码位置】7649-7652 `rowOf()`
- **原始问题**：云端「本地ID」列原样作为 `x.id`，最终进入 54 处 `data-id` 属性。
- **修改思路**：在同步入口做白名单清洗（只保留 `[A-Za-z0-9_-]`，截 64 位）；`remoteId` 仍保留原始 `recId` 供 SDK 调用，不影响云端读写。
- **改动代码片段**：
```js
// modified: [v31-P1] 云端「本地ID」列可被手工编辑，直接写进 data-id 等属性会造成属性注入；入口白名单清洗
var lid = String(gv(row, '本地ID') || '').replace(/[^A-Za-z0-9_\-]/g, '').slice(0, 64); // modified
var rid = recId(row); // modified
var x = cfg.rowToItem(row, gv);
x.id = lid || ('leg_' + String(rid).replace(/[^A-Za-z0-9_\-]/g, '')); // modified
x._lid = lid;
```

### F09 【代码位置】7771 `merge()`
- **原始问题**：云端合并后无条件 `render()`，整页替换 `#view.innerHTML`，正在输入会失焦丢字。
- **修改思路**：数据照常合并并 `save()`；仅当 `visible()`（无弹层、无聚焦输入、页面可见）时立即重绘，否则 900ms 后空闲补绘。数据不依赖重绘才存在。
- **改动代码片段**：
```js
if (before !== after) { // modified
  save(); // modified
  // modified: [v31-P1] 正在输入/弹层打开时不整页重绘（会失焦丢字），数据已合并，空闲后补一次重绘
  if (visible()) render(); // modified
  else { clearTimeout(deferTimer); deferTimer = setTimeout(function () { try { if (visible()) render(); } catch (_) {} }, 900); } // modified
}
```

### F10 【代码位置】3152 `uid()`
- **原始问题**：同毫秒批量创建时随机段仅 5 位，存在碰撞可能。
- **修改思路**：优先 `crypto.randomUUID()`（去掉连字符，保证可作 DOM 属性值），旧环境/非安全上下文自动降级并把随机段加倍。
- **改动代码片段**：
```js
// modified: [v31-P2] 同毫秒批量创建改用 crypto.randomUUID（去掉连字符，保持 id 可作属性值）；旧环境自动降级
const uid = () => {
  try {
    if (window.crypto && typeof window.crypto.randomUUID === 'function') return 'i' + window.crypto.randomUUID().replace(/-/g, ''); // modified
  } catch (e) {}
  return Date.now().toString(36) + Math.random().toString(36).slice(2, 7) + Math.random().toString(36).slice(2, 7); // modified
}; // modified
```

### F11 / F12 / F13 【代码位置】7661 / 7855 / 3605
```js
p['更新时间'] = { number: Number(x.updatedAt) || Date.now() }; // modified
finished = true; // modified: [v31-P2] 异常分支封口，迟到的异步回调不得再改写 t.sig 吞掉待推送改动
// save() catch 内：
const quota = e && (e.name === 'QuotaExceededError' || e.code === 22); // modified
toast(quota ? (LANG === 'en' ? 'Storage full: export a backup first; data is still in memory' : '本机存储已满：请先导出备份再清理，数据暂未丢失') : T('tSaveFail'), true); // modified
```

### F14 / N01 【代码位置】3613-3620（新增 `storage` 监听）
- **原始问题**：只有 `visibilitychange`，切回标签才发现另一标签已写入；`load()` 后同步内核 `t.sig` 未认账，会把别人的改动误判成本地改动全表回推。
- **修改思路**：新增 `StorageEvent` 监听（400ms 去抖；后台标签 / 弹层 / 输入中不打断；用 `_savedAt` 判新旧），载入后调 `window.__SYNC_RESIG()` 重新认账；`visibilitychange` 分支同样补认账。`CloudSync` 新增 `resig()` 方法并暴露到 window（幂等、无副作用）。
```js
// modified: [v31-N1] StorageEvent：另一标签页写入后即时同步，不必等切回本标签；输入/弹层期间不打断
let __crossTabTimer = 0; // modified
window.addEventListener('storage', e => { // modified
  if (!e || e.key !== KEY || !S || e.newValue == null || document.hidden) return; // modified
  if (document.querySelector('.mask')) return; // modified
  const ae = document.activeElement; // modified
  if (ae && /^(INPUT|TEXTAREA|SELECT)$/.test(ae.tagName || '')) return; // modified
  try {
    const savedAt = Number(JSON.parse(e.newValue)._savedAt) || 0; // modified
    if (savedAt <= (S._savedAt || 0) + 800) return; // modified
  } catch (_) { return; } // modified
  clearTimeout(__crossTabTimer); // modified
  __crossTabTimer = setTimeout(() => { // modified
    try { load(); window.__SYNC_RESIG && window.__SYNC_RESIG(); render(); } catch (_) {} // modified
  }, 400); // modified
}); // modified
```
```js
resig: function () { T.forEach(function (t) { t.sig = safeSnap(t.cfg); }); } // modified
if (CloudSync) { try { window.__SYNC_RESIG = CloudSync.resig; } catch (e) {} } // modified
```

### N02 【代码位置】5840 附近 / 5930-5955 / 6423-6425
- **需求**：F02 留档之后必须给用户一条可见的自救通道，否则备份仍等于「看不见」。
- **修改思路**：新增 `corruptBackup()` 读取 `..._corrupt_latest`；`vBackup()` 生成一张提示卡并拼到返回 HTML；`act()` 新增 `bk-corrupt-dl` 分支调用既有 `downloadText()` 下载原始串。无隔离备份时卡片不渲染，页面外观零变化。
```js
// modified: [v31-N2] 读取最近一份隔离备份，供「数据说明」页下载自救
function corruptBackup() {
  try {
    const raw = localStorage.getItem(KEY + '_corrupt_latest'); // modified
    if (!raw) return null; // modified
    const o = JSON.parse(raw); // modified
    return (o && typeof o.raw === 'string' && o.raw) ? o : null; // modified
  } catch (e) { return null; } // modified
}
// vBackup() 末尾：
      <div class="card-b">${diagHtml}</div></div>`
  + cbHtml; // modified
// act() 新增：
    case 'bk-corrupt-dl': { // modified: [v31-N2] 下载隔离的原始损坏数据
      const cb = corruptBackup(); // modified
      if (!cb) { toast(T('bkImportFail'), true); return; } // modified
      downloadText('涨潮归_损坏数据备份_' + new Date(cb.at || Date.now()).toISOString().slice(0, 10) + '.json', cb.raw, 'application/json;charset=utf-8'); // modified
      return; // modified
    }
```

---

## 6. 完整整合后的全部源代码

**交付文件：`个人办公工作台_v31_审计修复版.html`**（8328 行，单文件，可直接用浏览器打开或整份替换线上页面源码）。

- 全部 37 处改动 / 116 个标记均可在文件中全文检索 `// modified` 定位；模板字面量内部的行使用 `<!-- // modified -->`。
- 每行标记同时带 `[v31-P0/P1/P2/N1/N2]` 分组与简短用途说明。
- 未改动的部分逐字保留原样（与基线逐行对比：**98.5% 行未变化**，删除/替换 33 行、新增 127 行）。
- 附 `个人办公工作台_v31_变更对照.diff`（30 个 hunk / 349 行）供逐行人工复核。
- 校验结果：`node --check` **通过**（提取主脚本 6416 行后语法无错误）；回归测试见第 8 节。

> 8328 行源码不适合粘贴进对话，故以文件交付；该文件即本节要求的「完整整合后的全部源代码」。

---

## 7. 风险提示

| # | 风险 / 兼容性影响 | 说明与缓解 |
|---|---|---|
| 1 | **本地存档格式兼容** | `KEY`、`blank()` 结构、7 张数组、9 列契约、7 个 `dbId` 全部未变；旧存档可直接读取，云端无需改表。零破坏性修改。 |
| 2 | **新增 localStorage 键** | 新增 `pd_workbench_v2_corrupt_latest` + 最多 3 份 `..._corrupt_<ts>`。极端情况下（存储已满）隔离备份本身会写失败，已用 try/catch 静默降级，不影响主流程。 |
| 3 | **`esc()` 覆盖范围变化** | 历史数据中若真有人把 HTML 片段（如 `<b>加粗</b>`）存进标题/日期，今后会按纯文本显示而非渲染。这是预期的安全行为，但视觉可能与之前不同。 |
| 4 | **`crypto.randomUUID` 可用性** | 需安全上下文（https / localhost）；以 `file://` 打开时自动降级到原算法并把随机段加倍，功能不受影响。 |
| 5 | **云端 `本地ID` 被清洗** | 若云端该列含非法字符，清洗后可能与本地 `id` 不匹配，导致该行被当作新记录插入一次（异常数据场景，正常数据不受影响）。建议不要手工编辑云端「本地ID」列。 |
| 6 | **延后重绘** | 用户持续输入且云端恰好合并数据时，新数据可能要到下次交互才显示（**数据已合并、不会丢**，仅显示延迟）。 |
| 7 | **`StorageEvent` 行为** | 只在**其他**标签页写入时触发，本标签不会自激；后台标签仍走原有 `visibilitychange`。与既有 `beforeunload` / `storageNewerThanMine` 护栏不冲突。 |
| 8 | **平台质量工具仍报 PQ001 / PQ005** | `PQ001`（括号不配平）经 `node --check` 证实为误报——脚本含大量模板字符串与 SVG 字符串中的 `}`，工具按字符计数会算错。`PQ005` 是「innerHTML + 动态插值」的粗粒度架构告警：本项目采用模板字符串渲染，要彻底消除需 DOM API 全量重写（架构级重构，违背稳定性优先）。本次已用出口 `esc()` + 入口白名单净化收敛真实注入面，上传时建议带 `--ack` 声明。 |
| 9 | **未处理项** | A17（空业务键互认副本）、A18（自检只查首页）、日程/学习日期格式兜底：均因「改动会与去重/业务键逻辑耦合、可能造成重复插入或误删」而**刻意不改**，详见 4.4。 |
| 10 | **未回写线上页面** | 本报告与修复版为**离线交付**，线上工作台（version 40）当前仍是 v30 代码。按你「先复核再落地」的习惯，等你确认后我再提交新版本；提交前请务必先导出一份 JSON 备份。 |

---

## 8. 验证方案

### 8.0 准备（务必先做）

1. 打开线上工作台 → 「数据说明」→ **导出备份**，保存 JSON 到本地。
2. 用浏览器打开本地 `个人办公工作台_v31_审计修复版.html`。以 `file://` 打开时 `__SMART_PAGE__` 不存在，`CloudSync` 会返回 `null`，页面降级为**纯本地模式**，本地逻辑全部可测；云端相关项需把文件替换到线上页面后验证。
3. 打开 DevTools Console，用下面的脚本造数据 / 观察结果。

### 8.1 已完成的自动化验证（本次交付前已跑）

| 验证 | 命令 / 方法 | 结果 |
|---|---|---|
| 主脚本语法 | 提取主 script → `node --check main.fixed.js` | **通过**（基线也通过，PQ001 为误报） |
| 致命 bug 回归 | Node `vm` 隔离执行 `blank()/normalize()/reseqIns()`，构造含 1 条学习打卡的 `S` | 基线 `THROW -> TypeError: keep.get is not a function`；修复版 `PASS -> {"study":1,"tasks":1,"taskPrio":4,"personalPrio":4}`（数据保留 + 非法优先级兜底） |
| 改动范围 | 逐行 diff | 98.5% 行未变化；30 hunk；无标记落在标签内部 |
| 转义覆盖 | 扫描模板插值 | 用户可控的 `value=` / `data-d=` / `human()` / `relDatePill()` 全部经 `esc()`；剩余 26 处 `value=` 与 54 处 `data-*` 均为内部常量枚举 |

### 8.2 手工验证步骤（建议按顺序）

| # | 步骤 | 预期结果 |
|---|---|---|
| 1 | **致命 bug**：Console 执行 `localStorage.setItem('pd_workbench_v2', JSON.stringify({v:1,study:[{id:'s1',date:'2026-11-04',mins:30,updatedAt:Date.now()}],tasks:[{id:'t1',title:'测试',prio:2}]}))` 后刷新 | 页面正常打开、不白屏、不弹「读取失败」；Console 里 `S.study.length === 1` |
| 2 | **脏数据**：把上例的 `tasks` 改成 `[null, {id:'t2',title:'X',prio:99}]`，刷新 | 不白屏；`S.tasks.length === 1` 且 `S.tasks[0].prio === 4` |
| 3 | **损坏隔离**：Console 执行 `localStorage.setItem('pd_workbench_v2','{坏掉的JSON')` 后刷新 | 弹「读取失败」提示；`localStorage.getItem('pd_workbench_v2_corrupt_latest')` 有值且含原始串；进入「数据说明」页出现「检测到损坏数据备份」卡片，点「下载损坏备份」能下载 JSON |
| 4 | **恢复闭环**：把第 0 步导出的 JSON 通过「数据说明 → 导入恢复」导回 | 数据完整恢复，不白屏；刷新后仍在 |
| 5 | **跨标签同步**：用同一浏览器开两个标签（同一 origin），A 加一条任务，保持 B 在前台且未聚焦输入框 | 约 400ms 内 B 自动出现新任务，无需切标签；Console 无异常 |
| 6 | **输入不丢字**：在「今天待办」标题框持续输入，同时 Console 执行 `CloudSync && CloudSync.pullAll()`（线上环境） | 光标不丢、已输入文字不消失；空闲后列表刷新 |
| 7 | **XSS**：把某条任务标题改为 `<img src=x onerror=alert(1)>` 并保存，刷新 | 不弹窗；标题按纯文本原样显示；查看元素可见 `&lt;img ...` |
| 8 | **属性注入**：把某条学习打卡的日期字段在云端表手工改成 `2026-11-04" onmouseover="alert(1)`，触发一次拉取 | 不弹窗；`data-d` 属性被转义，页面结构不被破坏 |
| 9 | **清空彻底性**（线上）：先记一条学习打卡 → 「数据说明 → 清空全部」→ 等一轮同步 → 刷新 | 学习打卡不复活（墓碑已登记 `tombStudy`） |
| 10 | **UID 唯一性**：Console 执行 `const a=new Set(); for(let i=0;i<2000;i++)a.add(uid()); console.log(a.size)` | 输出 `2000` |
| 11 | **写入失败提示**：Console 临时把 `Storage.prototype.setItem` 改成抛 `QuotaExceededError`，再改一条任务 | 提示「本机存储已满：请先导出备份再清理，数据暂未丢失」，内存数据未被重置 |
| 12 | **回归冒烟**：逐页点开 12 个视图（焦点/待办/速记/专项/台账/日程/个人事项/学习/学习数据/复盘/搜索/数据说明），各做一次增改删 + 导出 Excel | 无 Console 报错、无白屏、导出文件可打开 |

### 8.3 上线前检查清单

- [ ] 已导出 JSON 备份并另存一份
- [ ] 本地 `file://` 模式跑完 8.2 的第 1-4、7、10、11、12 项
- [ ] 确认第 7 节风险 3 / 5 可接受
- [ ] 确认后再把 `个人办公工作台_v31_审计修复版.html` 整份提交为页面新版本（提交时质量工具 PQ001/PQ005 需按第 7 节第 8 条带 `--ack` 声明）
- [ ] 线上再跑第 5、6、8、9 项（涉及云端 SDK 的验证）
