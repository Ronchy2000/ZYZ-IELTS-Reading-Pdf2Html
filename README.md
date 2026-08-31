# 写在前面：

谢谢大家的关注，本项目是一个不成熟的半成品。项目的初衷是，为了将最新的pdf转化为html即网页的形式，由于当时还在备考阶段，所以整体上比较粗糙。

后来得知一位与ZYZ老师合作的马同学一直在维护每月最新的雅思阅读题目，项目地址为：[IELTS-practice
](https://github.com/sallowayma-git/IELTS-practice)，请大家移步至该项目进行学习。

感谢ZYZ老师以及上述仓库作者的辛勤付出，我也是万千受益于题库的“烤鸭”其中的一员，很幸运能遇到如此无私奉献的老师和同学，使我最终顺利通过雅思考试。
最后附上一些本项目的出发点以及开源作者的一些思考，如果你也对本类项目感兴趣，欢迎来信、交流！

<img width="921" height="529" alt="image" src="https://github.com/user-attachments/assets/17833dd6-df9f-4f8e-a522-585fdd1174f8" />

<img width="1469" height="639" alt="image" src="https://github.com/user-attachments/assets/5e1596ea-df48-4044-85c0-86208e41378e" />


# 项目简介

将 IELTS 阅读类 PDF 批量转换为可练习的 HTML：左侧文章、右侧题目、底部导航、右下角计时器，提交后再显示答案与评分。

页面状态（作答、笔记、高亮）默认可通过本地状态服务器落盘到同目录 `*.state.json`（换电脑/清理缓存也不会丢）；未启用服务器时会回退到浏览器 localStorage。

## 目录概览
- `Script/pdf_pipeline.py`：Python 主流程（步骤 A–G：类型判定、区块定位、段落拆分、题目解析、答案解析、校验、HTML 生成）。
- `Script/templates/base.html`：统一 HTML 模板（计时、导航、保存/重置、高亮、笔记、成绩展示）。
- `Petrol power an eco-revolution.html`：模板化后的示例页面。
- `ZYZ 1月高频（139篇+29背景）/`：用户数据，不纳入版本控制。

## 环境与依赖
- Python 3.9+（推荐使用项目内虚拟环境 `.venv/`）。
- 依赖：`pdfplumber`、`pypdf`（建议再装 `pymupdf` 用于渲染/对照）。

## 使用方式
1) 创建虚拟环境并安装依赖  
```bash
python3 -m venv .venv
. .venv/bin/activate
python -m pip install -U pip
python -m pip install -r requirements.txt pillow pymupdf
```
2) 批处理示例（单 PDF 先行测试）  
```bash
. .venv/bin/activate
python Script/pdf_pipeline.py "ZYZ 1月高频（139篇+29背景）/P3（41高+9次）/1. 高频/187. P3 - Petrol power an eco-revolution 交通的革命【高】.pdf" \
  --output-dir "ZYZ 1月高频（139篇+29背景）/P3（41高+9次）/1. 高频" \
  --bundle-pdf \
  --limit 1
```
参数说明：
- `--output-dir`：输出根目录（默认与输入同级）。每个 PDF 会生成同名文件夹，内含 HTML；`--bundle-pdf` 会顺带复制源 PDF。
- `--report`：`report.csv` 路径，记录页数、字符数、扫描疑似、解析结果等。
- `--force-html`：即便校验失败（NEEDS_FIX）也生成 HTML，便于手动/Gemini 修复。
- `--limit`：调试时限制数量。

## 流水线要点（A–G）
- A：抽取前 1–2 页，计算字符数与 ASCII 比例，标记疑似扫描。
- B：按关键词页定位 Passage / Questions / Answer Key 的起止。
- C：段落拆分：有 A/B/C 标签则按标签，否则按空行生成 P1/P2…。
- D：题型识别：MCQ（含 A/B/C…选项）、TF/NG、YN/NG、填空等；保留 raw 文本便于人工或 Gemini 兜底。
- E：答案键解析：支持 “1 C / 14 ii / TRUE / YES / NG” 等格式。
- F：一致性校验：题量/答案量匹配、扫描提示；失败则标记 NEEDS_FIX。
- G：HTML 生成：沿用模板，计时器固定右下，提交后显示成绩/答案，localStorage 自动保存。

## HTML 模板特性
- 右下角计时器（暂停/恢复），底部题号导航。
- 自动保存：优先通过本地状态服务器落盘（见下），否则回退到 localStorage；`Reset` 双重确认。
- 高亮与笔记：文本选中后可高亮/移除，高亮同步保存；侧边笔记支持复制。
- 提交后显示评分与答案列表；未提交前不暴露答案。
- 简洁风格，无花哨元素，移动端自适应。

## Gemini 兜底建议
- `report.csv` 中状态为 `NEEDS_FIX` 的题目可用 `--force-html` 生成草稿，再将解析失败的块（questions/answers/raw）喂给 Gemini CLI 做结构化修复，修后回填 JSON 或直接改 HTML。

## 故障排除

### 生成的HTML页面空白
**原因**：缺少PDF处理库，或者之前版本有bug。

**解决方案**：
1. 确保已安装依赖：
   ```bash
   pip install pdfplumber pypdf
   ```

2. 删除旧的HTML文件并重新生成：
   ```bash
   python Script/pdf_pipeline.py --force-html --bundle-pdf "path/to/your.pdf"
   ```

3. 检查`report.csv`查看解析状态

### 题型识别错误
- 检查PDF格式是否标准
- 部分MCQ题目可能被误识别为TF/NG，这通常是因为选项文本中包含"no"/"yes"等关键词
- 使用`--force-html`生成后手动调整

### 答案缺失
- 部分PDF不包含答案页（如本示例的187题），这是正常的
- 生成的HTML仍可正常使用，只是不会显示参考答案
- 可以提供同名侧车答案文件让脚本自动加载：`xxx.answers.json`（放在输出目录或 PDF 同目录）。键为 `q27` / `q28`… 值为 `A/B/C/D` 或 `YES/NO/NOT GIVEN` 等。

## 注意事项
- 数据目录 `ZYZ 1月高频（139篇+29背景）/` 已在 `.gitignore`，避免把用户 PDF/答题数据推到仓库。
- `*.state.json` 已加入 `.gitignore`，用于保存作答/高亮/笔记等状态文件。
- 复位按钮有二次确认，避免误触。

## 本地落盘保存（推荐）
浏览器无法直接写本地文件（file://），所以需要用本地服务器打开 HTML 才能把状态写到磁盘。

1) 启动服务器（在项目根目录）：
```bash
python3 Script/state_server.py --root . --port 8000
```

2) 用浏览器打开 `http://127.0.0.1:8000/`，再进入对应的 HTML。

状态会保存为同目录 `xxx.state.json`（与 HTML 同名），拷贝整个文件夹到别的电脑也能继续做题。

## 已知限制
- 当前环境未预装 PDF 解析库，需手动安装后再运行脚本。
```bash
conda install -c conda-forge pdfplumber pypdf
# 或者用 pip
pip install pdfplumber pypdf

```
- 题型识别为规则版：少量特殊版式可能落入 NEEDS_FIX；请结合 Gemini/人工校正。
