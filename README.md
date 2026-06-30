# aigo 电商主图生成 Skill

基于 ARK Seedream 图生图能力，自动生成 aigo 品牌电商主图矩阵。

## 能力

- **多平台产品图抓取**：京东 / 抖音商城自动抓取，天猫/淘宝多图上传降级
- **5 张差异化电商主图**：白底主推 / 尺寸对比 / 生活场景 / 数据巨字 / 促销爆款
- **LOGO 像素级合成**：官方 LOGO 资产后期 composite，100% 还原不重绘
- **硬性约束自检**：LOGO 排他区污染率检测，>5% 自动返工
- **飞书投递**：生成后自动推送到指定飞书会话

## 设计规范

| 约束 | 规则 |
|---|---|
| LOGO 位置 | 仅左上角（方形版）/ 顶部居中（横版）|
| LOGO 合成 | PIL alpha_composite，禁止 ImageMagick（避免黑边重影）|
| 产品保真 | 可换角度但不可虚构结构 |
| 风格多元化 | 5 张图背景/色调/构图/标题策略全部不同 |
| 文字约束 | ≤4 处文字单元，主标题 4-7 字，全部避开 LOGO 排他区 |
| 电商风格 | 参考天猫/京东/抖音真实跑量主图，避免 AI 风 |

## 资产

```
aigo-main-image/
├── SKILL.md                       # 完整工作流（含代码模板）
├── logo_official_square.png       # 方形版 LOGO（800×800，左上角用）
├── logo_official_horizontal.png   # 横版 LOGO（800×158，顶部居中用）
└── README.md
```

## 依赖

- ARK API（`ARK_API_KEY` + `ARK_BASE_URL`）
- PIL / numpy（LOGO 合成与自检）
- bua（浏览器自动化，京东/抖音抓取）
- openclaw message send（飞书投递）
