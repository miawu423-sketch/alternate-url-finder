# 可替代URL源码提取 - Case 记录

工具地址：https://miawu423-sketch.github.io/alternate-url-finder/
仓库：https://github.com/miawu423-sketch/alternate-url-finder

用途：记录使用过程中发现的问题、边界情况、修复历史，方便定期 review 找到系统性优化点。

---

## 已修复 Case

### 1. WordPress 站点的技术接口链接（RSS/API 等）
- 触发站点：peaksubstation.com
- 问题：提取到 `/feed/`、`/wp-json/`、`/oembed/`、`/comments/feed/` 等一堆机器接口
- 修复：v2 加入 noise filter，按路径/文件后缀过滤
- 修复日期：2026-05-26

### 2. iCal 日历订阅链接
- 触发站点：msviewsandnews.org
- 问题：提取到 `?ical=1` 结尾的日历订阅URL
- 修复：增加 `?ical=`、`?outlook-ical=`、`.ics`、`/ical/`、`/calendar/export` 过滤
- 修复日期：2026-05-26

### 3. JSON-LD 中的站点/组织级 url 字段
- 触发站点：rocketreach.co、sanfrancisco10.com
- 问题：`@type: Organization/WebSite` 的 url 指向首页被误抓
- 修复：JSON-LD 只提取内容类型（Article/BlogPosting/NewsArticle 等）的 url
- 修复日期：2026-05-26

### 4. 底部「全部复制」「全部打开」按钮无响应
- 问题：innerHTML 内联 onclick 转义层数过多导致代码断裂
- 修复：改用 `createElement + .onclick=` 绑定
- 修复日期：2026-05-26

### 5. 单个URL缺少复制按钮
- 需求：每行URL除了「打开」外，也要能单独复制
- 修复：每行加「复制」按钮（点击后显示 ✓ 反馈）
- 修复日期：2026-05-26

### 6. 无协议 URL 显示为死链
- 触发站点：sohu.com（og:url = `www.sohu.com/...`）
- 问题：搜狐的 og:url 没写 `https://`，导致展示的是不完整 URL
- 修复：`fixUrl` 自动补全协议（`//xxx` / `www.xxx` / `m.xxx` / `sub.xxx.com`）
- 修复日期：2026-05-26

### 7. 相对路径被错误拼接
- 触发站点：sohu.com（alternate href = `m.sohu.com/...`）
- 问题：浏览器把无协议的路径当相对路径解析成 `https://www.sohu.com/a/m.sohu.com/a/...`
- 修复：alternate 改用 `getAttribute('href')` 读原始值再 fixUrl；加 `isGarbledRelative` 检测（路径中出现域名格式的过滤掉）
- 修复日期：2026-05-26

### 8. URL 编码的 og:url
- 触发站点：cj.sina.com.cn（og:url = `http%3A%2F%2Fcj.sina.com.cn%2F...`）
- 问题：新浪 og:url 用了 URL 编码，展示为乱码
- 修复：`fixUrl` 检测到编码内容自动 `decodeURIComponent`
- 修复日期：2026-05-26

### 9. 裸域名首页链接被误抓
- 触发站点：CCTV 央视新闻（content-static.cctvnews.cctv.com）、cubbi.com.au
- 问题：og:url 或 JSON-LD 里指向了根域名（如 `https://content-static.cctvnews.cctv.com`），路径为空/`/`
- 修复：`isHomepage` 检查提升到 `addResult` 全局过滤（所有信号来源统一过滤裸域名）
- 修复日期：2026-07-03

### 10. SPA 站点 URL 构造函数被污染
- 触发站点：cubbi.com.au（SPA）
- 问题：`isHomepage` 用 `new URL()` 解析，某些站点覆盖了全局 URL 对象导致解析失败，过滤失效
- 修复：改用纯字符串匹配（去掉协议头后，看第一个 `/` 是否在末尾）
- 修复日期：2026-07-17

---

## 已知限制（不修复，属于工具能力边界）

### A. 跨站/跨子域内容分发无法自动发现
- 典型案例：新浪同集团分发
  - `cj.sina.com.cn/articles/view/7879776328/xxx`（财经）
  - `ent.sina.cn/2026-02-17/detail-xxx.html`（娱乐移动版）
  - `k.sina.com.cn/article_7879776328_xxx.html`（看点）
- 原因：源码里根本没写互相引用，工具无法发现
- 建议做法：走标题/正文关键句搜索（本工具不覆盖）

### B. 源码里的替代URL本身是坏数据
- 典型案例：timeshighereducation.com
  - 昆士兰大学排名页 shortlink 指向 `/node/9521`
  - 但 `/node/9521` 打开是完全无关的另一篇文章
- 原因：网站自己 CMS 数据错误（可能是数据迁移遗留）
- 建议做法：使用人员在验收环节人工确认内容一致性

---

## 定期 Review 观察点

每积累 10+ 新 case 时，review 一下是否有共性：

1. **过滤规则是否需要扩展？** 是否有新的噪音路径模式出现？
2. **fixUrl 是否需要处理更多变体？** 有没有其他编码方式或前缀省略？
3. **isHomepage 判定是否足够精准？** 有没有"看似路径但实际是首页跳转"的场景？
4. **是否需要给低可信度信号加视觉提示？** 例如 shortlink、正文提取的 URL 可能不准
5. **是否要支持批量处理？** 如果使用量大，考虑做一个批量输入URL列表的版本

---

## 未来可能的扩展方向（等积累到一定 case 数量再决策）

- [ ] 批量处理：输入URL列表，自动抓取每个页面的替代URL
- [ ] 可信度分层：不同信号来源标记不同的可信度等级
- [ ] 内容一致性校验：调用 API 检查替代URL的实际内容是否匹配
- [ ] 浏览器插件版本：避免每次改动都要重新拖拽安装
- [ ] 导出功能：把结果导出为 CSV / JSON 供标注工具使用
