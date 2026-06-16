# 02 · render prompt 拼装规则

> 每页 render prompt 必须严格按以下公式拼装 · 没有其他步骤

---

## 拼装公式

```
render_prompt
  = [公司视觉规范精简版]      （380 字符 · 来自 01_style_overlay/公司视觉规范_精简版.md）
  + "\n\n本页主题："          + 源文 H1
  + "\n\n以下完整内容须 100% 呈现在图中（一字不删 · 不替换 · 不截断）：\n\n"
  + [源文完整正文]            （来自 00_source/ · 一字不动）
```

## 长度公式

```
render_prompt_length = 380 + len(源文 H1) + ~80（固定连接词）+ len(源文正文)
```

**校验**：实际 render_prompt 字符数必须 ≈ 上述公式结果。如果偏差 > 5%，说明发生了未授权的 transform。

---

## 明确禁止的步骤

| 禁止行为 | 反例 | 后果 |
|---|---|---|
| 提取"H2/H3/列表项"作为"核心内容" | `extract_core_content()` | 默默删除段落级文字 |
| 设字符上限做截断 | `max_chars=3800` | 高密度页被砍掉一半 |
| 摘要化处理 | "用一句话概括" | 数字 / 术语丢失 |
| 术语替换不上报 | `RAISE → 可问责AI` 静默替换 | 用户不知道发生了什么 |
| 删除"看起来像噪声"的行 | 删 markdown 注释 / 引用块 | 可能误删正文 |
| 改写句式 | 把口语化的金句改"工程化" | 改变了用户原意 |

---

## 允许的步骤（必须显式记录到 diff_report.json）

### 允许 1：移除明显的工程包装层
- 移除 `<!-- ====== -->` 注释块
- 移除"📍 框架定位"等元数据块
- **必须在 diff_report 中记录 `removed_wrapper_blocks: [...]`**

### 允许 2：替换禁词
- 仅当本项目有禁词清单时
- **必须在 diff_report 中记录 `term_replacements: [{from: ..., to: ...}]`**

### 不允许的"灰色地带"
以下行为即使看起来"无害"也不允许：
- 合并相邻的空行
- 调整 markdown 标题层级
- 重排列表顺序
- 把表格转换为列表

---

## Python 参考实现（伪代码）

```python
def build_render_prompt(source_md_path, style_overlay_path):
    source = read_text(source_md_path)
    style = read_text(style_overlay_path)  # 380 字符精简版

    # 仅提取 H1 · 不动正文
    h1 = extract_first_h1(source)

    # 拼装 · 无 transform
    prompt = (
        style
        + "\n\n本页主题：" + h1
        + "\n\n以下完整内容须 100% 呈现在图中"
        + "（一字不删 · 不替换 · 不截断）：\n\n"
        + source  # 完整源文 · 不做任何处理
    )

    # 校验
    expected_len = len(style) + len(h1) + 80 + len(source)
    assert abs(len(prompt) - expected_len) < expected_len * 0.05, \
        f"render_prompt 长度异常 · 可能发生未授权 transform"

    return prompt
```

---

## 自检 checklist

每次跑完 build_render_prompts.py 后必须自检：

- [ ] 每个 render_prompt 文件存在
- [ ] 每个 render_prompt 长度 ≈ 380 + len(源文)
- [ ] 没有任何文件被截断
- [ ] diff_report.json 记录了所有 wrapper 移除和术语替换
- [ ] 抽 3 个文件人工 diff · 确认正文一字未改
