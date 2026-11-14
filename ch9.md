# 第9章 第 9 段：着色器与图形管线

**一句话**：一批顶点从 `VBO` 出发，经过顶点着色器、`图元装配`、`光栅化`、`裁剪`、片元着色器与`混合`，才变成`颜色缓冲`里的像素——每一站都可能成为瓶颈。

**为什么需要它**：你换了一个看起来更真实的`光照模型`，帧时间立刻翻倍；或者透明物体叠在一起，后面的东西画到了前面。前者的成本落在`片元着色器`，后者落在`深度测试`与`混合`的顺序。不知道管线分几段，就只能靠调参数碰运气。LearnOpenGL 官方把这条链列得很清楚，它也是判断「该优化顶点还是片元」的依据。

**核心结构**：官方阶段表是 `Vertex Data`（用 `VBO` / `VAO` / `EBO` 与`顶点属性`管理）→ `primitives` 图元提示（`GL_TRIANGLES` 等）→ `顶点着色器`（输出到 `gl_Position`）→ 可选的`几何着色器` → `图元装配`（`primitive assembly`）→ `光栅化`（`rasterization stage`，产出 `fragments`）→ `裁剪`（`clipping`，丢弃视图外片元）→ `片元着色器`（算最终颜色）→ `alpha test and blending stage`（检查`深度测试`与`模板测试`、判断前后、检查 alpha）。另有可选的曲面细分与 transform feedback。

**关键设计**：现代 OpenGL **必须**自己定义至少一个 `vertex shader` 与一个 `fragment shader`，用 `GLSL` 编写，GPU 上没有默认顶点/片元着色器。`NDC`（`归一化设备坐标`）规定 x/y/z 都在 -1.0 到 1.0 之间才可见，范围外丢弃/裁剪；NDC 再经 `glViewport` 提供的`视口变换`转成`屏幕空间`坐标。`VBO` 把大批顶点一次送进显存（官方指出 CPU→GPU 相对慢），`glBufferData` 的 `GL_STATIC_DRAW` / `GL_DYNAMIC_DRAW` / `GL_STREAM_DRAW` 声明数据改动频率；`VAO` 保存属性配置与关联的 VBO，核心模式要求绑定 VAO 否则很可能拒绝绘制；`EBO` 存索引，用`索引绘制`只存唯一顶点。

**拆解**：
### 顶点数据怎么被解释
`glVertexAttribPointer` 指定 location、size、type、normalize、`stride` 与 `offset`，再用 `glEnableVertexAttribArray` 启用（顶点属性默认禁用）。`VAO` 把这些调用记下来，之后每帧只要绑定对应 VAO 就能复用整套配置。官方给出的绘制序列是 `glUseProgram` → `glBindVertexArray` → `glDrawArrays` 或 `glDrawElements`。
### 着色器怎么接起来
每个着色器以版本声明开头，顶点属性用 `in` 声明，输出赋给预定义的 `gl_Position`；片元着色器用 `out` 输出一个 `vec4`（RGBA，各分量 0.0～1.0，alpha 是不透明度）。编译用 `glCreateShader` / `glShaderSource` / `glCompileShader`，链接用 `glCreateProgram` / `glAttachShader` / `glLinkProgram`；链接时把上一阶段输出接到下一阶段输入，不匹配就会报链接错误。
### 从 NDC 到像素
顶点着色器输出的坐标应在 `NDC` 范围内，超出的被`裁剪`；经`视口变换`到`屏幕空间`后，`光栅化`把图元映射成片元；片元着色器算最终颜色；`alpha test and blending` 再检查深度与模板值、决定片元在其它物体前还是后、并据此丢弃或`混合`。综合报告口径下，`overdraw` 之所以贵，正因为片元着色器常常已经算完，深度判断才把它丢掉。
### 瓶颈在哪一站
顶点少而屏幕覆盖大（比如全屏后处理）时，成本多在`片元着色器`；几何极密但覆盖小，成本多在顶点与`图元装配`；`纹理`与`材质`频繁切换则卡在`渲染状态`。综合报告口径下，先判断是哪一站饱和，再决定降几何、降分辨率还是合批。

**今天怎么做**：拿一个掉帧的画面，分别试着降低几何密度、降低渲染分辨率、减少材质切换，看哪一项真正改善`帧时间`；三项分别对应顶点段、片元段与状态切换段，比盲目调 shader 有效得多。

<div class="chapend" markdown="1">

**容易误解的地方**：以为 `VBO` 里的坐标就是屏幕坐标，忽略中间还有 `NDC` 与`视口变换`；以为`裁剪`发生在光栅化之前（官方把它列在片元着色器之前、但丢弃的是片元）；以为`深度测试`能省下被遮挡片元的着色成本，忽略很多情况下着色已经算完。

**记住这一句**：先弄清卡在管线的哪一站，再谈优化哪一行的 shader。

</div>
