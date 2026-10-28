[make_summary.py](https://github.com/user-attachments/files/34077339/make_summary.py)#!/usr/bin/env python3
"""生成实训一成果汇总 PDF（混合横版 / 竖版），并导出页面预览。"""
import base64
import html
import io
from pathlib import Path

from PIL import Image
from playwright.sync_api import sync_playwright
from pypdf import PdfReader, PdfWriter
from reportlab.pdfgen import canvas

WORKSPACE = Path("/workspace")
DELIVER = WORKSPACE / "实训一-低保真原型"
BUILD = WORKSPACE / "_build"
EXE = "/root/.cache/puppeteer/chrome/linux-155.0.8059.39/chrome-linux64/chrome"

PC_DIR = DELIVER / "03_低保真线框原型" / "PC端线框图PNG"
MOB_DIR = DELIVER / "03_低保真线框原型" / "移动端线框图PNG"
ARCH_DIR = DELIVER / "02_架构设计"
ARCH_PNG = ARCH_DIR / "架构图导出PNG"
SLICE_DIR = BUILD / "mobile_slices"
SLICE_DIR.mkdir(parents=True, exist_ok=True)

CONTENT_W = 186


def b64(path: Path) -> str:
    return "data:image/png;base64," + base64.b64encode(path.read_bytes()).decode()


def esc(text: str) -> str:
    return html.escape(text)


pc_pages = [
    {
        "file": "01_首页.png",
        "title": "首页 / Home",
        "kicker": "PC · PAGE 01",
        "task": "5 秒内说清“我是谁、做什么、为什么继续看”，并把人引向作品或联系。",
        "modules": [
            "首屏：姓名、方向、一句主张、主视觉、主 CTA",
            "精选作品：3 个案例卡片，覆盖视觉、交互、动效",
            "能力矩阵：视觉 / 交互 / 动效 / 基础前端",
            "信任证据：项目数、实习、奖项、推荐人、合作 Logo",
            "联系 CTA + 页脚导航",
        ],
        "interactions": [
            "「查看作品」→ 作品列表；卡片 → 作品详情",
            "导航常驻「联系我」；简历按钮 → 占位 PDF",
            "移动端 390px 下单列重排，导航收起为菜单",
        ],
        "file_note": "index.html · 1440 × 2944",
    },
    {
        "file": "02_作品列表.png",
        "title": "作品列表 / Works",
        "kicker": "PC · PAGE 02",
        "task": "让访客按类型快速检索 6 个项目，并在 1 次点击内进入案例。",
        "modules": [
            "页头：标题、说明、辅助信息",
            "筛选栏：全部 / 视觉 / UI 交互 / 动效影像 / 其他",
            "项目网格：封面、名称、类型、年份、查看案例",
            "底部：加载更多（示意）+ 关于页 CTA",
        ],
        "interactions": [
            "筛选按钮即时刷新卡片，并更新项目计数",
            "卡片整体可点击，进入同一案例详情模板",
            "空状态与加载更多为后续真实数据预留",
        ],
        "file_note": "works.html · 1440 × 1947",
    },
    {
        "file": "03_作品详情.png",
        "title": "作品详情 / Case Study",
        "kicker": "PC · PAGE 03",
        "task": "用完整过程与个人角色证明能力，降低“这真的是你做的吗”的怀疑。",
        "modules": [
            "标题、副标题、封面与四栏项目元信息",
            "背景与目标 / 我的角色（调研、概念、设计、交付）",
            "设计过程 3 步：调研、概念迭代、系统落地",
            "结果 3 项指标 + 最终成果图",
            "反思、下一个项目、联系 CTA",
        ],
        "interactions": [
            "面包屑返回作品列表；下一个项目可连续浏览",
            "图片区域使用灰色占位图，正式版替换为真实过程稿",
            "案例尾 CTA 直接进入联系页，形成转化闭环",
        ],
        "file_note": "work-detail.html · 1440 × 4223",
    },
    {
        "file": "04_关于.png",
        "title": "关于 / About",
        "kicker": "PC · PAGE 04",
        "task": "补充人设、经历、技能与简历，让访客在作品之外建立完整判断。",
        "modules": [
            "简介：肖像、姓名、方向、两段介绍、简历与作品按钮",
            "经历时间线：教育、实习、项目、求职状态",
            "技能与工具：设计能力 4 项、工具 4 项",
            "推荐语 2 条 + 联系 CTA",
        ],
        "interactions": [
            "「下载简历 PDF」→ 占位简历文件，正式提交前替换",
            "时间线与技能为静态信息，移动端改为单列",
            "页面底部继续引导至作品或联系",
        ],
        "file_note": "about.html · 1440 × 3075",
    },
    {
        "file": "05_联系.png",
        "title": "联系 / Contact",
        "kicker": "PC · PAGE 05",
        "task": "承接高意向访客，用最低阻力完成邮件或表单联系。",
        "modules": [
            "左侧：标题、邀请语、邮箱、回复时间、可接类型、状态",
            "右侧：姓名、邮箱、项目类型、预算、项目说明、提交按钮",
            "成功反馈与表单说明；页脚提供邮箱与社交入口",
        ],
        "interactions": [
            "表单必填校验；提交后显示示意成功状态",
            "邮箱为可点击 mailto 链接；社交为占位外链",
            "移动端表单单列，输入字号 16px，降低缩放",
        ],
        "file_note": "contact.html · 1440 × 1091",
        "extra": ("表单字段与校验", [
            "姓名（必填）· 邮箱（必填，格式校验）· 项目类型（下拉）",
            "预算（选填）· 项目说明（必填，≥ 20 字）· 提交按钮",
            "状态：空 / 聚焦 / 错误 / 成功，提交后原位反馈",
        ]),
    },
]

m_pages = [
    {"file": "01_首页.png", "title": "移动端 · 首页", "kicker": "MOBILE · PAGE 01"},
    {"file": "02_作品列表.png", "title": "移动端 · 作品列表", "kicker": "MOBILE · PAGE 02"},
    {"file": "03_作品详情.png", "title": "移动端 · 作品详情", "kicker": "MOBILE · PAGE 03"},
    {"file": "04_关于.png", "title": "移动端 · 关于", "kicker": "MOBILE · PAGE 04"},
    {"file": "05_联系.png", "title": "移动端 · 联系", "kicker": "MOBILE · PAGE 05",
     "notes": [
         ("布局策略", "全站两栏在 960px 以下折叠为单列；移动端保持“身份 → 作品 → 能力 → 联系”的纵向阅读顺序。"),
         ("表单可用性", "标签置于输入框上方，输入字号 16px 避免 iOS 自动缩放，提交按钮横跨整行。"),
         ("触控与导航", "导航收纳为抽屉菜单；按钮与卡片最小点击区域 44 × 44px，邮箱保持可点击 mailto。"),
     ]},
]


def make_slices():
    """把移动端长图切为 844px 高的视口切片，末片补白，便于汇总 PDF 阅读。"""
    result = {}
    for page in m_pages:
        src = MOB_DIR / page["file"]
        im = Image.open(src).convert("RGB")
        w, h = im.size
        slices = []
        step = 844
        idx = 0
        for top in range(0, h, step):
            idx += 1
            crop = im.crop((0, top, w, min(top + step, h)))
            canvas_im = Image.new("RGB", (w, step), "white")
            canvas_im.paste(crop, (0, 0))
            out = SLICE_DIR / f"{src.stem}_{idx}.png"
            canvas_im.save(out)
            slices.append(out)
        result[page["file"]] = slices
        page["slices"] = slices
        page["height"] = h
    return result


make_slices()

# ---------------- 横版 HTML ----------------
landscape_css = """
@page { size: A4 landscape; margin: 14mm 14mm 16mm 14mm; }
* { box-sizing: border-box; }
html, body { margin: 0; padding: 0; background: #fff; color: #111;
  font-family: "Noto Sans CJK SC","Noto Sans SC","Source Han Sans SC",sans-serif;
  font-size: 9.5pt; line-height: 1.6; -webkit-print-color-adjust: exact; print-color-adjust: exact; }
.page { height: 180mm; overflow: hidden; break-after: page; position: relative; }
.page:last-child { break-after: auto; }
.kicker { font-family: "JetBrains Mono",Consolas,monospace; font-size: 7pt;
  letter-spacing: 2px; color: #888; margin: 0 0 3px; text-transform: uppercase; }
.page-head { display: flex; align-items: flex-end; justify-content: space-between;
  gap: 20px; border-bottom: 1px solid #111; padding-bottom: 4mm; margin-bottom: 5mm; height: 14mm; }
.page-head h2 { margin: 0; font-size: 18pt; line-height: 1.2; letter-spacing: -0.01em; }
.page-head .meta { font-family: "JetBrains Mono",Consolas,monospace; font-size: 7pt;
  color: #888; text-align: right; line-height: 1.8; }
h3 { font-size: 10pt; margin: 0 0 4px; }
p { margin: 0 0 6px; }
ul { margin: 0 0 0 14px; padding: 0; }
li { margin: 2px 0; }
table { width: 100%; border-collapse: collapse; font-size: 8pt; line-height: 1.45; }
th, td { border: 1px solid #cfcfcf; padding: 3px 5px; vertical-align: top; text-align: left; }
th { background: #f2f2f2; font-weight: 700; }
.mono { font-family: "JetBrains Mono",Consolas,monospace; font-size: 7.5pt; }
.small { font-size: 8pt; color: #666; }

/* Cover */
.cover { display: flex; flex-direction: column; justify-content: space-between; }
.cover-bar { height: 7mm; background: #111; margin-bottom: 7mm; }
.cover h1 { font-size: 41pt; line-height: 1.06; margin: 0 0 4mm; letter-spacing: -0.03em; }
.cover-sub { font-size: 13pt; color: #333; margin: 0 0 3mm; }
.cover-desc { font-size: 9.5pt; color: #666; max-width: 190mm; }
.cover-stats { display: grid; grid-template-columns: repeat(4, 1fr); gap: 5mm; margin: 2mm 0 4mm; }
.cover-stat { border-top: 2px solid #111; padding-top: 3mm; }
.cover-stat b { display: block; font-size: 22pt; line-height: 1; letter-spacing: -0.03em; }
.cover-stat span { font-size: 8pt; color: #777; }
.cover-bottom { display: flex; justify-content: space-between; align-items: flex-end; gap: 20mm;
  border-top: 1px solid #ccc; padding-top: 4mm; }
.cover-toc { display: flex; gap: 9mm; flex-wrap: wrap; font-size: 8pt; color: #555; }
.cover-toc b { font-family: "JetBrains Mono",Consolas,monospace; font-size: 7pt; color: #999; display: block; }
.cover-meta { font-family: "JetBrains Mono",Consolas,monospace; font-size: 7.5pt; color: #888;
  text-align: right; line-height: 1.8; white-space: nowrap; }

/* Research */
.research { display: grid; grid-template-columns: 0.85fr 1.15fr; gap: 8mm; height: 145mm; }
.research .col { display: flex; flex-direction: column; gap: 3mm; }
.quote { border-left: 2px solid #111; padding: 2mm 0 2mm 4mm; font-size: 10.5pt; color: #222;
  background: #f7f7f7; }
.research table { font-size: 7.8pt; }
.callout { border-left: 3px solid #111; background: #f7f7f7; padding: 3mm 4mm; font-size: 8.5pt; }
.callout b { font-family: "JetBrains Mono",Consolas,monospace; font-size: 7pt; color: #888;
  display: block; margin-bottom: 1mm; letter-spacing: 1.5px; }

/* Diagrams */
.diagram { height: 140mm; display: flex; align-items: center; justify-content: center; }
.diagram img { max-height: 140mm; max-width: 100%; border: 1px solid #dedede; }
.caption { display: flex; justify-content: space-between; gap: 20px; margin-top: 2mm;
  font-size: 7.5pt; color: #777; font-family: "JetBrains Mono",Consolas,monospace; }
.diagram-notes { display: grid; grid-template-columns: repeat(3, 1fr); gap: 6mm; margin-top: 3mm; }
.diagram-notes div { border-top: 1px solid #111; padding-top: 2mm; font-size: 8pt; }
.diagram-notes b { display: block; font-size: 9pt; margin-bottom: 1mm; }

/* PC */
.pc-grid { display: grid; grid-template-columns: 102mm 1fr; gap: 10mm; height: 158mm; }
.pc-frame { border: 1px solid #d6d6d6; display: flex; align-items: flex-start; justify-content: center;
  padding: 3mm; background: #fafafa; align-self: start; }
.pc-frame img { max-height: 152mm; max-width: 96mm; width: auto; height: auto; }
.pc-notes { display: flex; flex-direction: column; gap: 4mm; height: 158mm; }
.pc-notes .block { border-top: 1px solid #111; padding-top: 2.5mm; }
.pc-notes h3 { font-size: 9.5pt; }
.pc-notes ul { font-size: 8.5pt; }
.pc-notes .task { font-size: 10pt; color: #222; }
.pc-notes .file-note { margin-top: auto; font-family: "JetBrains Mono",Consolas,monospace;
  font-size: 7.5pt; color: #888; border-top: 1px solid #ddd; padding-top: 2mm; }

/* Closing */
.score-grid { display: grid; grid-template-columns: repeat(5, 1fr); gap: 4mm; margin-top: 6mm; }
.score-grid div { border-top: 1px solid #ccc; padding-top: 2mm; font-size: 7.6pt; color: #555; }
.score-grid b { display: block; color: #111; font-size: 8.2pt; margin-bottom: 1mm; }
.score-grid + .callout { margin-top: 7mm; }

.two-col { display: grid; grid-template-columns: 1.15fr 0.85fr; gap: 10mm; }
.tree { font-family: "JetBrains Mono",Consolas,monospace; font-size: 8pt; line-height: 1.85;
  background: #f7f7f7; border: 1px solid #ddd; padding: 5mm; white-space: pre; }
.checklist { list-style: none; margin: 0; padding: 0; }
.checklist li { border-bottom: 1px solid #e3e3e3; padding: 2.2mm 0; font-size: 8.5pt; }
.checklist li::before { content: "□"; font-family: "JetBrains Mono",Consolas,monospace;
  color: #888; margin-right: 3mm; }
"""

cover = f"""
<section class="page cover">
  <div>
    <div class="cover-bar"></div>
    <p class="kicker">TRAINING 01 / LOW-FI PROTOTYPE / 2026</p>
    <h1>个人网站原型设计</h1>
    <p class="cover-sub">数字媒体艺术专业 · 个人作品集网站 · 5 页 × 双端低保真</p>
    <p class="cover-desc">本汇总包含设计调研、站点地图、用户流程、PC 端 5 页线框与移动端 5 页线框。所有页面均为黑白灰低保真，使用占位文字与灰色占位图，用于验证信息架构与浏览路径。</p>
  </div>
  <div class="cover-stats">
    <div class="cover-stat"><b>5</b><span>核心页面</span></div>
    <div class="cover-stat"><b>10</b><span>线框图（PC + 移动）</span></div>
    <div class="cover-stat"><b>2</b><span>架构图（站点 / 流程）</span></div>
    <div class="cover-stat"><b>1</b><span>可点击原型</span></div>
  </div>
  <div class="cover-bottom">
    <div class="cover-toc">
      <div><b>01</b>调研文档</div>
      <div><b>02</b>站点地图</div>
      <div><b>03</b>用户流程</div>
      <div><b>04</b>PC 端线框</div>
      <div><b>05</b>移动端线框</div>
      <div><b>06</b>交付清单</div>
    </div>
    <div class="cover-meta">班级-姓名-学号<br>2026.10 / V1.0<br>LOW-FIDELITY</div>
  </div>
</section>
"""

research = f"""
<section class="page">
  <div class="page-head">
    <div><p class="kicker">01 / RESEARCH SUMMARY</p><h2>调研结论摘要</h2></div>
    <div class="meta">个人定位 · 用户画像 · 内容分层<br>详见 01_调研文档</div>
  </div>
  <div class="research">
    <div class="col">
      <div class="quote">面向设计岗位与项目合作的数字媒体艺术应届生，用跨媒介的视觉与交互能力，把复杂信息转化为清晰、可用的体验。</div>
      <div>
        <h3>核心用户与诉求</h3>
        <table>
          <tr><th style="width:16%">优先级</th><th style="width:34%">用户</th><th>核心诉求</th></tr>
          <tr><td>P0</td><td>校招 HR / 设计负责人</td><td>30–90 秒判断方向匹配、作品成熟度与联系方式</td></tr>
          <tr><td>P1</td><td>研究生导师 / 招生老师</td><td>看过程、方法论、反思与持续产出</td></tr>
          <tr><td>P2</td><td>同行 / 潜在合作方</td><td>看风格、能力边界与合作可能</td></tr>
        </table>
      </div>
      <div>
        <h3>三条设计需求</h3>
        <ul>
          <li><b>首屏说清楚</b>：姓名 + 方向 + 一句主张 + 一个主 CTA。</li>
          <li><b>案例可验证</b>：背景、角色、过程、结果、反思五段式。</li>
          <li><b>联系无阻力</b>：导航、首屏、案例尾、页脚均设联系入口。</li>
        </ul>
      </div>
      <div>
        <h3>成功标准</h3>
        <p class="small">5 秒识别率 ≥ 80% · 3 击到达任一案例 · 390px 无横向滚动 · 5 页路径闭环 · 10 张线框齐全</p>
      </div>
    </div>
    <div class="col">
      <div>
        <h3>内容清单与信息分层</h3>
        <table>
          <tr><th style="width:15%">页面</th><th style="width:44%">L1 一级信息</th><th>L2 / L3 次级信息</th></tr>
          <tr><td>首页</td><td>身份与主张、主视觉、精选作品、联系 CTA</td><td>能力矩阵、信任证据、页脚导航</td></tr>
          <tr><td>作品列表</td><td>标题、说明、类型筛选、项目卡片</td><td>项目计数、加载更多、空状态</td></tr>
          <tr><td>作品详情</td><td>项目名、一句话成果、封面、角色与元信息</td><td>背景、过程、结果、反思、下一个项目</td></tr>
          <tr><td>关于</td><td>简介、简历下载、方向</td><td>时间线、技能与工具、推荐语</td></tr>
          <tr><td>联系</td><td>邮箱、表单、提交按钮</td><td>回复时间、可接类型、状态、社交链接</td></tr>
        </table>
      </div>
      <div>
        <h3>阅读顺序</h3>
        <p class="small">首页：身份（0–5s）→ 精选作品（5–20s）→ 能力与证据（20–40s）→ 联系。<br>
        案例：封面与成果 → 背景 → 角色 → 过程 → 结果 → 反思 → CTA。</p>
      </div>
      <div>
        <h3>本阶段边界</h3>
        <p class="small">只做黑白灰线框与占位内容；不做色彩、图标、装饰与复杂动效。视觉方向、字体与图片风格留到实训二，且不得反向修改信息架构。</p>
      </div>
    </div>
  </div>
  <div class="callout" style="margin-top:3mm;">
    <b>LOW-FI CONCLUSION</b>
    信息架构收敛为 5 个核心页面：首页负责吸引，作品列表负责检索，案例详情负责说服，关于负责补充信任，联系负责收口。全站无孤立页面，任一内页均可返回首页或进入联系。
  </div>
</section>
"""

sitemap = f"""
<section class="page">
  <div class="page-head">
    <div><p class="kicker">02 / SITE MAP</p><h2>站点地图</h2></div>
    <div class="meta">5 个核心页面 · L0 → L1 → L2<br>5 CORE PAGES</div>
  </div>
  <div class="diagram"><img src="{b64(ARCH_PNG/'站点地图.png')}" alt="站点地图"></div>
  <div class="caption"><span>FILE: 02_架构设计/站点地图.svg · 架构图导出PNG/站点地图.png</span><span>1600 × 860</span></div>
  <div class="diagram-notes">
    <div><b>层级关系</b>首页为唯一入口；作品、关于、联系为一级栏目；作品详情、简历 PDF、表单为二级动作。</div>
    <div><b>全局导航</b>顶部导航与页脚常驻 4 个入口；案例页尾与全站页脚均设置联系 CTA。</div>
    <div><b>闭环检查</b>无孤立页面；任一内页可返回首页；作品详情可回列表，联系页可回任意栏目。</div>
  </div>
</section>
"""

userflow = f"""
<section class="page">
  <div class="page-head">
    <div><p class="kicker">03 / USER FLOW</p><h2>用户流程图</h2></div>
    <div class="meta">主路径 · 回退分支 · 低意向分支<br>FLOW CHECK</div>
  </div>
  <div class="diagram"><img src="{b64(ARCH_PNG/'用户流程图.png')}" alt="用户流程图"></div>
  <div class="caption"><span>FILE: 02_架构设计/用户流程图.svg · 架构图导出PNG/用户流程图.png</span><span>1700 × 900</span></div>
  <div class="diagram-notes">
    <div><b>主路径</b>首页 → 作品列表 → 筛选 → 作品详情 → 联系页 → 表单 / 邮件 → 提交成功。</div>
    <div><b>回退分支</b>未找到案例 → 调整筛选；不认可能力 → 返回列表或看下一个案例。</div>
    <div><b>低意向分支</b>首页 → 关于 → 下载简历 → 结束；导航与案例尾仍保留联系快捷入口。</div>
  </div>
</section>
"""

pc_html = ""
for page in pc_pages:
    modules = "".join(f"<li>{esc(x)}</li>" for x in page["modules"])
    interactions = "".join(f"<li>{esc(x)}</li>" for x in page["interactions"])
    extra = ""
    if page.get("extra"):
        label, items = page["extra"]
        lis = "".join(f"<li>{esc(x)}</li>" for x in items)
        extra = f'<div class="block"><h3>{esc(label)}</h3><ul>{lis}</ul></div>'
    pc_html += f"""
<section class="page">
  <div class="page-head">
    <div><p class="kicker">{esc(page['kicker'])}</p><h2>{esc(page['title'])}</h2></div>
    <div class="meta">PC 1440px · 低保真<br>WIREFRAME</div>
  </div>
  <div class="pc-grid">
    <div class="pc-frame"><img src="{b64(PC_DIR/page['file'])}" alt="{esc(page['title'])}"></div>
    <div class="pc-notes">
      <div class="block"><h3>页面任务</h3><p class="task">{esc(page['task'])}</p></div>
      <div class="block"><h3>模块顺序</h3><ul>{modules}</ul></div>
      <div class="block"><h3>关键交互</h3><ul>{interactions}</ul></div>
      {extra}
      <div class="file-note">{esc(page['file_note'])}</div>
    </div>
  </div>
</section>
"""

closing = """
<section class="page">
  <div class="page-head">
    <div><p class="kicker">06 / DELIVERABLES</p><h2>交付清单与替换说明</h2></div>
    <div class="meta">FILE STRUCTURE<br>CHECKLIST</div>
  </div>
  <div class="two-col">
    <div>
      <h3>目录结构</h3>
      <div class="tree">实训一-低保真原型/
├─ 01_调研文档/
│  ├─ 个人定位与需求分析.pdf
│  └─ AI辅助设计说明.pdf
├─ 02_架构设计/
│  ├─ 站点地图.svg
│  ├─ 用户流程图.svg
│  └─ 架构图导出PNG/（站点地图、用户流程图）
├─ 03_低保真线框原型/
│  ├─ 原型工程源文件/
│  │  ├─ index / works / work-detail / about / contact.html
│  │  ├─ style.css / script.js
│  │  ├─ 个人网站低保真原型.fig（Figma 工程 · 电脑端）
│  │  ├─ 个人网站低保真原型.xd（Adobe XD 工程 · 电脑端）
│  │  └─ assets/（占位图与简历 PDF）
│  ├─ PC端线框图PNG/（5 张）
│  ├─ 移动端线框图PNG/（5 张）
│  └─ 可点击在线原型链接.txt
└─ 04_成果汇总.PDF</div>
    </div>
    <div>
      <h3>提交前检查</h3>
      <ul class="checklist">
        <li>5 个核心页面齐全，PC + 移动双端各 5 张 PNG</li>
        <li>站点地图与用户流程图含源文件与 PNG 导出</li>
        <li>原型可点击，页面之间跳转正常，无死路</li>
        <li>线框仅黑白灰，占位文字与灰色占位图</li>
        <li>调研文档与 AI 辅助说明为 PDF，内容完整</li>
        <li>PC / 移动线框图无横向滚动，长图完整可读</li>
        <li>压缩包内无临时文件、缓存目录与重复文件</li>
        <li>根目录命名为“班级-姓名-学号-个人网站原型设计”</li>
      </ul>
    </div>
  </div>
  <div class="score-grid">
    <div><b>01 · 需求定位与内容分层</b>定位陈述含角色、方向、受众；内容清单按 L1–L3 分层，五页各有明确任务。</div>
    <div><b>02 · 站点地图与用户流程</b>三层结构完整，5 页全部可达；主路径、回退分支、低意向分支均无断点。</div>
    <div><b>03 · 线框图页面齐全</b>5 个核心页面 × PC / 移动双端，共 10 张 PNG，统一黑白灰与占位内容。</div>
    <div><b>04 · 跳转交互与源文件</b>可点击原型含全站导航与页内跳转；除 HTML/CSS/JS 外，另提供 .fig 与 .xd 二进制工程源文件，5 个电脑端画板全部可编辑，移动端以 PNG 呈现。</div>
    <div><b>05 · AIGC 使用说明</b>记录工具、环节、提示词与人工二次优化，无直接照搬，边界与合规声明清晰。</div>
  </div>
  <div class="callout">
    <b>REPLACE BEFORE SUBMISSION</b>
    正式提交前请替换：姓名 / 班级 / 学号、真实项目与过程稿、真实数据、推荐人、邮箱与社交链接、简历 PDF、在线原型部署链接。源文件已包含 HTML/CSS/JS 与 .fig、.xd 两套二进制工程文件（均为电脑端 5 页）：.fig 用 Figma 导入即可打开，.xd 用 Adobe XD 打开即可编辑；若本机缺少 Noto Sans SC / Roboto Mono 字体，导入后统一替换字体即可。
  </div>
</section>
"""

landscape_html = f"""<!DOCTYPE html><html lang="zh-CN"><head><meta charset="UTF-8">
<title>实训一成果汇总 · 横版</title><style>{landscape_css}</style></head><body>
{cover}{research}{sitemap}{userflow}{pc_html}{closing}
</body></html>"""

# ---------------- 竖版移动端 HTML ----------------
portrait_css = """
@page { size: A4 portrait; margin: 12mm 12mm 14mm 12mm; }
* { box-sizing: border-box; }
html, body { margin: 0; padding: 0; background: #fff; color: #111;
  font-family: "Noto Sans CJK SC","Noto Sans SC","Source Han Sans SC",sans-serif;
  font-size: 9pt; line-height: 1.55; -webkit-print-color-adjust: exact; print-color-adjust: exact; }
.m-page { height: 271mm; overflow: hidden; break-after: page; }
.m-page:last-child { break-after: auto; }
.kicker { font-family: "JetBrains Mono",Consolas,monospace; font-size: 7pt;
  letter-spacing: 2px; color: #888; margin: 0 0 3px; }
.page-head { display: flex; align-items: flex-end; justify-content: space-between;
  gap: 12px; border-bottom: 1px solid #111; padding-bottom: 3mm; margin-bottom: 5mm; height: 15mm; }
.page-head h2 { margin: 0; font-size: 17pt; line-height: 1.2; }
.page-head .meta { font-family: "JetBrains Mono",Consolas,monospace; font-size: 7pt;
  color: #888; text-align: right; line-height: 1.8; }
.m-grid { display: grid; grid-template-columns: repeat(3, 52mm); gap: 4mm 5mm;
  justify-content: center; align-content: start; }
figure { margin: 0; }
figure img { width: 52mm; height: auto; display: block; border: 1px solid #bbb; }
figcaption { font-family: "JetBrains Mono",Consolas,monospace; font-size: 6.8pt;
  color: #888; padding-top: 1mm; display: flex; justify-content: space-between; }
.m-note { margin-top: 4mm; border-top: 1px solid #ccc; padding-top: 2mm;
  font-size: 8pt; color: #666; text-align: center; }
.m-notes { display: grid; grid-template-columns: repeat(3, 1fr); gap: 5mm; margin-top: 9mm; }
.m-notes div { border-top: 1px solid #111; padding-top: 2.5mm; font-size: 8pt; color: #555; }
.m-notes b { display: block; font-size: 8.8pt; color: #111; margin-bottom: 1.5mm; }
.m-table { width: 100%; border-collapse: collapse; font-size: 7.8pt; margin-top: 8mm; }
.m-table th, .m-table td { border: 1px solid #cfcfcf; padding: 2px 4px; text-align: left; }
.m-table th { background: #f2f2f2; font-weight: 700; }
"""

mobile_html = ""
for page in m_pages:
    slices = page["slices"]
    figs = ""
    for i, s in enumerate(slices, 1):
        figs += f"""<figure><img src="{b64(s)}" alt="{esc(page['title'])} 切片 {i}">
<figcaption><span>{i:02d} / {len(slices):02d}</span><span>390px</span></figcaption></figure>"""
    notes = ""
    table = ""
    if page.get("notes"):
        items = "".join(f"<div><b>{esc(k)}</b>{esc(v)}</div>" for k, v in page["notes"])
        notes = f'<div class="m-notes">{items}</div>'
        table = """<table class="m-table">
<tr><th style="width:18%">页面</th><th style="width:34%">双端信息架构</th><th style="width:28%">导航与跳转</th><th>横向滚动</th></tr>
<tr><td>首页</td><td>一致，两栏折叠为单列</td><td>抽屉导航 + 首屏 CTA</td><td>无</td></tr>
<tr><td>作品列表</td><td>一致，3 列变 1 列</td><td>筛选 + 卡片整块可点</td><td>无</td></tr>
<tr><td>作品详情</td><td>一致，元信息两栏折叠</td><td>面包屑 + 下一个项目</td><td>无</td></tr>
<tr><td>关于</td><td>一致，肖像与简介堆叠</td><td>简历下载 + 作品跳转</td><td>无</td></tr>
<tr><td>联系</td><td>一致，信息与表单单列</td><td>表单提交 + 邮件链接</td><td>无</td></tr>
</table>"""
    mobile_html += f"""
<section class="m-page">
  <div class="page-head">
    <div><p class="kicker">{esc(page['kicker'])}</p><h2>{esc(page['title'])}</h2></div>
    <div class="meta">390 × {page['height']}<br>{len(slices)} SLICES</div>
  </div>
  <div class="m-grid">{figs}</div>
  {notes}
  {table}
  <p class="m-note">按 390px 视口切分长图，便于在 A4 页面阅读；完整长图见“移动端线框图PNG”文件夹。</p>
</section>
"""

mobile_html = f"""<!DOCTYPE html><html lang="zh-CN"><head><meta charset="UTF-8">
<title>实训一成果汇总 · 移动端</title><style>{portrait_css}</style></head><body>
{mobile_html}
</body></html>"""

BUILD.joinpath("summary_landscape.html").write_text(landscape_html, encoding="utf-8")
BUILD.joinpath("summary_mobile.html").write_text(mobile_html, encoding="utf-8")

# ---------------- 打印 PDF ----------------
land_pdf = BUILD / "summary_landscape.pdf"
mob_pdf = BUILD / "summary_mobile.pdf"
footer = ("<div style=\"width:100%;font-family:'Noto Sans CJK SC',sans-serif;font-size:7.5px;"
          "color:#999;padding:0 14mm;display:flex;justify-content:space-between;\">"
          "<span>个人网站原型设计 · 实训一成果汇总</span>"
          "<span>LOW-FIDELITY / V1.0</span></div>")

with sync_playwright() as p:
    browser = p.chromium.launch(executable_path=EXE, args=["--no-sandbox"])
    for src, out, landscape in [
        ("summary_landscape.html", land_pdf, True),
        ("summary_mobile.html", mob_pdf, False),
    ]:
        pg = browser.new_page()
        pg.goto((BUILD / src).as_uri(), wait_until="networkidle")
        pg.evaluate("document.fonts.ready")
        pg.pdf(
            path=str(out),
            format="A4",
            landscape=landscape,
            print_background=True,
            display_header_footer=True,
            header_template="<div></div>",
            footer_template=footer,
            margin={"top": "14mm" if landscape else "12mm",
                    "bottom": "16mm" if landscape else "14mm",
                    "left": "14mm" if landscape else "12mm",
                    "right": "14mm" if landscape else "12mm"},
        )
        print(src, "->", out, len(PdfReader(out).pages), "pages")
        pg.close()
    browser.close()

# ---------------- 合并 + 连续页码 ----------------
reader = PdfReader(str(land_pdf))
mobile_reader = PdfReader(str(mob_pdf))
all_pages = list(reader.pages) + list(mobile_reader.pages)
total = len(all_pages)
writer = PdfWriter()

for idx, page in enumerate(all_pages, 1):
    w = float(page.mediabox.width)
    h = float(page.mediabox.height)
    buf = io.BytesIO()
    c = canvas.Canvas(buf, pagesize=(w, h))
    c.setFont("Helvetica", 7.5)
    c.setFillColorRGB(0.55, 0.55, 0.55)
    c.drawRightString(w - 34, h - 15, f"{idx:02d} / {total:02d}")
    c.save()
    buf.seek(0)
    overlay = PdfReader(buf).pages[0]
    page.merge_page(overlay)
    writer.add_page(page)

final_pdf = DELIVER / "04_成果汇总.PDF"
with final_pdf.open("wb") as fh:
    writer.write(fh)

print("FINAL", final_pdf, len(PdfReader(final_pdf).pages), "pages")
