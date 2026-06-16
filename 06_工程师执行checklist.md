# 06 · 工程师执行 checklist

> 每次做 PPT 都按这 8 步执行 · 跳步 = 工程失败

---

## 8 步执行流程

### Step 1 · 准备项目目录

```bash
项目根目录/
├── 00_source/             ← 用户把源文件放这里
├── 01_style_overlay/      ← 从 SOP 复制公司视觉规范
├── 02_render_prompt/      ← 留空 · 自动生成
├── 03_render_report/      ← 留空 · 自动生成
└── 04_output/             ← 留空 · 自动生成
```

- [ ] 4 个子目录都已创建
- [ ] `01_style_overlay/` 已放入 `公司视觉规范_精简版.md`（380 字符）

---

### Step 2 · 源文件入库

- [ ] 所有源文件 `*.md` 放入 `00_source/`
- [ ] 文件名规范：`L1_P{页码}_{主题}_{版本}.md`
- [ ] 不动 `01_style_overlay/`

---

### Step 3 · 跑 build_render_prompts.py

```python
# 伪代码 · 严格按 02_render_prompt_拼装规则.md
for source in 00_source/*.md:
    render = build_render_prompt(source, style_overlay)
    write(02_render_prompt/{name}.prompt.md, render)
```

- [ ] `02_render_prompt/` 下每个源文件对应一个 `*.prompt.md`
- [ ] 无任何文件丢失

---

### Step 4 · 校验 render_prompt

每个 render_prompt 必须满足：

- [ ] 长度 ≈ 380 + len(H1) + 80 + len(源文) （偏差 < 5%）
- [ ] 头部是公司视觉规范精简版（380 字符）
- [ ] 中部是"本页主题：XXX"
- [ ] 尾部是完整源文（不删 / 不改 / 不截断）

抽 3 个文件人工 diff 确认。

---

### Step 5 · 跑 route_and_render.py（按密度分流出图）

```python
for prompt in 02_render_prompt/*.prompt.md:
    real_chars = count_real_chars(对应源文)
    size = decide_size(real_chars)  # 1k / 2k / 4k
    image_url = gpt_image_2(prompt, size=size)
    save(04_output/images/{name}.png)
    log_to_route_json(...)
```

密度分流（详见 04_出图路由_密度分流.md）：

| 字符数 | size |
|---|---|
| ≤ 3,000 | 1k |
| 3,000-5,000 | 2k |
| **5,000-8,000** | **2k** |
| **> 8,000** | **4k** |

- [ ] 每页都已出图
- [ ] `03_render_report/density_route.json` 已产出
- [ ] density_risk = high / extreme 的页已提醒用户

---

### Step 6 · 落 diff_report.json

```python
# 严格按 05_diff_report_schema.json 字段
report = {
    "project": "...",
    "sop_version": "v1.0",
    "build_time": "...",
    "files": [...],
    "summary": {...}
}
write(03_render_report/diff_report.json, report)
```

- [ ] `diff_report.json` 字段完整
- [ ] `summary.total_truncations == 0`
- [ ] `summary.failed_files == 0`
- [ ] 所有 transforms 都已记录

---

### Step 7 · 抽检 OCR 漏字率

```python
# 抽 5 张图（含 1 张 high/extreme 密度）
for img in random.sample(images, 5):
    ocr_text = ocr(img)
    source_text = read_source(对应文件)
    miss_rate = compute_miss_rate(source_text, ocr_text)
    if miss_rate > 5%:
        mark_for_revision(img)
```

- [ ] 5 张图抽检完成
- [ ] 漏字率 > 5% 的页已上报用户
- [ ] 用户决策：接受 / 重出 / 降级矢量

---

### Step 8 · 合成最终 PPT

```python
from pptx import Presentation
from pptx.util import Emu

prs = Presentation()
prs.slide_width = Emu(12192000)   # 16:9 · 1920
prs.slide_height = Emu(6858000)   # 16:9 · 1080

for img_path in sorted(images):
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    slide.shapes.add_picture(img_path, 0, 0, prs.slide_width, prs.slide_height)

prs.save(04_output/L1_v4_完整版.pptx)
```

- [ ] PPT 文件已生成
- [ ] 页数 = 源文件数
- [ ] 顺序正确（按页码排序）
- [ ] 文件已上传 AI Drive

---

## 完工标志

只有满足以下全部条件才算完工：

- [ ] `04_output/` 下有最终 PPT
- [ ] `03_render_report/diff_report.json` 通过校验
- [ ] `03_render_report/density_route.json` 完整
- [ ] OCR 抽检结果已上报
- [ ] 用户已签收

任一项缺失 = **工程未完工 · 不得交付**。
