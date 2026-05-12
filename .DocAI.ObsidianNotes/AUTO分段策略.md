AUTO 策略在 `segmentByAuto()` 方法（`DocumentSegmenter.java:114`）中的判断逻辑很简单，两步决策：

**第一步：检测是否有章节结构**
```java
boolean hasChapterMarkers = detectChapterMarkers(content);
```
扫描文档是否包含 `第1章`、`第1节`、`Chapter`、`Section` 等关键词。有 → 用 **CHAPTER（按章节分段）**。

**第二步：按长度判断**
无章节标记时根据文档长度：
- 内容长度 `> 2000` 字符 → 用 **SEMANTIC（语义分段）**
- 否则 → 用 **FIXED_LENGTH（固定长度分段，chunkSize=500, overlap=50）**

所以 AUTO 的决策树是：
```
文档是否有章节标记？
  ├─ 是 → CHAPTER（按章节）
  └─ 否 → 文档长度 > 2000？
       ├─ 是 → SEMANTIC（语义）
       └─ 否 → FIXED_LENGTH（固定500字）
```