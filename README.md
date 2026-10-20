# 笔刷转换器 · Procreate ⇄ 画世界Pro

一个**纯前端**的笔刷双向转换器：把 Procreate 的 `.brush` / `.brushset` 与画世界Pro 的 `.bru`
互相转换，让笔刷在两端都能真正使用。所有解析、转换、打包都在浏览器里完成，**文件不离开你的电脑**。

---

## ⚠️ 当前进度

| 模块 | 状态 | 说明 |
|---|---|---|
| 工程骨架 + 网页 UI | ✅ 完成 | `index.html` / `css` / `js/app.js`，支持拖拽、预览、导出中间 JSON |
| bplist 解析 | ✅ 完成 | `js/bplist.js`，解析 Apple 二进制属性列表 |
| NSKeyedArchiver 解码 | ✅ 完成 | `js/nskeyed.js`，重建 SilicaBrush 对象图 |
| 通用中间格式 | ✅ 完成 | `js/schema.js`，双向转换的枢纽 |
| Procreate 解析（读取） | ✅ 已完成并**端到端验证** | `js/procreate.js`；已用真实结构的 `.brush` 样本验证，字段名按真实 SilicaBrush 键（`paintSize`/`plotSpacing`/`dynamicsPressureSize`/`textureScale`/`taperStartLength`/`dynamicsMix` 等）校准 |
| **画世界Pro .bru 逆向** | ✅ 已取得突破 | 容器结构 + 每图前的 protobuf 元数据块已完整解包（字段表见下文）；**字段与画世界Pro 设置项的对应关系仍未确认**（缺配对样本） |
| .bru 读取器 | ✅ 已实现并验证 | `js/bru.js`：魔数 `hsjbru` / 版本 / 文件大小校验 + 逐 chunk 提取全部嵌入 PNG + 按结构启发式分类（笔尖/纹理/封面/形状） |
| **.bru 压感处理** | ✅ 已修复 | 参数未解码期间默认写入通用压感曲线（`szVr.bVTy=2` / `dynamicsPressureSize=0.65`），界面可关；此前 ABR 写 `bVTy=0` 导致导入 Procreate 无压感（用户实测反馈） |
| **`.brushset` 笔刷清单** | ✅ 已修复 | 笔刷清单改由 `brushset.plist`（XML plist）驱动；此前按「叶子文件夹」规则会把 Procreate 写在 `<uuid>/Reset/`、`<uuid>/SubNN/` 里的内部母版当成笔刷（实测 14 支而非 7 支，组名也读错） |
| **内置笔尖缺失处理** | ✅ 已修复 | Procreate 内置笔尖只以**文件名**（`bundledShapePath`）引用、**不随导出包分发**，文件里根本没有像素。现在明确报出缺失的内置笔尖名，并用默认圆形笔尖兜底使转换不中断（此前直接抛「没有笔尖图像」整单失败） |
| **内置形状库可视选择器** | ✅ v45 起 | 笔尖缺失时弹出可视化选择器：**网格模式**（292 个真实笔尖 + 搜索 + 当前指定高亮，打开时懒解码）与**总览图模式**（内嵌 292 tips 总览图，点格子直接指定；格子→库键映射经 OCR + 图像相似度逐格核对，292/292 双射） |
| **外层 ZIP 包裹** | ✅ 已修复 | 外层 ZIP 里包着 `.brushset`/`.brush`（macOS 压缩、聊天工具、网盘常见）时自动解包，此前直接报「ZIP 内没有 Brush.archive」 |
| **笔尖极性 `shapeInverted`** | ✅ 已修复 | Procreate 笔尖图规范是「**白=着墨**」，但笔刷可在 `Brush.archive` 里置 `shapeInverted=true` 把规则翻转成「暗=着墨」。此前一律按亮度直读，导致这类笔刷整幅变成 2048² 纯色方块、图案被挖成空洞（文件也从 8MB 虚胖到 11.7MB）。现在按旗标反相**再裁剪**，笔尖回归图案本身 |
| **bplist 编码（写）** | ✅ 完成并**双向交叉验证** | `js/bplist-writer.js`：生成 Apple 二进制 plist；与 Python `plistlib` 互读一致（负数/bool/float/UTF-16/UID/binary 全通过） |
| **Procreate 写回器** | ✅ 完成并**端到端验证** | `js/procreate-writer.js`：生成合法 `.brush`（`Brush.archive` + `Shape.png` + `Grain.png` + `QuickLook/Thumbnail.png`），19 项参数往返无损 |
| **PNG → alpha 蒙版** | ✅ 完成并验证 | `js/png-alpha.js`：浏览器走 `createImageBitmap`+canvas，Node 走内置 PNG 解码器（zlib + 全滤波器链，8/16 位、颜色类型 0/2/3/4/6）；bbox 裁剪 + 面积平均降采样至 2500px |
| **ABR 写回器（通用输出）** | ✅ 完成并**三方验证** | `js/abr-writer.js`：生成 Photoshop ABR v6.2（`8BIM samp/patt/desc/phry` + 264 字节 VMAL + Action Descriptor v16 完整 32 项 brushPreset）。PackBits 与真实 Photoshop 输出**逐字节一致**；经**独立 Rust 解析器（brushkit）**回读验证笔尖/名称/动力学全部正确 |
| **双向转换联调** | ✅ `.bru → .brush` 已打通 | 真实 12.5MB `.bru` → 538KB `.brush`，笔尖/纹理/封面**字节级无损**（`tools/test_pipeline.js`） |
| **`.bru` / `.brush` → `.abr`** | ✅ 已打通 | 用户真实 `.bru`（2048² alpha 蒙版）→ 97KB `.abr`，笔尖平面与源 PNG alpha **逐字节一致**（`tools/test_abr_e2e.js`）；多笔刷打包带 Grup 层级（`tools/test_abr_multi.js`） |
| .bru 写回器（反向） | ⛔ 未实现 | 私有格式写回风险高且**无法本地验证**（需在画世界Pro 真机导入测试）；当前给出明确提示而非生成不可用文件 |

**一句话现状**：**任意笔刷（`.brush`/`.brushset`/`.bru`）→ 通用 `.abr`** 全链路已跑通并经独立解析器验证；`.bru → .brush` 亦字节级无损；内置形状库配套**可视笔尖选择器**（网格 + 292 tips 总览图双模式，v45）。反向生成 `.bru` 因私有格式无法本地验证，暂不提供。

---

## 🎯 默认输出：Adobe `.abr`（为什么）

Procreate 与画世界Pro（3.0+）都支持导入 `.abr`，因此转换器把**所有投入的笔刷统一输出为 `.abr`**，
一个格式同时满足两端（也能在 Photoshop 里用）。

**已验证的格式细节**（对真实 Photoshop CS -era 文件逐字节逆向标定）：

- 文件头 `major=6, subversion=2`；段顺序 `samp` → `patt`(空) → `desc` → `phry`
- `samp` 条目：Pascal 式 UUID 字符串 + 264 字节 VMAL 元数据 + 矩形 + `depth=8` + PackBits + 8 字节尾填充
- `desc`：Action Descriptor v16 → `Brsh` VlLs → **32 项 brushPreset**（键序与 Photoshop 完全一致）
- `phry`：单笔刷为空列表（Photoshop 行为）；多笔刷为 `Grup` + N×`preset` + `groupEnd`

**参数映射**：`Dmtr = max(w,h)`、`Spcn = 间距%`、`Rndn`、`Angl`、压感→`szVr`/`opVr`/`prVr`(bVTy=2)、
倾斜→笔尖倾斜、速度→淡出、随机旋转→`angleDynamics`、散布→`useScatter`/`Cnt `/`scatterDynamics`、
`render.blendMode`→`Md  ` 枚举。

> ⚠️ **ABR 的固有限制**：不携带 Procreate 的纹理(Grain)、渲染模式(Wet Mix/渗流)、锥度(taper)与
> 颜色动态。笔尖位图与大小/间距/压感等核心行为完整迁移，上述参数会被丢弃。

### 两步完成：转换 → 一键导入

界面把流程压缩成两步，不再让用户在「下载目录里找文件 → 打开 App → 导入」之间来回跳：

1. 拖入文件，点 **转换** —— 页面立刻生成目标文件（不自动下载）
2. 主按钮变成 **「导入 Procreate / Photoshop」**（按当前目标格式显示对应文案）——
   点它直接调起系统分享面板，选目标 App 即可导入；
   若浏览器不支持网页分享文件（如桌面端），自动降级为下载并附导入路径指引
3. 想直接拿文件也行：**「仅下载文件」** 次级按钮随时可点，拿走 `.abr` / `.brush` 自己导

> 📱 **能力边界（如实说明）**：纯前端网页无法直接调起 Procreate/画世界Pro 的内部导入接口，
> 分享面板是浏览器允许的最短路径——在手机上点「导入」后选 Procreate，即等同于「一键导入」；
> 桌面上则退化为「下载 + 一行导入指引」。生成 `.brush` 时导入指引为：iPad「文件」App 打开或
> AirDrop 后「用 Procreate 打开」。

---

## ✅ 现在能做什么

1. 拖入 **Procreate `.brush`/`.brushset`** → 解析全部参数与图片 → 预览 → 点「转换」生成 `.abr`（默认）或 `.brush`，再点「导入」直达目标 App
2. 拖入 **画世界Pro `.bru`**（若上传被拦，改名为 `.bru.zip` 即可）→ 自动识别并提取
   **笔尖（alpha 蒙版）/ 纹理 / 封面 / 形状**等全部嵌入图像 → 点「转换」→「导入」直达目标 App
3. 一次拖入多支笔刷（`.brushset`）→ 合并导出为**一个** `.abr` 笔刷组（带 Grup 层级）
4. 笔刷组被外层 ZIP 包着（macOS 压缩 / 聊天工具 / 网盘）→ 自动解包
5. 笔刷用的是 Procreate **内置**笔尖（文件里没有笔尖图）→ 明确报出缺失的笔尖名，
   用默认圆形笔尖兜底，参数照常转换、转换不中断
6. 笔尖图极性由 `shapeInverted` 决定（白场暗图=反相着墨 / 黑场白图=直接着墨）→
   按旗标反相后再裁剪，笔尖保持图案原样（如蕾丝包 58 支 2048² 实心方块 → 图案尺寸）

> `.brush` 的可用性已在**两个独立解析器**（本项目解析器 + Python `plistlib`）上验证通过；
> `.abr` 已在**独立 Rust 解析器（brushkit crate）**上回读验证。

### 笔尖图（Shape.png）多级查找

笔刷组的笔尖**不一定**在 `<笔刷目录>/Shape.png`：第三方笔刷制作工具、分享导出、
二次打包的包会把它改名、挪进子目录、塞进 `Brush.archive`（NSData / base64），
或集中放到笔刷组根目录。这些包在 Procreate 里一切正常，但只认标准路径的转换器
会全部报「缺少笔尖图」。因此读取器按以下级联查找，并把**命中的规则**记录在每支
笔刷上（UI 卡片显示「笔尖：…」）：

1. `<笔刷目录>/Shape.png`（精确匹配）
2. 同目录任意大小写的 `shape.png`
3. 同目录下其他 PNG（`Brush.png` / `Tip.png` …；`Grain.png` 与 `Preview.png` 永不充当笔尖）
4. 笔刷目录子目录下的 PNG（如 `Shape/Shape.png`）
5. `Brush.archive` 内嵌的 PNG（NSData 二进制或 base64 字符串）
6. 笔刷组内以该笔刷 UUID 命名的文件（笔刷档案带 `uuid` 字段时精确匹配）
7. 根目录/共享目录里的零散笔尖 PNG，数量与缺笔尖笔刷数相等时按顺序匹配（UI 会标注「按顺序匹配」）

数量不匹配时**不猜**（宁缺勿错），同时在页面「文件内部结构」面板里展开完整
ZIP 清单（每个笔刷目录含哪些文件、全部 PNG 路径与大小），便于从截图定位异常包。
命令行取证：`node tools/diag_brushset.js <文件.brushset>`。

#### 还有一种情况：笔尖图根本不在文件里

Procreate 的笔刷**按名字**引用笔尖，不是按像素。`Brush.archive` 里只有
`bundledShapePath` / `bundledGrainPath` 两个字符串；图片要么在笔刷目录里
（`Shape.png`），要么——当这支笔刷用的是 **Procreate 内置**笔尖时——在
Procreate App 自己的资源库里。**导出 `.brushset` 时内置资源不会被打包**，
因为 Procreate 假设导入方也是 Procreate，导入时按名字去自己的库里找就行了。

这就是「为什么画世界Pro 转 ABR 一直正常、Procreate 转 ABR 找不到笔尖」：
`.bru` 把图片**嵌在文件内部**（自包含），而 `.brushset` 只是**引用文件名**。
用真实样本「雨后清明@阿昔.zip」在字节级验证过：ZIP 内 217 个条目、
72 张 PNG 全是预览图（Thumbnail / Signature / AuthorPicture），
**一个笔尖图都没有**；36 个 `Brush.archive` 里也搜不到任何图像魔数，
只有 `bundledShapePath = "Brush-Preset-Paint-Dab-1.jpg"` 这样的文件名。

这种包在 Procreate 里使用完全正常，但**任何**第三方工具都无法从文件中取出
这张图。现在的处理分三级：

1. **内置形状库**（见下）：`bundledShapePath` 命中的笔刷直接取得真实笔尖；
2. 库外名字：卡片上给出「从形状库选择笔尖」按钮，打开**可视选择器**手动指定，
   映射会被记住（localStorage，换文件、换会话持续有效）。选择器见下节；
3. 都找不到：明确报出缺失的内置笔尖文件名，同时用默认圆形笔尖兜底，
   让参数照常转换、转换不中断。

#### 内置形状库（把 Procreate 的笔尖收进来）

既然内置笔尖不随包分发，就把它们**预先收录进转换器**：
用户先后提供 13 张 Procreate 笔刷库截图（含 1024×31172 的源库长截图，
覆盖整个内置形状库），按单元格几何（块高 435px、行周期 570px、左列
x=54–590、右列 x=616–1152）逐格提取，配合公开资源库
（tohsakarat/Brush-Converter 的内置形状原图，CC BY-NC 4.0）**全量收录**，
生成一份**内部文件名 → 笔尖灰度图**的映射：

- `js/shape-library-data.js`：**292 个主条目、1174 个引用键**（内部文件名、
  显示名、连字符/无空格/小写变体互相注册为别名），灰度 PNG 白=着墨，
  前两批走 512px、全量批走 256px，共约 14 MB，整体内嵌进单文件 HTML
  （实测 15 MB 页面加载约 1.4 s）；
- `js/shape-library.js`：解析优先级 = 库直连 → 用户手动映射 → 未命中；
  带扩展名互换兜底（`.jpg`/`.png`/大小写）；
- 显示名与内部文件名一对一确定（如 Thai Ink 2 → `Brush-Pocket-Thai-Ink2.jpg`、
  Flat Brush 2 → `Brush-Preset-Flat-Brush2.jpg`），两种写法都能命中。

收录边界：Procreate 内置库里的**形状与纹理都在其中**（仓库未按类型拆分），
所以这些图既能当笔尖也能当纹理用。极性统一按「白=着墨」，
纯色图（如全白的 Blank）不做反相——否则会变成完全透明的空笔尖。

效果（「雨后清明@阿昔.zip」，7 支笔刷全部引用内置笔尖）：

| | 形状库上线前 | 上线后 |
|---|---|---|
| 笔尖 | 7/7 合成圆形兜底 | 7/7 真实笔尖（Brush-Preset-Paint-Dab-1.jpg、Acrylic-Stamp-5.png、Brush-Artery-Fat.jpg） |
| 输出 .abr | 49 KB | 714 KB |
| 卡片标注 | 「已用默认圆形笔尖代替」 | 「内置形状库 · 真实笔尖名」 |

#### 可视笔尖选择器（网格 + 总览图双模式）

笔尖缺失时卡片上的「从形状库选择笔尖」打开一个模态选择器，两种浏览模式：

- **网格模式（默认）**：库中 292 个主条目逐个渲染成卡片（真实笔尖图 + 名称 + 尺寸），
  顶部搜索框按名称/内部文件名过滤，当前已指定的条目高亮，底部可「清除此笔刷的手动指定」。
  292 个形状的 base64 全量解码约 14MB，打开时一次性解码会卡顿，因此缩略图改为
  滚动到可见区域再解码（IntersectionObserver，rootMargin 400px；不支持时退化为立即解码），
  blob URL 有缓存，反复打开/搜索不重复解码。
- **总览图模式**：内嵌一张 292 tips 的全库总览图（1112×6064，8 列网格），
  一眼扫全库；图上叠 292 个透明热区，**点格子即完成指定**（悬停显示真实形状名）。
  总览图为 AI 合成图（页面标注「AI生成」），部分标签在格子边缘被裁切，
  因此格子→库键映射不依赖图上文字：格子几何逐格测定，标签经 OCR 读取，
  239 个名称直配 + 53 个截断标签按图像相似度指派，最终 **292↔292 双射、
  与库内 PNG 相似度中位 0.948**，并人工目检了 16 对最低相似度（全部一致）。
  总览图以 WebP q75 内嵌（约 860KB）；看不清或找不到时一键切回网格模式。

两种模式点击后都走同一条落库链路：`ShapeLibrary.setUserMapping(bundledShapePath, shapeId)`
→ 写入 localStorage → 笔刷卡片立即换成真实笔尖 → 之后转换自动应用。

#### 内置纹理（Grain）链路：bundledGrainPath 也从库找回

`Brush.archive` 的 **`bundledGrainPath`** 与 `bundledShapePath` 是同一套机制：
笔刷引用一张 Procreate 内置纹理（如 `X.png`、`Brush-Artery-Ocean.jpg`），
导出包里**不随包分发**这张图。转换器现在把这条链路也接上了：

- `js/procreate.js`：`parseZip` 提取 `bundledGrainPath` 与 5.2+ 的四个纹理通道
  （`bundledRoughnessPath` / `bundledMetallicPath` / `bundledHeightPath` /
  `bundledTexturePath`）；`archiveToSchema` 对 grain 名走与笔尖相同的库查找
  （随包分发的 `Grain.png` 优先，其次查库）；
- 命中后卡片与诊断面板标注「纹理：内置形状库 · 原名」，`.brush` 写回自动带上
  真实 `Grain.png`（实测 61 KB 的 .brush 内 Grain.png / Shape.png 俱全）；
- ABR 没有纹理通道，所以纹理对 .abr 只用于预览/诊断——UI 里如实说明。

同样用「雨后清明@阿昔.zip」验证：7 支笔刷**全部**引用内置纹理
（4 支 `X.png` 半调网点、3 支 `Brush-Artery-Ocean.jpg`），7/7 从库取回真实像素
（`tools/test_grain_e2e.js`，26 项断言含 .brush 写回）。

#### 纹理显示名映射（第四轮截图：2 张纹理库长截图）

用户又提供 2 张 Procreate **纹理库**长截图（1024×16860 + 1024×10122，按名字
文字行像素定位逐行识别，行周期 485px），覆盖 111 个纹理显示名。逐个人工核对
（含缩略图视觉比对消歧）后录入 `build_shape_lib.py` 的
`TEX_DISPLAY_TO_INTERNAL`，**108/111 个显示名已能解析到库中真实纹理像素**：

- 纹理内部名有三种前缀：`Brush-Artery-*`（4.x 主力）、`Brush-Preset-*` /
  `Brush-Pocket-*`（5.x 新增），映射时逐一验证；
- 源库文件名有 Procreate 的拼写变体：显示名 Charcoal **Coarse** → 文件名
  `Brush-Artery-Charcoal-Corse.jpg`、Orange **Peel** → `Brush-Artery-Orange-Peal.jpg`；
- 同名不同图：「Bark」在库里有**两张**（`Brush-Artery-Bark` 浅色裂纹 /
  `Brush-Preset-Bark` 深色碎屑），显示名映射注册其一，另一张仍可靠内部名解析；
  同类还有 Brick / Canvas / Wood，均经缩略图视觉比对确认；
- 资源库中缺失的 3 个：**Agate / Concrete / Dry Acrylic Wash**（截图里有，
  tohsakarat 仓库没有对应原图），已如实记录，其余 108 个全部生效。

收录后形状库 **292 主条目 / 1624 键**（原 1174），单文件 15.1 MB，
线上实测加载约 1.6 s。

#### 笔尖的极性：白是墨，但笔刷可以要求反相

`Shape.png` 的明暗含义不是固定的。Procreate 的规范是**「白=着墨、黑=透明」**
（brushfactory 的格式解析器同样记录 "Inked parts are white in both Shape and
Grain"），而 `Brush.archive` 里的布尔字段 **`shapeInverted`** 可以把单支笔刷
翻过来：置 `true` 时**暗色部分才是着墨区**。

用真实样本「呼呼｜蕾丝1.0.zip」（59 支）在字节级验证过：

| | 蕾丝笔刷 ×58 | 通知笔刷 ×1 |
|---|---|---|
| `Shape.png` | 2048² **白场** + 暗色蕾丝图案 | 2048² **黑场** + 白色声明文字 |
| `shapeInverted` | **true** | **false** |

两支笔刷的图极性正好相反，旗标也正好一一对应（58:1，零例外）；缩略图里两者
都渲染为黑底白图，与「白=墨 + 按旗标反相 + 白色预览色」单一规则完全吻合。
导入时不反相的后果（西班牙语教程原文）：*"el resultado será un rectángulo
opaco con la silueta en negativo"*——一块纯色方块，图案呈负片——正是修复前
转换器的输出：58 个 2048×2048 实心笔尖、文件虚胖到 11.7MB。

现在的处理：`js/png-alpha.js` 的 `invert()` 在**裁剪之前**反相 alpha 平面
（顺序很重要——先裁剪的话白场永远占满全图、bbox 裁剪失效），笔尖回到图案
本身（如 651×652），文件降到 8.0MB；`shapeInverted=false` 的笔刷不受影响。
通用格式带 `tip.invert` 标志，写回 `.brush` 时也会把旗标写回 `Brush.archive`。

---

## 🔬 .bru 逆向发现（首个样本「回忆里绕圈@夏.bru」, version 5）

文件魔数为 **`hsjbru`**，是大端二进制容器（非 ZIP、未加密）。结构如下：

| 偏移 | 内容 | 说明 |
|---|---|---|
| `0x00` | `68 73 6a 62 72 75` = `hsjbru` | 魔数 |
| `0x06` | uint32 **大端** = `5` | 版本号 |
| `0x0A` | uint32 **大端** = 文件总字节数 | 样本中为 `12515825`，与实際大小**完全吻合** |
| `0x0E`… | 0 填充 | 直到第一个嵌入 PNG |

之后是一串**完整的 PNG 图像流**（无压缩、无加密），每张图（除第一张外）前有一段元数据块，
其末尾 16 字节模式为：
`[前导: ff ff ff 01 | 01 98 05 01 | ff ff ff ff ff ff 01] a0 05 <类型> a8 05 02 b0 05 02 d0 05 01`
其中 `<类型>` 为小整数（实测 02–0b），疑似**图像角色 ID**；元数据块内还含形如
`01ri_0SN6bCHGYY-9nTQ` 的 **hash 字符串**（资源 ID）。

样本中嵌入 **17 张 PNG**，经像素统计分类：

| 角色 | 特征 | 样本中的图像 |
|---|---|---|
| **封面/预览** | 最大图、不透明、彩色 | `#0` 2880×2880 RGBA（书法字「红尘」展示图） |
| **笔尖（TIP）** | 大尺寸 RGBA、**RGB 全黑 + alpha 变化**（alpha 蒙版式笔尖） | `#13` 2048×2048（`rgbstd=0`，alpha mean≈28） |
| **纹理（GRAIN）** | 不透明 RGB 照片/布纹/皮革 | `#1 #3 #7 #8 #9 #14 #15`（200×200） |
| **形状/笔触** | 透明 RGBA 线稿/涂鸦 | 其余 200×200 透明图 |
| 重复 | `img_03` 与 `img_14` 字节完全一致 | 容器存在重复资源 |

**关键限制**：笔刷名「回忆里绕圈」**未**以明文 UTF-8 存入文件（由 App 侧管理）。

### 元数据块 = protobuf（数值参数就在这里）

每张图前面的 300–700 字节元数据块是** protobuf wire format 的消息**（前 50 字节左右是二进制头，之后从 `0a 16 …` 开始是完整消息），已用 `tools/bru_meta_table.py` 完整解出。以笔尖图 `#13` 前的块为例（共 52 个字段）：

| 字段 | 类型 | 值 / 说明 |
|---|---|---|
| f1 | string(22) | 资源 hash ID，如 `fH01ri_0SN6bCHGYY-9nTQ` |
| f2 | string(22) | 恒为 `3eHQCJOOSGSVRJAFqMa0mw`（作者账号 ID） |
| f3 | string | 全局序号 `"01"…"17"`（跳过 3 个大块，说明它们也是条目） |
| f4 | string(13) | 资源 ID（`#3`/`#14` 相同，字节也相同） |
| f6 | string | `"pt25"` `"pt22"` … 资源类型码 |
| f7 | string(13) | 毫秒时间戳（≈2024-05-01，随序号递增） |
| f8 / f20 | float | 100–395 / 41–288 的整数型浮点（大小/像素？） |
| f12 / f37 / f85 / f86 / f90 | 恒定 | 500 / 23 / 2 / 2 / 1 |
| f16 / f82 | varint | 恒为 `-1`（哨兵值） |
| f22–f27、f17、f18 | float | 0–4 之间的小数（不透明度/流量/间距/抖动候选） |
| f33 / f35 | varint | 20–100 的整数（百分比候选） |
| f54 | string | **中文资源名**：`图形` / `笔刷` / `常用` |
| f75–f78 | float | 同组资源共享（`#10`–`#13` 均为 8 / 4.61329 / 0.93 / 0.45） |
| f84 | varint | **图像角色 ID**（2/3/4/5/6/7/9/11，即早先看到的 `a0 05 <n>`） |
| f81 | float | 0.64 / 1.0 |

字段语义**尚未与画世界Pro 的设置面板对应**（只有一个样本，无法验证哪个字段是"压感-大小"、哪个是"间距"）。块内还出现过 3 个 ~3.27 MB 的"噪声条目"——大部分是 0，夹杂华为相机 AWB 校准数据（`APAWB` / `HUAWEI` / `awb_ext1`），来源不明。

### 压感：为什么转过去"没有压感"，以及现在的处理

**原因**：`.bru` 的数值参数解不出来 → 转换器只能填默认值 → 而默认值里压感是 0：
ABR 写入器把 `szVr`/`opVr`/`prVr` 的 `bVTy`（Photoshop 动力学控制码）写成 `0`（off），
`.brush` 写入器把 `dynamicsPressureSize` 写成 `0`。两个格式都明确表示"不随压力变化"，
导入 Procreate 后笔刷就成了定值印章——但笔尖/纹理图像都正常迁移，所以"能画、没压感"。
（已用 Rust 独立校验器复盘：修复前 `pSizeCtl=None`，修复后 `pSizeCtl=Some(2)`。）

**现在的处理**：`.bru` 来源默认写入一条**通用压感曲线**（界面复选框可关）：

| 输出 | 写入内容 |
|---|---|
| `.abr` | `szVr.bVTy=2`（钢笔压力）+ jitter 65% + `Mnm` 15%；`opVr.bVTy=2` + jitter 45%；`minimumDiameter` 15% |
| `.brush` | `dynamicsPressureSize=0.65`、`dynamicsPressureOpacity=0.45`、`minSize` 比例 0.3 |

这是**缺省行为而非解码结果**——曲线幅度可在 Procreate 笔刷工作室（压感 → 大小/不透明度）里微调。
要按文件真实取值，需要配对样本（见下节）把 protobuf 字段和画世界Pro 的设置项对应起来。

---

## 📦 需要的样本（决定项目能否完成）

`.bru` 没有任何公开逆向资料（百度网盘/付费商店资源都需要登录或购买，无法自动抓取）。
protobuf 元数据块的**结构**已经解出，缺的是**字段语义**——即哪个字段是"压感-大小"、
哪个是"间距"。要解锁 `.bru` 参数解析与写回，**请你提供以下任意一种**：

1. **（最省事）画世界Pro 里的笔刷设置截图**：打开「回忆里绕圈」这支笔的编辑页，
   把 大小 / 不透明度 / 流量 / 间距 / 压感（大小、不透明度）/ 抖动 / 散布 的数值拍下来。
   这些数字能直接和已解出的 protobuf 字段值对上（例如笔尖图块里有 `f33=39`、`f78=0.45`
   这样的候选值），一次截图就能锁定映射。
2. **成对样本**：同一个笔刷，既有 `.bru`，又有它在画世界Pro 里「导出 ABR」得到的 `.abr`。
   `.abr` 是 Adobe 公开规范，对比两者能反推出参数映射，大幅降低逆向难度。
3. **3–5 个不同类型的 `.bru`**（硬边勾线 / 带纹理 / 强压感 / 水彩喷枪），
   用于交叉验证字段语义（同一个字段在不同笔刷间的变化规律）。

> 拿到样本后我会运行 `tools/bru_analyzer.py` 做二进制分析，确定 `.bru` 是
> ZIP / 自定义容器 / 加密，以及图片与参数如何存储，然后实现读写器。

---

## 🧱 技术架构

```
index.html              入口页面（拖拽 / 目标格式选择 / 预览 / 两步导入）
css/style.css          样式
js/bplist.js           Apple binary plist 字节解析
js/nskeyed.js          NSKeyedArchiver 对象图重建
js/schema.js           通用笔刷中间格式 (UniversalBrush)
js/png-alpha.js        PNG → 8 位 alpha 蒙版（浏览器 canvas / Node zlib 双路径）
js/abr-writer.js       Adobe ABR v6.2 编码器（samp/desc/phry + Action Descriptor）
js/procreate.js        Procreate .brush/.brushset 读取 + 字段映射 + 笔尖图多级查找
js/procreate-writer.js Procreate .brush 写回器
js/bru.js              画世界Pro .bru 读取器（写出未实现）
js/app.js              UI 逻辑 / 格式分流 / 预览 / 导出 / 文件结构诊断面板 / .bru 压感开关
tools/bru_analyzer.py  .bru 二进制逆向分析脚本（标准库，无依赖）
tools/bru_meta_dump.py .bru 元数据块逐块 hexdump + protobuf 解析
tools/bru_meta_table.py .bru 元数据块字段横向对比表（常量 vs 变量字段）
tools/diag_brushset.js 笔刷组 ZIP 取证工具（条目/大小/键名一览）
```

转换流程：

```
Procreate(.brush/.brushset) ──parse──▶ UniversalBrush ──write──▶ Adobe(.abr)   ← 默认
画世界Pro(.bru)             ──parse──▶ UniversalBrush ──write──▶ Adobe(.abr)
Procreate(.brush/.brushset) ──parse──▶ UniversalBrush ──write──▶ Procreate(.brush)
```

---

## ▶️ 本地预览

```bash
cd /workspace
python3 -m http.server 8080
# 浏览器打开 http://localhost:8080
```

拖入一个 Procreate `.brush` / `.brushset` 即可看到笔刷预览、笔尖/纹理图与全部参数。

### 构建单文件成品

`index.html` + `css/` + `js/` 是开发版（多文件）；交付用的单文件由打包脚本生成：

```bash
python3 tools/bundle_single_file.py        # 生成「笔刷转换器.html」（约 15MB，含内嵌形状库）
cp 笔刷转换器.html dist/index.html         # 同步部署目录（GitHub Pages / 本地 3000 端口预览用）
```

`dist/` 与成品 HTML **不单独提交来源**，由上述命令随时重建（源码包里同样不含这两个产物）。

---

## 🧪 测试

```bash
node tools/test_procreate.js        # Procreate 解析
node tools/test_procreate_writer.js # .brush 写回往返（19 项）
node tools/test_bplist_roundtrip.js # bplist 编解码自校验
node tools/test_pipeline.js         # .bru → .brush 字节级无损
node tools/test_ui_flow.js          # 模拟 UI 两条转换分支
node tools/test_brushset_layouts.js # 笔刷组 4 种 ZIP 布局（含分享导出 Sub01…）
node tools/test_brushset_plist.js  # brushset.plist 驱动笔刷清单 + 内置笔尖缺失兜底（25 项）
node tools/test_abr_writer.js       # ABR 与真实 Photoshop 文件逐字节对比（28 项）
node tools/test_abr_multi.js        # 多笔刷打包 + 真实笔尖 + 动力学映射
node tools/test_abr_e2e.js          # 真实 .bru / .brush → .abr 端到端
node tools/test_tip_fallback.js     # 笔尖图 7 级查找 + 负例（36 项）
node tools/test_bru_pressure.js     # .bru 压感修复：ABR/.brush 动力学写入 + 关闭分支（19 项）
node tools/test_shape_inverted.js   # 笔尖极性 shapeInverted：反相/不反相/alpha 三例 + 真实蕾丝包（30 项）
node tools/test_shape_lib_e2e.js    # 内置形状库：292 条目 / 1624 键解析、雨后清明 7/7 命中
node tools/test_grain_e2e.js        # 内置纹理（Grain）链路：bundledGrainPath + 5.2+ 四通道（26 项）
```

`tools/test_abr_writer.js` 用真实 Photoshop 文件（`dual.abr`）做基准：除样本 UUID 与
VMAL 中的取样矩形外，**整个 `samp` 段与 Photoshop 输出逐字节一致**。
`tools/test_tip_fallback.js` 的夹具由 `tools/make_tip_fixtures.py` 生成（每支笔刷的
笔尖是不同的纯灰 PNG，可验证匹配到的是**正确**那张，而不是随便一张图）。
`tools/test_bru_pressure.js` 断言 `.bru` 来源默认写出 `szVr.bVTy=2`（钢笔压力）、
`jitter=65`、`Mnm=15`，且关闭开关时退回全 0（与修复前逐字节一致的行为）。
`.bru` 元数据块的 protobuf 解包可用 `python3 tools/bru_meta_table.py <文件>` 复现。

浏览器端到端（playwright + chromium，会真的拖文件、点按钮、下载 `.abr` 并用
Rust 解析器回读）：

```bash
python3 tools/browser_e2e.py
```

浏览器 E2E 覆盖「两步导入」新流程：点「转换」后按钮须变为「导入 …」态、出现「仅下载文件」
次级按钮、点导入在无分享面板的无头环境下正确降级为下载（`convert_and_download` helper）。

线上链接端到端（playwright + chromium，打开已发布 URL 复验）：

```bash
python3 tools/test_live_e2e.py
```

`tools/test_brushset_plist.js` 复刻了用户上报的「雨后清明@阿昔.zip」结构：
`brushset.plist` 列 7 支笔刷、每个 `<uuid>/` 下另有 `Reset/` 与 `Sub01/` 母版档案、
且全包没有任何笔尖图。它断言笔刷数是 7 而不是 14、组名来自 plist、每支笔刷都报出
缺失的内置笔尖文件名、ABR 能正常生成，并且（若该上传文件在本机）直接拿原文件复验。

`tools/test_shape_inverted.js` 复刻了「呼呼｜蕾丝1.0.zip」的极性结构：白场+暗环
（`shapeInverted=true`）、黑场+白环（`false`）、带真实 alpha 通道的 RGBA（`true`）
三支笔刷，断言反相后的平面裁剪到图案外框、环为墨、环心透明，再从生成的 ABR 里
把 `samp` 样本解回来逐项比对（顺带覆盖行长度表 + PackBits 布局），并验证写回
`.brush` 时旗标不丢；若蕾丝包在本机，直接复验 59 支、58 反相、无 2048² 笔尖。

---

## 📐 关于「完美转换」的现实

两套笔刷引擎完全不同（Procreate 的 Valkyrie vs 画世界Pro 自研引擎），因此：

- **可高质量迁移**：笔尖形状图、纹理图、间距、流量、压感曲线、圆度、散布等核心资产。
- **无法 1:1 还原**：引擎专属特效（如 Procreate 的 Wet Mix 湿混、特定渲染模式）只能用
  近似参数尽量逼近，做不到像素级一致。

目标定位是「高度可用 + 核心笔触一致」，而非字面意义的「完美」。
