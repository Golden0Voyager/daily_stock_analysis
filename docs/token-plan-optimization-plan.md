# Token Plan 请求优化方案

> 目标：针对 SenseNova Token Plan（按请求次数计费/限次），将每轮分析的 LLM 请求数从 ~13-25 次降至 1-3 次。

---

## 1. 现状分析

### 1.1 当前请求结构（12 只自选股）

| 步骤 | 请求数 | 说明 |
|------|--------|------|
| 个股分析 × 12 | 12-24 次 | 每只股票 1 次主请求 + 最多 1 次完整性重试 |
| 大盘复盘 | 1 次 | `main.py --market-review` 时触发 |
| **合计** | **13-25 次/轮** | |

### 1.2 现有"批量"代码的局限

`src/analyzer.py:3411` 的 `batch_analyze()` 只是**顺序循环** + 固定延迟，并未减少请求数：

```python
def batch_analyze(self, contexts, delay_between=2.0):
    for i, context in enumerate(contexts):
        if i > 0:
            time.sleep(delay_between)  # ❌ 只是加延迟，不是合并请求
        result = self.analyze(context)  # ❌ 每只股独立调用 LLM
```

### 1.3 Token Plan 的约束

- **计费维度**：按请求次数（非 Token 数）
- **配额重置**：每 5 小时
- **模型选择**：`sensenova-6.7-flash-lite` / `deepseek-v4-flash`
- **上下文窗口**：128K+（足够容纳 12 只股的完整数据）

**核心洞察**：Token Plan 下，Prompt 多长都"免费"，关键在于**减少请求次数**。

---

## 2. 优化方案总览

```
优化前：12 只股 × (1 主请求 + 0~1 重试) + 1 大盘 = 13~25 请求

优化后：1 次批量请求（12 只股）+ 0~1 次完整性重试 = 1~2 请求
        或：2 次批量请求（6+6）+ 0~1 次重试 = 2~3 请求

预期降幅：85%~90%
```

### 2.1 方案矩阵

| 策略 | 预期节省 | 实现复杂度 | 风险 |
|------|----------|-----------|------|
| **A. 真·批量 LLM 分析** | ~85% | 中 | JSON 解析失败影响整批 |
| **B. 关闭完整性重试** | ~15% | 低 | 用占位符补全缺失字段 |
| **C. 取消请求间隔** | ~20% 时间 | 低 | 可能触发限流 |
| **D. 数据未变跳过** | ~30%（长期） | 中 | 需缓存设计 |

**推荐组合：A + B + C**（一次性实施，互不冲突）

---

## 3. 详细方案

### 3.1 策略 A：真·批量 LLM 分析（核心）

#### 3.1.1 设计思路

将多只股票的**所有数据**打包进一个 Prompt，让 LLM 一次性返回一个**股票分析数组**。

```
Prompt 结构（12 只股）：

[系统提示词：决策仪表盘 JSON 格式说明]

---

# 批量分析请求（共 12 只股票）

请对以下每只股票独立分析，返回一个 JSON 数组，数组中每个元素对应一只股票的分析结果。

## 股票 1：603893 瑞芯微
[技术面数据...]
[筹码分布...]
[趋势分析...]
[新闻...]

## 股票 2：002241 歌尔股份
[技术面数据...]
...

## 股票 N：...

输出格式：
[
  {"stock_name": "瑞芯微", "code": "603893", "sentiment_score": 75, ...},
  {"stock_name": "歌尔股份", "code": "002241", "sentiment_score": 62, ...},
  ...
]
```

#### 3.1.2 分批策略（容错考虑）

为降低"一损俱损"风险，采用**中等批次**：

```python
BATCH_SIZE = 6  # 每批 6 只股，12 只股 = 2 次请求
# 或 BATCH_SIZE = 12  # 激进模式：一次性全部（更省请求）
```

| 批次大小 | 12 只股请求数 | 失败影响 |
|---------|--------------|---------|
| 12（全部）| 1 次 | 全部失败需 fallback |
| 6（推荐） | 2 次 | 仅影响半批 |
| 4（保守） | 3 次 | 仅影响 1/3 |

#### 3.1.3 新增方法签名

```python
class GeminiAnalyzer:
    def analyze_batch(
        self,
        contexts: List[Dict[str, Any]],
        news_contexts: Optional[Dict[str, str]] = None,  # code -> news
        *,
        batch_size: int = 6,
        progress_callback: Optional[Callable[[int, str], None]] = None,
    ) -> List[AnalysisResult]:
        """真·批量分析：将多只股票合并为单次 LLM 请求。"""

    def _format_batch_prompt(
        self,
        contexts: List[Dict[str, Any]],
        news_contexts: Optional[Dict[str, str]],
        report_language: str,
    ) -> str:
        """格式化批量 Prompt（每只股的独立 section）。"""

    def _parse_batch_response(
        self,
        response_text: str,
        contexts: List[Dict[str, Any]],
    ) -> List[AnalysisResult]:
        """解析 LLM 返回的 JSON 数组，映射到各只股票。"""
```

#### 3.1.4 Prompt 设计要点

```markdown
# 批量股票分析请求

你是一个专业的股票分析师。请对以下 {N} 只股票逐一进行独立分析，
输出一个严格的 JSON 数组，数组长度为 {N}。

## 重要规则
1. 每只股票的分析相互独立，不要互相影响判断
2. 必须返回完整的 JSON 数组，不能省略任何一只股票
3. 数组中元素的顺序必须与股票输入顺序一致

## 股票列表

### [1/12] 603893 瑞芯微
[数据表格...]

### [2/12] 002241 歌尔股份
[数据表格...]

...

## 输出格式
返回一个 JSON 数组：
```json
[
  {"stock_name": "...", "code": "603893", "sentiment_score": 75, ...},
  {"stock_name": "...", "code": "002241", "sentiment_score": 62, ...}
]
```
```

#### 3.1.5 错误处理策略

```python
def analyze_batch(...):
    all_results = []
    for chunk in chunks(contexts, batch_size):
        try:
            # 1. 尝试批量分析
            results = self._analyze_batch_chunk(chunk)
            all_results.extend(results)
        except Exception:
            # 2. 批量失败 → 降级为单只分析
            logger.warning("批量分析失败，降级为单只分析")
            for ctx in chunk:
                result = self.analyze(ctx)  # 回退到现有方法
                all_results.append(result)
    return all_results
```

---

### 3.2 策略 B：关闭完整性重试

当前配置：
- `REPORT_INTEGRITY_RETRY=1`（默认）→ 每只股最多 2 次请求
- 优化后：`REPORT_INTEGRITY_RETRY=0` → 每只股仅 1 次请求

已有兜底机制：
- `apply_placeholder_fill()` 会在字段缺失时自动填充占位符
- 报告仍然可用，只是部分内容可能为"数据暂缺"

**环境变量调整**：
```bash
# .env 或 GitHub Actions Secrets
REPORT_INTEGRITY_RETRY=0
```

---

### 3.3 策略 C：取消请求间隔

当前配置：
- `GEMINI_REQUEST_DELAY=2.0`（默认）→ 每只股分析前等待 2 秒
- 12 只股 = 额外 22 秒等待时间

优化后：
```bash
# 批量模式下不再需要间隔（单次请求内部自然串行）
GEMINI_REQUEST_DELAY=0
```

---

### 3.4 策略 D：数据未变跳过（可选进阶）

长期优化：如果某只股票的收盘价、成交量、新闻等关键数据与上次分析时完全一致，则跳过 LLM 调用，直接复用上次的 `AnalysisResult`。

实现思路：
1. 生成数据指纹：`hash(code + close + volume + news_digest)`
2. 对比上次指纹，一致则跳过
3. 适用于：持仓股日常波动不大的场景

**预期效果**：
- 大跌/大涨日：12 只股都变 → 仍需 1-2 次请求
- 平淡日：仅 3-5 只股变化 → 只需 1 次请求

---

## 4. Pipeline 集成方案

### 4.1 修改 `StockAnalysisPipeline.run()`

```python
def run(self, stock_codes=None, ...):
    # ... 现有数据获取逻辑 ...

    # 新增：判断使用批量模式还是传统模式
    use_batch_llm = getattr(self.config, 'llm_batch_mode', False)

    if use_batch_llm:
        # 批量模式：先收集所有 context，再一次性分析
        contexts = []
        for code in stock_codes:
            context = self._build_analysis_context(code)
            contexts.append(context)

        # 一次性 LLM 调用（或 2 次，取决于 batch_size）
        results = analyzer.analyze_batch(
            contexts,
            batch_size=getattr(self.config, 'llm_batch_size', 6),
        )
    else:
        # 传统模式：线程池并行（保持现有逻辑）
        with ThreadPoolExecutor(...) as executor:
            ...
```

### 4.2 配置项设计

```python
# src/config.py 新增
llm_batch_mode: bool = False          # 是否启用批量分析
llm_batch_size: int = 6               # 每批股票数
llm_batch_fallback: bool = True       # 批量失败时是否降级为单只分析
```

```bash
# .env.example 新增
# Token Plan 优化配置
# LLM_BATCH_MODE=true           # 启用批量分析（大幅减少请求数）
# LLM_BATCH_SIZE=6              # 每批分析股票数（12只股=2次请求）
# LLM_BATCH_FALLBACK=true       # 批量失败时自动降级为单只分析
# REPORT_INTEGRITY_RETRY=0      # 关闭完整性重试（减少额外请求）
# GEMINI_REQUEST_DELAY=0        # 取消请求间隔（批量模式下不需要）
```

---

## 5. 实施步骤

### Phase 1：低风险配置优化（立即生效，0 代码改动）

1. **修改 `.github/workflows/daily_analysis_my.yml`**：
   ```yaml
   env:
     # ... 现有配置 ...
     REPORT_INTEGRITY_RETRY: 0      # 关闭重试
     GEMINI_REQUEST_DELAY: 0        # 取消间隔
   ```

2. **预期效果**：请求数从 ~24 次降至 ~12 次（降幅 50%）

### Phase 2：批量分析实现（主要优化）

1. 在 `src/analyzer.py` 新增 `analyze_batch()` / `_format_batch_prompt()` / `_parse_batch_response()`
2. 在 `src/core/pipeline.py` 集成批量调用路径
3. 在 `src/config.py` 新增批量配置项
4. 更新 `.env.example` 和文档

**预期效果**：请求数从 ~12 次降至 1-2 次（额外降幅 85%）

### Phase 3：数据缓存（长期优化）

1. 实现数据指纹比对
2. 缓存 `AnalysisResult` 到本地 SQLite
3. 未变化股票直接复用

---

## 6. 风险评估

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| 批量 JSON 解析失败 | 中 | 整批失败 | 自动降级为单只分析 |
| Prompt 超长超限 | 低 | 请求被拒 | 动态调小 batch_size |
| LLM 输出不完整（漏股票） | 中 | 缺失结果 | 对比预期数量，缺失的单独补分析 |
| 多股数据互相干扰 | 低 | 分析质量下降 | Prompt 明确"独立分析"指令 |
| Token Plan 限流（频率） | 低 | 请求被拒 | 保留 1 秒间隔作为保险 |

---

## 7. 预期收益

| 指标 | 优化前 | 优化后（Phase 1+2） | 降幅 |
|------|--------|---------------------|------|
| 每轮请求数 | 13-25 次 | 1-3 次 | **~90%** |
| 每轮分析时间 | ~5-10 分钟 | ~2-4 分钟 | **~60%** |
| 配额消耗（5小时）| 2-4 轮 | 10-20 轮 | **5~10x** |

---

## 8. 推荐实施顺序

```
立即执行（今天）：
  └── Phase 1：修改 workflow 环境变量（REPORT_INTEGRITY_RETRY=0, GEMINI_REQUEST_DELAY=0）

近期执行（本周）：
  └── Phase 2：实现 analyze_batch() 方法 + Pipeline 集成

远期执行（按需）：
  └── Phase 3：数据指纹缓存
```

---

## 附录：批量 Prompt 示例（2 只股）

```markdown
你是一个专业的 A 股投资分析师。请对以下 2 只股票逐一进行独立分析，
输出一个严格的 JSON 数组（长度为 2）。

## 股票 1/2：603893 瑞芯微

### 基础信息
| 项目 | 数据 |
|------|------|
| 收盘价 | 185.20 元 |
| 涨跌幅 | +3.52% |
| MA5 | 182.50 |
| MA10 | 179.30 |
| MA20 | 175.80 |
| 量比 | 1.85 |
| 换手率 | 4.2% |

### 趋势分析
均线多头排列，乖离率 1.5%，安全范围。

---

## 股票 2/2：002241 歌尔股份

### 基础信息
| 项目 | 数据 |
|------|------|
| 收盘价 | 28.50 元 |
| 涨跌幅 | -1.20% |
| MA5 | 29.10 |
| MA10 | 29.50 |
| MA20 | 30.20 |
| 量比 | 0.85 |
| 换手率 | 2.1% |

### 趋势分析
均线空头排列，乖离率 -2.1%，弱势。

---

## 输出格式

请严格按照以下 JSON 数组格式输出，数组长度必须为 2：

```json
[
  {
    "stock_name": "瑞芯微",
    "code": "603893",
    "sentiment_score": 75,
    "trend_prediction": "看多",
    "operation_advice": "持有",
    "decision_type": "hold",
    "confidence_level": "中",
    "dashboard": { ... },
    "analysis_summary": "...",
    "risk_warning": "..."
  },
  {
    "stock_name": "歌尔股份",
    "code": "002241",
    "sentiment_score": 35,
    "trend_prediction": "看空",
    "operation_advice": "观望",
    "decision_type": "hold",
    "confidence_level": "低",
    "dashboard": { ... },
    "analysis_summary": "...",
    "risk_warning": "..."
  }
]
```
```
