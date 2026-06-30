---
name: aigo-main-image
description: aigo品牌电商主图生成。当用户要求为aigo/爱国者品牌产品生成电商主图、产品主图、营销图、商品图时触发。支持从京东/抖音链接自动抓取产品图，或接收用户上传的多张产品图。严格遵循LOGO神圣化、产品保真、风格多元化、LOGO排他等多条硬性约束规则，基于ARK Seedream图生图能力生成5张不同风格的1:1电商主图并投递到飞书。
---

# aigo 电商主图生成

## 目标
基于用户提供的产品图（链接抓取或多图上传），生成 5 张符合当下天猫/京东/抖音商城主图标准的 aigo 品牌电商主图，并投递到飞书。

## 支持的产品类型
仅支持以下主体类型：
- 数码产品（录音笔、充电宝等）
- 汽车（暂未实测）
- 生活家电产品（暂未实测）

## 必需输入
用户必须提供以下之一：
1. **产品链接**（京东 / 抖音商城）→ 自动抓取产品图和详情
2. **多张产品图**（3-6 张不同角度）→ 天猫/淘宝/未知平台或抓取失败时的降级路径

品牌名自动识别为 aigo/爱国者。

---

## 任务类型识别（强制先识别）

| 任务类型 | 判定关键词 | 输出特征 |
|---|---|---|
| **电商主图矩阵**（默认） | "主图"、"电商图"、"商品图"、"X张" | 1:1 带 LOGO + 产品名 + 卖点的电商推广风格主图 |
| **营销海报** | "海报"、"营销"、"传播图"、"含文案" | 含主标题/副标题/卖点的合成海报，需等用户确认文案后才出图 |

> **默认行为契约**：用户给"产品链接 + 张数"时，默认走电商主图矩阵流程，不每步追问。

---

## 默认行为契约（已确认）

### 默认参数
- 任务类型：电商主图矩阵
- 数量：用户指定（默认 5 张）
- 比例：1:1（2048×2048）
- 模式："事实上 A 模式"（ARK Seedream 4.0 图生图，保主体强提示词）
- 投递目标：飞书私聊 `ou_5839fe4c0748566a7dc925b767ea022b`
- 投递形式：N 条独立图片消息（每张一条）

### 跳过的环节
- ❌ 不再问"A 还是 B 任务类型"
- ❌ 不再问"严格 A 还是事实上 A"
- ❌ 不再问"卖点关键词"（agent 从详情页/参考图提取）
- ❌ 不再问"比例"
- ❌ 不再问"投递目标/形式"

### 仍需对齐的环节（hard stop）
- ✅ 主体不在支持范围 → 终止流程并告知
- ✅ 用户明确说"营销海报/含文案" → 走任务 A 完整流程
- ✅ 链接来自天猫/淘宝/不支持平台 或 抓图失败 → 发多图上传提示
- ✅ ARK 接口异常 → 报错并寻求决策

---

## Step 0：产品信息获取

### 0.1 平台路由

| 平台 | host 关键词 | 抓取方式 | 状态 |
|---|---|---|---|
| 京东 | `jd.com` | `bua` + 移动端 URL `item.m.jd.com/product/<sku>.html` | ✅ |
| 抖音商城 | `jinritemai.com` | `bua` + Swiper background-image 抓取 | ✅ |
| 天猫/淘宝 | `tmall.com` / `taobao.com` | **不支持**，走 0.3 多图上传降级 | ❌ |
| 其他 | — | 走 0.3 多图上传降级 | ❌ |

### 0.2 各平台抓取细节

**京东**：
```bash
bua start --session jd_test --url "https://item.m.jd.com/product/<sku>.html"
sleep 5
bua --session jd_test eval "..."  # 抽 document.images 中带 360buyimg.com 的高清版本
```

**抖音商城**：
- 不需要登录即可渲染商品页（价格被打码为 `1??`）
- 主图在 CSS `background-image` 而非 `<img>` 标签——遍历 `.swiper-slide` 的 `getComputedStyle().backgroundImage`
- 所有 slide 的 bg URL 从一开始就全部存在，无需驱动 Swiper 切换
- 图片 host：`*.ecombdimg.com`，无防盗链

```js
// 抓取代码模板
const mainImgs = new Set();
document.querySelectorAll('.swiper-wrapper .swiper-slide').forEach(s => {
  s.querySelectorAll('*').forEach(el => {
    const m = getComputedStyle(el).backgroundImage.match(/url\("?(https:[^")]+)"?\)/);
    if (m && m[1].includes('ecombdimg')) mainImgs.add(m[1]);
  });
});
```

**天猫/淘宝**：全入口强制登录，mtop API 需 `_m_h5_tk` token，自动抓取直接放弃。

### 0.3 多图上传降级（通用兜底）

触发条件：天猫/淘宝链接、抓取失败、用户只发了 1 张图

提示模板：
> 这个平台无法自动抓取商品图（或抓取失败）。请你直接把产品图发给我，建议 **3–6 张不同角度**：正面、侧面、细节、包装、与人/手对比的实拍。

收到图后：
1. 全部保存到 `workspace/poster_output/source_*.jpg`
2. 选主体最完整、分辨率最高、无遮挡的那张作为图生图 `image` 入参，存为 `source_main.jpg`
3. 其余图片用于 agent 提炼"必须保留的具象元素"写进保主体契约
4. 品牌/产品名 + 主体类别由用户口头给出

---

## 强约束规则（R1–R5，违反即返工）

### R1 — LOGO 神圣化（最严约束）

- **唯一 LOGO 来源**：`workspace/poster_output/assets/` 中两张官方资产，**禁止修改/裁剪/调色**
  - `logo_official_square.png`（800×800 方形版："aigo" 大字 + 下方"国民好物"叠层）→ **左上角使用**
  - `logo_official_horizontal.png`（800×158 横版："aigo 国民好物"横排）→ **顶部居中位置使用**
- LOGO 是 **PNG 带 alpha 透明底**，必须直接 composite 与底色融合，**禁止套白色矩形/圆角矩形/任何硬边底板**
- **禁止**从详情页或参考图里裁切 LOGO，**禁止**把 LOGO 还原任务交给 Seedream
- **LOGO 位置只允许两处**：
  - **左上角**（NorthWest）→ 用方形版
  - **顶部居中**（North）→ 用横版
  - **禁止**：底部居中 / 中部 / 右上 / 右下 / 任何其他位置
- 位置选择：用 PIL + numpy 计算 TL（380×380）和顶居中（1000×200）两区域平均 luminance，选更干净的位置
- 5 张图组里两个位置都要出现（推荐比例约 3:2）

### R2 — 产品保真

- 产品可从不同角度展示（正面/侧面/45°/特写/俯视/悬挂/手持/躺放）
- **不允许虚构**：禁止凭空加按钮、改颜色、改结构、改材质、改比例
- 5 张图里产品角度尽量分散

### R3 — 风格多元化

5 张图之间强制差异化：
- 背景类型不同（白底 / 场景 / 渐变 / 实拍 / 抽象）
- 主色调不同（至少 4 种）
- 信息密度不同（极简 / 中等 / 密集）
- 构图不同（居中 / 偏左 / 偏右 / 上下分割）

### R4 — 重复项尽量少

- 同一个卖点关键词最多出现 2 次
- 色块形状不重复
- 卖点位置不重复
- 任意两张视觉雷同就算违规

### R5 — 多重标题策略

每张图主标题用不同策略，5 张图覆盖至少 4 种：
- 利益型 / 场景型 / 数据型 / 对比型 / 疑问型 / 身份型

### R1+ 附加 — LOGO 排他约束

- LOGO 占据的矩形区域内 **绝对不允许任何主标题/副标题/卖点标签/角标/数据徽章/任何文字图形**
- 标准 LOGO 占位（含 40px padding 排他区）：
  - 左上角：460×460 像素（y=0~460, x=0~460）
  - 顶部居中：780×220 像素（y=0~220，横向居中）
- **出图后必须自检**：用 numpy 计算 LOGO 排他区内"非纯色像素占比"，> 5% 则该图返工

---

## 电商主图设计规范

### 参考标准
参考天猫/京东/抖音真实跑量主图的共性特征：
- 文字单元 ≤ 4 处（不要堆砌）
- 主标题 4-7 字、副标题 6-12 字
- 产品本体视觉权重 ≥ 50%（文字是辅助而非主角）
- 品牌色克制、对比强烈但不噪

### 主流主图类型（5 张各选一种）
1. **白底主推款**：产品居中 + 标准电商阴影 + 极简文字 + 红色角标
2. **尺寸对比图**：产品 vs 口红/硬币并排 + 数据标注
3. **生活场景图**：自然窗光 + 暖木桌面 + 手持产品的真实瞬间
4. **数据巨字主图**：深色背景 + 巨大亮色数字（180h / 7h / 20g）
5. **促销主图**：红底/黄底 + 白色标题 + 对话气泡卖点

### 避免的设计风格
❌ 霓虹光晕、紫色密集渐变、思维导图节点、抽象科技几何、AI 风密集小标签

---

## 执行流程（必须按此顺序）

### Step A：准备 LOGO 资产
确认 `workspace/poster_output/assets/` 中有 4 个 LOGO 文件：
- `logo_official_square.png`（原版 800×800）
- `logo_official_square_white.png`（反白版，深色背景用）
- `logo_official_horizontal.png`（原版 800×158）
- `logo_official_horizontal_white.png`（反白版）

若反白版不存在，用 PIL 一次性生成：
```python
from PIL import Image
import numpy as np

def make_white(in_path, out_path):
    im = Image.open(in_path).convert("RGBA")
    arr = np.array(im)
    a = arr[:,:,3] / 255.0
    rgb = arr[:,:,:3].astype(np.float32)
    is_dark = (a > 0.05) & (rgb[:,:,0] < 80) & (rgb[:,:,1] < 80) & (rgb[:,:,2] < 80)
    is_red  = (a > 0.05) & (rgb[:,:,0] > 180) & (rgb[:,:,1] < 130) & (rgb[:,:,2] < 130)
    rgb[is_dark] = [255, 255, 255]
    rgb[is_red]  = [255, 90, 90]
    arr[:,:,:3] = np.clip(rgb, 0, 255).astype(np.uint8)
    Image.fromarray(arr, "RGBA").save(out_path)
```

### Step B：制定 5 张图差异矩阵
列出每张图的差异化坐标（角度 / 背景类型 / 主色调 / 标题策略 / 卖点切片），写成表格自检，确保无重复。

### Step C：生成 ARK Seedream prompt
- 保主体契约：列出产品颜色/按键/LOGO/金属环/挂绳孔等具体元素
- LOGO 排他区：明确告诉 AI "左上 460×460 矩形 / 顶居中 780×220 矩形内绝对不要放任何文字图形"
- 文字单元 ≤ 4 处，全部避开 LOGO 排他区

### Step D：ARK Seedream 调用

```bash
# 5 张并行调用（xargs -P5 控制并发，避免 429）
ENDPOINT="${ARK_BASE_URL}/images/generations"

curl -s -X POST "$ENDPOINT" \
  -H "Authorization: Bearer $ARK_API_KEY" \
  -H "Content-Type: application/json" \
  -d @/tmp/req_<slug>.json \
  -o poster_output/vX/responses/<slug>.json
```

请求 body：
```json
{
  "model": "doubao-seedream-4-0-250828",
  "prompt": "...",
  "image": "data:image/jpeg;base64,<B64>",
  "size": "2048x2048",
  "response_format": "url",
  "watermark": false,
  "sequential_image_generation": "disabled",
  "stream": false
}
```

⚠️ base64 不能塞命令行参数（超 ARG_MAX），必须用 Python 写到 JSON 文件再用 `curl -d @file`。

### Step E：LOGO 自检（出图后第一步）
```python
from PIL import Image
import numpy as np

def check(raw_path, region):
    im = Image.open(raw_path).convert("RGB")
    arr = np.array(im); H,W,_ = arr.shape
    if region == 'TL':
        crop = arr[:460, :460]
    else:
        crop = arr[:220, (W-780)//2:(W+780)//2]
    gray = 0.299*crop[:,:,0] + 0.587*crop[:,:,1] + 0.114*crop[:,:,2]
    m = gray.mean()
    bright = (gray > m + 50).sum()
    dark = (gray < m - 50).sum()
    pollution = (bright + dark) / crop[:,:,0].size * 100
    return m, pollution  # pollution > 5% → 返工
```

### Step F：LOGO 合成（PIL alpha_composite，不用 ImageMagick）

```python
from PIL import Image
import numpy as np

def composite(bg_path, logo_path, target_w, gravity, padding, out_path, white_version=False):
    bg = Image.open(bg_path).convert("RGBA")
    BW, BH = bg.size
    logo = Image.open(logo_path).convert("RGBA")
    arr = np.array(logo).astype(np.float32)
    a = arr[:,:,3] / 255.0
    if white_version:
        rgb = arr[:,:,:3]
        is_dark = (a>0.05) & (rgb[:,:,0]<80) & (rgb[:,:,1]<80) & (rgb[:,:,2]<80)
        is_red  = (a>0.05) & (rgb[:,:,0]>180) & (rgb[:,:,1]<130) & (rgb[:,:,2]<130)
        rgb[is_dark] = [255,255,255]; rgb[is_red] = [255,90,90]
        arr[:,:,:3] = rgb
    arr_u8 = np.clip(arr, 0, 255).astype(np.uint8)
    logo2 = Image.fromarray(arr_u8, "RGBA")
    H, W = arr.shape[:2]
    th = int(target_w * H / W)
    logo_resized = logo2.resize((target_w, th), Image.LANCZOS)
    px, py = padding
    pos = (px, py) if gravity == 'NW' else ((BW - target_w)//2, py)
    bg.alpha_composite(logo_resized, dest=pos)
    bg.convert("RGB").save(out_path, quality=95)
```

参数：
- 方形版 LOGO：`target_w=380, gravity='NW', padding=(60,60)`
- 横版 LOGO：`target_w=700, gravity='N', padding=(0,60)`
- 深色背景（luminance < 100）：`white_version=True`，用反白版

### Step G：投递飞书

```bash
OPENID="ou_5839fe4c0748566a7dc925b767ea022b"

for img in poster_output/vX/images/*.jpg; do
  result=$(openclaw message send --channel feishu --target "$OPENID" --media "$img" --message " " --json 2>&1)
  json=$(echo "$result" | sed -n '/^{/,$p')
  ok=$(echo "$json" | jq -r '.payload.ok // false')
  mid=$(echo "$json" | jq -r '.payload.messageId // ""')
  echo "$(basename $img): ok=$ok mid=$mid"
  sleep 1  # 避免频控
done
```

⚠️ `openclaw message send --json` 输出前有 `[info]: [ 'client ready' ]` 日志，必须用 `sed -n '/^{/,$p'` 提取 JSON，再用 `.payload.ok`（不是顶层 `.ok`）。

---

## 标准产物布局

```
workspace/poster_output/
├── assets/
│   ├── logo_official_square.png          # 方形版原版
│   ├── logo_official_square_white.png    # 方形版反白
│   ├── logo_official_horizontal.png      # 横版原版
│   └── logo_official_horizontal_white.png # 横版反白
├── source_main.jpg            # 选定的主原图
├── source_*.jpg               # 用户上传的其他原图
└── v<N>/
    ├── responses/             # ARK 调用返回 JSON
    │   └── <slug>.json
    ├── raw/                   # ARK 返回的原始成品图（未合成 LOGO）
    │   └── <slug>.jpg
    └── images/                # 合成 LOGO 后的最终成品图
        └── <slug>.jpg
```

---

## 关键工程经验

1. **京东 PC 版会强制跳登录** → 用 `item.m.jd.com/product/<sku>.html` 移动端 URL
2. **base64 不能塞命令行参数**（230KB 就超 ARG_MAX）→ 写 JSON 文件 `curl -d @file`
3. **当前可用模型**：`doubao-seedream-4-0-250828`（v4.5 和 `ep-xxx` endpoint 在当前网关 AccessDenied）
4. **文生图最小尺寸** 3686400 像素，图生图无此限制
5. **TOS 签名 URL 24h 过期**，生成后必须立即下载到本地
6. **飞书 `--media` 多次传只保留最后一个**，多图必须循环调用
7. **`bua` 必须带 `--session <sid>`**，否则报错
8. **抖音商城主图在 CSS background-image**，不在 `<img>` 标签——所有 Swiper slide 初始渲染就带完整 bg URL
9. **天猫/淘宝全入口强制登录**，自动抓取直接放弃
10. **`openclaw message send --json` 输出有 `[info]` 日志污染** → `sed -n '/^{/,$p'` 提取 JSON，`.payload.ok` 不是 `.ok`
11. **LOGO 合成必须用 PIL `alpha_composite`**（不是 ImageMagick `-composite`）→ ImageMagick 会预乘 alpha 产生黑边重影
12. **保主体提示词越具体越好**：列出"颜色/按键/LOGO/金属环/挂绳孔"等具象元素
13. **5 张图差异化必须事前规划差异矩阵**，事后挑选容易雷同
14. **深色背景用反白版 LOGO**（黑字→白字，alpha 完整保留），不是套白底矩形
