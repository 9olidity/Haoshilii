# haoshili（好视力）——在电脑屏幕上模拟“红色聚焦”近视色差离焦

本项目是一个 macOS 桌面工具：实时捕获屏幕内容，并在自己的窗口中用数字滤镜模拟论文所述的“红色聚焦（myopic LCA / red-focused）”刺激：**红色通道保持清晰，绿色与蓝色通道分别按不同强度模糊**。

> 论文背景：*Pilot study: Simulating myopic chromatic aberration on computer screens induces progressive thickening of the choroid in myopic eyes*（Swiatczak et al., 2024）
https://open.lnu.se/index.php/sjovs/article/view/4232


## 这和论文的关系（我们实现了什么）

论文中的“红色聚焦”数字滤镜核心是：
- **R（红）保持清晰**
- **G（绿）与 B（蓝）进行低通/模糊**

本项目当前实现：
- 在 GPU 上合成输出 `R=原图R`、`G=绿模糊G`、`B=蓝模糊B`（见 `Shaders.metal`）
- 绿/蓝 **分别做一次高斯模糊**（两次 blur），以更贴近“按波长校准模糊量”的思路（见 `CaptureEngine.swift`）
- 参数固定，不提供 UI slider（论文实验是固定刺激条件；本项目同样默认固定参数）

本项目**不负责**论文中的实验流程部分（例如固定观看距离、每天 2 小时、连续 12 天、恢复期、亮度匹配、AL/ChT 测量），这些需要使用者自行按 protocol 执行。

## 工作原理（代码级）

1. **屏幕捕获**：使用 `ScreenCaptureKit` 捕获主显示器（`SCStream`）。
2. **窗口裁切**：根据窗口在屏幕上的位置（物理像素偏移）从整屏纹理里裁出“窗口覆盖区域”。
3. **双模糊**：对裁切结果分别生成：
   - `greenBlurTexture`：sigma = `FilterPreset.sigmaGreen`
   - `blueBlurTexture`：sigma = `FilterPreset.sigmaBlue`
4. **通道合成**：Metal compute shader 输出：
   - `R` 来自清晰原图
   - `G` 来自绿模糊
   - `B` 来自蓝模糊

相关文件：
- `CaptureEngine.swift`：捕获、裁切、双模糊、渲染调度
- `Shaders.metal`：通道合成（`compositeChannels`）
- `haoshiliApp.swift`：窗口置顶/穿透、UI、会话时长展示

## 固定参数（可改）

当前固定参数在 `CaptureEngine.swift`：
- `CaptureEngine.FilterPreset.sigmaGreen`
- `CaptureEngine.FilterPreset.sigmaBlue`

你可以按需要调整两个 sigma 来改变绿/蓝模糊强度（一般期望 **蓝 > 绿**）。

## 使用方式


### 模式
- **非穿透模式**：窗口可操作、可调整大小；边框有可见的 resize chrome 便于定位和缩放。
- **穿透模式（开始工作）**：窗口忽略鼠标事件（click-through），用于“覆盖显示但不挡操作”。
  - 快捷键：`Cmd+T` 进入/退出穿透模式
  - 退出穿透模式后会在面板显示“本次穿透持续时间”

## 重要说明 / 已知差异

- 论文提到的“按 6.5mm 瞳孔、基于 LCA 函数校准的低通滤波”在本文是用 **高斯模糊 sigma** 近似表达；目前 sigma 值是工程默认值，未做严格光学标定。
- 本项目是显示层面的“刺激呈现工具”，不等同于医学器械，也不提供任何诊断/治疗建议。

## 隐私

屏幕捕获与滤镜处理均在本机完成；项目不包含网络上传逻辑。为避免“无限递归捕获自身窗口”，应用会在捕获滤镜中排除自身的覆盖窗口。

