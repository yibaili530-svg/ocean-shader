# 一次性提示词：ocean-shader（黄金时刻写实海洋，单 HTML，WebGL2）

本文件是一份可以一次性复现本仓库 `index.html` 的完整提示词。其中的数值都取自当前代码。

---

请构建一个文件 `index.html`：一片无限延伸的黄金时刻海洋，深邃清澈、以蓝色为主，金色的太阳碎光柔和，天空有成片的高积云。先完整读完本规格再写代码。第 0 节是技术契约，优先级最高。可调数值都写成 `COMMON` 顶部的具名 `const`，方便统一调整。

## 0. 技术契约
- 单个自包含 HTML 文件，原生 WebGL2 + GLSL ES 3.00，所有噪声与颜色都在代码中程序化生成，文件离线即可运行。
- 管线依次为：
  1. **天空全景缓存** `panoramaRT`：2048×512 RGBA16F，等距柱状投影，仰角方向按幂 `PANORAMA_ELEVATION_POWER = 2.0` 映射，让行数集中在地平线附近。每帧更新 8 行，循环刷新，第一次完整生成后标记为就绪。
  2. **场景 pass** `sceneRT`：RGBA16F，尺寸为画布 × `renderScale`。一个全屏片元 shader 完成海面光线步进、水面着色和水下渲染，天空像素直接采样全景缓存。
  3. **Bloom**：从场景依次 5 级半分辨率下采样，再用 3×3 tent（权重 1-2-1）逐级上采样叠加。
  4. **合成**：`(scene + bloom × 0.055) × EXPOSURE(1.05)` → ACES fitted（Stephen Hill 矩阵）→ 暗角 0.25 → 颗粒 0.02（按亮度开方缩放）→ sRGB 编码 → 三角抖动 1/255。
- 浮点渲染目标用 RGBA16F（`EXT_color_buffer_float`）。扩展缺失时切换到 RGBA8 + RGBM 编码（`exposed = col/64`，alpha 存倍率），画面保持一致。
- 单位：米、秒、弧度；+Y 向上；海平面 y = 0。太阳方向 `SUN_DIR = (0, 0.069756474, -0.99756405)`，即仰角 4°（`SUN_ELEVATION = 0.06981317008`），方位在相机默认朝向 −Z。
- 确定性：所有动画只读一个 uniform `uTime`。支持 URL 参数 `?t=`（冻结时间，用于截图）、`?scale= ?x= ?y= ?z= ?yaw= ?pitch=`。默认相机 x=0, y=2.5, z=0, yaw=0, pitch=0.03。
- 清晰度：
  - `renderScale` 默认 1.0，手动模式；`devicePixelRatio` 取 `min(dpr, 1)`。
  - `[` / `]` 以 0.05 步长在 0.25–1.0 之间调整，并切回手动模式。
  - `R` 切换自动分辨率：每 0.5 s 统计一次，fps < 54 时降 0.05，fps > 61 时升 0.03，范围 0.35–1.0。
- 健壮性：
  - 每个 shader 检查 `getShaderInfoLog` / `getProgramInfoLog`，出错时显示在红色屏幕框中并 `console.error`，正常运行时控制台保持干净。
  - 数值都保持有限：`pow`/`sqrt`/`normalize` 之前用 `max(…, 1e-4)`，归一化统一走 `safeNorm`。
  - 处理 `webglcontextlost` / `webglcontextrestored`：恢复后重建 program、纹理、渲染目标，然后继续渲染。
  - 渲染循环复用预分配的数组和 `Float32Array`；渲染目标只在尺寸变化时重建。
- 暴露 `window.__ocean = { fps, altitude, renderScale, frames, hdr }` 和全局 `waterHeight(x, z, t)`，供自动检查使用。

## 1. 波浪（几何）——10 个 Gerstner 波
- 主波长 120 m，后续每个分量的波长乘以 `WAVE_RATIO = 0.6344941742`（120 m → 约 1.7 m）。
- 参数表：
  - `AMPLITUDES = (0.60, 0.35, 0.22, 0.16, 0.115, 0.078, 0.053, 0.035, 0.024, 0.016)` m
  - `ANGLES = (0.06, -0.32, 0.49, -0.68, 0.90, -0.16, 0.65, -1.02, 0.27, -0.52)` rad，方向向量 `dir = (sin a, cos a)`
  - `PHASES = (0.4, 2.1, 4.8, 1.3, 5.5, 3.2, 0.7, 4.2, 2.7, 5.9)`
  - `STEEPNESS = (0.16, 0.15, 0.14, 0.13, 0.12, 0.11, 0.10, 0.09, 0.08, 0.07)`
- 相位 `k·dot(dir, q) − sqrt(g·k)·t + phase`，其中 g = 9.81，深水色散。
- 欧拉点高度：用 4 次 Newton 迭代反解水平 Gerstner 位移（每步修正量限制在 ±8 m）。
- 按像素足迹淡出：`waveFade(λ, fp) = smoothstep(2fp, 4fp, λ)`；被淡掉的斜率方差累计到 `lost`，计入粗糙度。
- 法线细节：2 层梯度噪声涟漪（波长 1.5 m 起，强度 0.075 和 0.045），各自随时间平移，只影响法线。
- 海面求交：最多 32 步（`WATER_MARCH_STEPS = 32`）步进后再二分细化。每条朝下的视线都着色为水面，步数用完时取最后的位置。
- CPU 端 `waterHeight(x, z, t)` 与 GLSL 使用完全相同的 10 个波和同样的 Newton 反解，用于相机贴水和 HUD 高度。

## 2. 水面材质（移植自 ABYSSAL 的海水着色）
- **粗糙度**：
  - Cox–Munk 均方斜率 `mss = 0.003 + 0.00512 × 7.0`（风速 7 m/s）。
  - 像素足迹 `fp = max(dist × pixelAngle, |dFdx(p.xz)|, |dFdy(p.xz)|)`；按 `share = (0.06, 0.30, 0.64)` 分配给波长 2.0 m / 1.5 m / 0.3 m 三档，`unresolved = dot(share, 1 − resolved²)`。
  - `mssUnresolved = mss × max(unresolved, 0.10) + 0.0009`；`alpha = clamp(sqrt(2·mssUnresolved), 0.012, 0.62)`；`roughness = clamp(sqrt(alpha), 0.02, 0.86)`。
  - 200–1000 m 距离内，法线向 (0,1,0) 平滑混合 30%。
- **Fresnel** `fresnelWater(NoV, roughness)`：`f = 0.02 + 0.98·(1−NoV)^5`，返回 `mix(f, clamp(f·0.72 + 0.06, 0, 1), clamp(roughness·1.5, 0, 1))`。
- **天空反射**：
  - 采样全景缓存，模糊宽度匹配 GGX 波瓣（由 alpha 决定），再乘以 (0.59, 0.64, 0.66)。
  - 反射方向朝下时向上弯折：`R = normalize(R.x, mix(0.02, 0.35, roughness) − R.y·0.15, R.z)`。
- **天空辐照度**：余弦加权采样全景缓存，天顶 × 0.4142 + 两个 45° 方向 × 0.2929，再乘以 `BODY_SKY_BALANCE = (3.2, 2.20, 0.83)` 得到 `bodySky`。
- **水体颜色**：
  - `waterScatter = (0.010, 0.045, 0.092)`，`waterAbsorb = (0.003, 0.016, 0.036)`。
  - `sun = SUN_COLOR(1.0, 0.66, 0.34) × SUN_IRRADIANCE(3.0)`，`sunUp = max(SUN_DIR.y, 0)`。
  - `body = scatter·(sun·sunUp·(1 − fresnelWater(sunUp, 0))/π + bodySky·0.94) + absorb·bodySky·0.8`。
- **逆光波峰辉光**：
  - `heightNorm = clamp(h·0.35 + 0.35, 0, 1.5)`。
  - `backlit = heightNorm·pow(sat(dot(L, −V)), 4)·pow(0.5 − 0.5·dot(L, N), 3)`。
  - `body += scatter·sun·backlit·3.4 / (1 + max(0, −L.y)·4)`。
  - 最后 `body *= BODY_GAIN (3.8)`。
- **太阳碎光**：
  - GGX，alpha 按太阳圆盘加宽：`aP = clamp(alpha + 0.00465/2, 0, 1)`，能量归一 `alpha²/aP²`。
  - Smith-GGX correlated 可见性；VoH 上用 Schlick 0.02；再乘 NoL。
  - `specHDR = SUN_IRRADIANCE·glint·0.5`，软上限 `SPEC_MAX = 8.0`：`spec = SUN_COLOR²·(8·specHDR / (8 + specHDR))`，碎光带因此呈柔和的金色。
- **合成**：`color = mix(body, reflection, fresnelWater(NoV, roughness)) + spec`。
- **白浪**（稀疏、柔和）：
  - 种子：`breaker = (1 − smoothstep(FOAM_THRESHOLD − 0.13, FOAM_THRESHOLD, J))·smoothstep(0.12, 0.65, h)`，`FOAM_THRESHOLD = 0.78`，J 为雅可比。
  - 用风向坐标系（风向角 0.06 rad）采样两层噪声，坐标拉伸 (0.22, 1.0)×0.8、随时间平移 0.12 m/s，形成顺风的长条泡沫纹：`raft = sat(0.5 + n)`，`fine = sat(0.5 + n(×2.03 + 7))`。
  - `carved = breaker·(0.10 + (raft·0.65 + fine·0.35)·1.55)`，onset 0.80，`foam = smoothstep(0.80, 1.10, carved)·mix(0.35, 1, fine)`。
  - 距离窗口：12–24 m 渐入，140–260 m 渐出；按足迹 `waveFade(3.5, fp)` 淡出。
  - 泡沫颜色 `(0.93, 0.96, 0.985)·mix(0.62, 1, fine)`，光照 `0.28 + sun·wrapNoL·0.035 + bodySky·0.20`，其中 `wrapNoL = sat((N·L + 0.45)/1.45)`。
  - 另有 `foamThin`（onset × 0.85）以 20% 的强度覆盖为半透明薄膜。
- **空气透视**：按 `1 − exp(−dist·0.00012)` 向地平线天空颜色混合；输出上限 60。

## 3. 天空
- 解析晴空 `clearSky`，基色为：天顶 `ZENITH = (0.030, 0.095, 0.420)`，朝太阳的地平线 `HORIZON_SUN = (1.0, 0.62, 0.32)`，背向太阳的地平线 `HORIZON_AWAY = (0.48, 0.32, 0.47)`，按仰角和朝太阳程度 `pow(…, 4)` 混合。
- 太阳圆盘角半径 `SUN_RADIUS = 0.00471238898`，辐亮度 `SUN_RADIANCE = 60`，并带 HG Mie 光晕。太阳圆盘直接在主 pass 中解析绘制，保持锐利。

## 4. 云（只在天空全景缓存中步进）
- 云层 `CLOUD_BOTTOM = 1500` m 到 `CLOUD_TOP = 3000` m，随时间沿 −Z 以 `CLOUD_WIND = 3` m/s 漂移。
- 密度：
  - 噪声坐标为 `q = (p − wind·t)·(0.00115, 0.0018, 0.00115)`。
  - `shape = 0.56 + perlinWorleyFBM(q) − CLOUD_COVERAGE(0.71)`，密度为 `smoothstep(0, 0.12, shape)`。
  - 垂直剖面 `smoothstep(0, 0.15, h)·(1 − smoothstep(0.48, 1, h))`。
- 步进：每条视线最多 22 km。主步数随仰角从 14 增加到 32，取每步中点、不加抖动，结果平滑无颗粒。
- 光照：
  - 向太阳方向 3 步、每步 350 m 的光照采样；消光 `CLOUD_EXTINCTION = 0.0045`。
  - 用 Beer × Powder `2(1 − e^{−2d})` 计算亮度；双瓣 HG（0.8·HG(0.6) + 0.2·HG(−0.2)）。
  - 环境光 (0.35, 0.42, 0.55)，从云底到云顶亮度从 0.65 倍增加到 1.15 倍。
  - `lighting = ambient + SUN_COLOR·(beer·(0.55 + phase·5)·powder + beer³·phase·0.7)`。
- 地平线过渡：仰角在 `CLOUD_HAZE_START = 0.025` 到 `CLOUD_HAZE_END = 0.105` 之间时，云平滑融入晴空；远处的云按 `1 − exp(−start·0.000018)` 融入霾中。

## 5. 水下（相机低于水面 0.15 m 以下）
- 深度雾：
  - 视线深度 `viewDepth = depth + 30·(1 − smoothstep(0.25, 0.80, rd.y))`。
  - 雾色从 (0.05, 0.22, 0.20) 过渡到 `DEEP_WATER = (0.0, 0.02, 0.05)`（viewDepth 达到 30 m 时）。
  - 基础亮度为雾色 × `mix(0.11, 0.85, sat(rd.y·0.6 + 0.4))`。
- 抬头看水面：
  - Snell 窗的临界角由 n = 1.333 计算，窗口边缘用 ±0.015 平滑过渡。
  - 窗内显示折射后的全景天空，窗外为全反射：`fog·(0.25 + 0.6·tir.y) + (0.004, 0.012, 0.025)`。
  - 按距离逐通道吸收，`ABSORPTION = (0.45, 0.09, 0.05)`。
- 光柱：沿视线 10 步（`GOD_RAY_STEPS = 10`）、带哈希抖动，最远 65 m；在折射后的太阳方向上采样两层扭曲 Voronoi 焦散，HG g = 0.8；只在 rd.y 介于 0.55–0.80 及以上的视线上显示。
- 向下看时颜色趋向 `DEEP_WATER`。在水面上下 ±0.15 m 内，水上与水下画面平滑混合。

## 6. 相机、操作、界面
- 第一人称相机：鼠标拖动观察（双击锁定指针，Esc 释放）；WASD 移动；Space / Q 上升 / 下潜；Shift ×5；滚轮在 0.5–50 m/s 之间调节速度（默认 4）。距离水面 1 m 以内时，相机随波面平滑起伏。
- HUD：12px 白色等宽字体，70% 不透明度。左上显示 FPS、离水高度、`Scale 1.00 manual` 或 `auto`、内部分辨率；左下显示一两行操作说明。

## 7. 完成标准
用 headless Chrome（SwiftShader 即可）在 1280×720、`?scale=1&t=12` 下渲染三张图：`above.png`（默认视角）、`down.png`（`&pitch=-0.8`）、`under.png`（`&y=-4&pitch=0.7`）。逐项确认：
1. 控制台干净，shader 编译和链接日志为空。
2. `above.png` 下方 35% 区域：B > G > R，G/B < 0.85，平均亮度 0.08–0.35（目标约 sRGB 0.13 / 0.23 / 0.37）。
3. 顶部 15%（天顶）比地平线上方色带更暗、更蓝；地平线色带 R > B。
4. 最亮像素位于画面中心列 ±5% 宽度以内（太阳与碎光）。
5. `under.png` 下方三分之一的平均亮度低于上方三分之一的 0.5 倍。
6. 地平线附近每条水平色带的方差都在相邻色带的 3 倍以内。
7. 近白水面像素（亮度 > 0.8 且饱和度 < 0.15）占比约 0.05%，碎光呈柔和的金色；`down.png` 平均颜色约 sRGB 0.04 / 0.17 / 0.33。

## 8. 文件内代码布局
依次为：`<style>`；canvas 与 UI div；shader 字符串（`COMMON`、`PANORAMA_FS`、`SCENE_FS`、`BLOOM_DOWN_FS`、`BLOOM_UP_FS`、`COMPOSITE_FS`）；JS（setup、URL 参数、输入、`waterHeight`、资源与 context 恢复、渲染循环）。
