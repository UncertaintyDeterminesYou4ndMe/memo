# RGB → CMYK 印刷色彩预处理

## 背景

设计稿（PNG）在 Adobe Illustrator 里转 CMYK 时，鲜艳的绿色、黄色会明显变暗发灰。这不是软件 bug，是物理现象：RGB 是发光色彩（屏幕），色域远大于 CMYK（油墨反光）。荧光绿这类颜色，油墨根本调不出来。

用 Python 代码做预处理，可以把色差降到最低，同时生成印刷厂能直接用的 CMYK TIFF。

---

## 核心方案

用 macOS 内置 ICC Profile + Pillow 做颜色空间转换：

```python
from PIL import Image, ImageCms

SRGB_PROFILE = "/System/Library/ColorSync/Profiles/sRGB Profile.icc"
CMYK_PROFILE = "/System/Library/ColorSync/Profiles/Generic CMYK Profile.icc"

srgb = ImageCms.getOpenProfile(SRGB_PROFILE)
cmyk = ImageCms.getOpenProfile(CMYK_PROFILE)

to_cmyk = ImageCms.buildTransformFromOpenProfiles(
    srgb, cmyk, "RGB", "CMYK",
    renderingIntent=ImageCms.Intent.PERCEPTUAL  # 整体色彩关系保持最好
)
cmyk_img = ImageCms.applyTransform(rgb_img, to_cmyk)

cmyk_img.save(
    output_path,
    format="TIFF",
    icc_profile=open(CMYK_PROFILE, "rb").read(),  # 嵌入 ICC，Illustrator 直接识别
    compression="tiff_deflate",  # 关键：DEFLATE 而非 LZW
    dpi=(300, 300),
)
```

**两个容易踩的坑**：

1. **RGBA 必须先拼合白底**。印刷不支持透明通道，直接转会报错或颜色错乱。
   ```python
   if img.mode == "RGBA":
       canvas = Image.new("RGB", img.size, (255, 255, 255))
       canvas.paste(img, mask=img.split()[3])
   ```

2. **不要用 LZW 压缩**。TIFF 默认或常用的 LZW 对密集彩色插画几乎没压缩效果，4301×5719 的图会产生 94MB 文件。改用 `tiff_deflate`（DEFLATE 算法，和 PNG 一样），同样文件压到 3.6MB。

---

## 关键判断

**为什么用 PERCEPTUAL 渲染意图而不是 RELATIVE_COLORIMETRIC？**

- `PERCEPTUAL`：整体色彩关系保持最好，色域外颜色被"压缩"进可印刷范围，视觉上自然
- `RELATIVE_COLORIMETRIC`：色域内颜色精确映射，色域外颜色被截断，可能更鲜艳但会失真

插画和包装设计，整体色调关系比单色精确更重要，选 PERCEPTUAL。

**为什么 TIFF 而不是直接给 PNG？**

PNG 只能是 RGB，Illustrator 导入后依然要触发转换。TIFF 可以嵌入 CMYK ICC Profile，Illustrator 识别后无需再转换，色差为零。

---

## 色差验证方式

做 RGB→CMYK→RGB 的"圆形映射"，对比还原前后的像素差：

```python
to_rgb = ImageCms.buildTransformFromOpenProfiles(cmyk, srgb, "CMYK", "RGB", ...)
preview = ImageCms.applyTransform(cmyk_img, to_rgb)

# 计算平均像素差
orig_pixels = list(rgb_img.getdata())
prev_pixels = list(preview.getdata())
avg_delta = sum(abs(a-b) for o,p in zip(orig_pixels, prev_pixels) for a,b in zip(o,p)) / (n * 3)
```

| 评级 | 平均色差 |
|------|----------|
| 🟢 优秀 | < 5/255 |
| 🟡 良好 | 5–12/255 |
| 🔴 注意 | > 12/255 |

包装盒设计实测：平均色差 1.9/255，超色域像素 0%，说明原图颜色本身已在 CMYK 安全范围内。

---

## 还没做但值得做的优化

当前方案只解决了"色域边界"，印刷还有三个物理问题未处理：

**1. TIC（Total Ink Coverage）控制**
某些像素 C+M+Y+K 可能超过 340%，墨水来不及干燥会蹭墨。包装印刷通常要求 ≤ 300%。
正确做法：增加 K、等量减少 CMY（GCR，灰成分替换），保持颜色不变。

**2. Dot Gain 补偿**
油墨落纸后向四周渗透，中间调会系统性偏暗 10-25%。
需要根据纸张类型（铜版纸/哑粉纸/新闻纸）预亮化，用 Gamma 曲线补偿。

**3. 用正确的 ICC Profile**
macOS 的 `Generic CMYK Profile` 是通用配置，没有针对真实印刷机。
亚洲印厂标准是 `Japan Color 2001 Coated`，需要单独下载。
用错 Profile，屏幕模拟结果和实际印刷就会对不上。

---

## 封装为 Claude Skill

将脚本封装成 `rgb-to-cmyk` skill，结构：

```
rgb-to-cmyk/
├── SKILL.md          # 触发描述 + 工作流程
└── scripts/
    └── convert.py    # 自包含的转换脚本
```

关键：脚本路径硬编码为 `~/.claude/skills/rgb-to-cmyk/scripts/convert.py`，ICC Profile 用 macOS 系统路径，不依赖外部文件。

同步到 `~/.agents/skills/` 供多 agent 共用（参考：`claude-skills-sync方案.md`）。
