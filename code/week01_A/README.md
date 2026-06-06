# week01_A — CMake 工程（Windows / MSVC / VS2022+）

第 1 周代码验证闭环的 Windows 工程版。把 AI 生成的几何函数粘到 `test_*.cpp` 的 PASTE ZONE，
F5 即可运行测试、生成 SVG 可视化。

---

## 目录结构

```
week01_A/
├── CMakeLists.txt       ← CMake 工程定义
├── geo.h                ← 公共头：Point3D + SVG writer（不要改）
├── test_bezier.cpp      ← Bezier 求值测试 harness
├── test_bspline.cpp     ← B 样条基函数测试 harness
├── test_nurbs.cpp       ← NURBS 求值测试 harness
├── README.md            ← 本文档
└── .gitignore
```

---

## 环境要求

- **Visual Studio 2022 (17.x)** 或 **Visual Studio 2026 (18.x)**
- 安装时勾选 **"使用 C++ 的桌面开发"** 工作负载（包含 MSVC 编译器 + CMake + Ninja）
- 不需要安装任何第三方库（不需要 Eigen / Boost / Qt）

---

## 方法 A：直接在 Visual Studio 里打开（推荐）

VS 原生支持 CMake，最省事的方式：

1. 启动 Visual Studio
2. 选 **"打开本地文件夹"** → 选 `week01_A` 文件夹
3. VS 自动检测 `CMakeLists.txt` 并配置工程（首次约 10 秒）
4. 顶部工具栏的"启动项"下拉框选 **`test_bezier.exe`**
5. 按 **F5**（调试运行）或 **Ctrl+F5**（直接运行）
6. 控制台显示测试结果；SVG 文件生成在 `out/build/<preset>/bin/` 下

切换运行 `test_bspline` 或 `test_nurbs`：只需改顶部的启动项下拉框。

---

## 方法 B：命令行构建（适合脚本化）

打开 **"x64 Native Tools Command Prompt for VS 2022"**（开始菜单里找），
cd 到 `week01_A` 目录，执行：

```cmd
cmake -S . -B build
cmake --build build --config Release
```

构建产物：
```
build\bin\Release\test_bezier.exe
build\bin\Release\test_bspline.exe
build\bin\Release\test_nurbs.exe
```

运行：
```cmd
cd build\bin\Release
test_bezier.exe
test_bspline.exe
test_nurbs.exe
```

SVG 输出会在当前目录（即 `build\bin\Release\`）。

### 显式指定生成器

CMake 默认会用最新的 VS。如需显式指定：

```cmd
:: VS2022
cmake -S . -B build -G "Visual Studio 17 2022" -A x64

:: VS2026（如已安装）
cmake -S . -B build -G "Visual Studio 18 2026" -A x64

:: 或者用 Ninja（更快）
cmake -S . -B build -G "Ninja"
```

---

## 方法 C：VSCode + clangd（推荐用于日常写代码）

### 装两个扩展

| 扩展 | id | 作用 |
|---|---|---|
| CMake Tools | `ms-vscode.cmake-tools` | 配置 / 构建 / 运行 / 调试 |
| clangd | `llvm-vs-code-extensions.vscode-clangd` | C++ 智能提示（补全、跳转、查找引用、refactor） |

**注意**：装 clangd 时如果 VSCode 提示"检测到 Microsoft C/C++ 扩展，是否禁用 IntelliSense？"——**点禁用**。两者同时开会打架。如果之前装过 cpptools，到扩展设置里把 `C_Cpp.intelliSenseEngine` 设为 `disabled`。

### 配置 CMake Tools 用 Ninja 生成器（重要）

`compile_commands.json` 只在 Ninja / Makefile 生成器下产出，Visual Studio 生成器不产出。

- **Mac / Linux**：CMake Tools 默认就走 Unix Makefiles 或 Ninja，无需配置
- **Windows**：打开 `.vscode/settings.json`（没有就新建），加：
  ```json
  {
    "cmake.generator": "Ninja"
  }
  ```
  Ninja 在装 Visual Studio "使用 C++ 的桌面开发"工作负载时已经一起装好

### 启动流程

1. `File → Open Folder` 选 `week01_A`
2. 状态栏底部点 **"No Kit Selected"** 选编译器：
   - Mac：AppleClang
   - Linux：GCC 或 Clang
   - Windows：`Visual Studio Community 2022 Release - amd64`（或 2026）
3. 按 **F7** 构建。第一次构建后 `build/compile_commands.json` 生成，clangd 自动加载
4. 打开任一 .cpp，几秒后看到状态栏 clangd 提示 "indexed" — 智能提示就绪
5. 状态栏选 launch target（`[test_bezier]` / `[test_bspline]` / `[test_nurbs]`），**Shift+F5** 跑，**Ctrl+F5** 调试
6. SVG 文件落在 `build/bin/`（Mac/Linux）或 `build/bin/<Config>/`（Windows MSVC），拖进浏览器看

### 验证 clangd 工作

- 把鼠标移到 `Point3D` 上 → 应弹出定义和成员
- 在函数名上按 **F12** → 跳到定义
- 故意写 `Point3D p; p.w = 1.0;` → `w` 下应该出现红波浪线（Point3D 没有 w 字段）

如果完全没提示：clangd 没找到 compile_commands.json。检查 `build/compile_commands.json` 是否存在；不存在说明用了 VS 生成器，回去做"配置 Ninja"那步。

---

## 工作流：把 AI 代码接入

以 Bezier 为例：

1. 在 Gemini 上跑你的 prompt，拿到 `bezier_cubic` 的 C++ 实现
2. 打开 `test_bezier.cpp`，找到这两行注释中间的区域：
   ```cpp
   // >>>>>>>>>>  PASTE AI-GENERATED bezier_cubic BELOW  <<<<<<<<<<
   ...占位实现...
   // >>>>>>>>>>             END OF PASTE ZONE              <<<<<<<<<<
   ```
3. 用 AI 的实现**整体替换**占位实现（保留函数签名一致）
4. **保存 → Ctrl+F5 运行**
5. 控制台看 PASS/FAIL；浏览器拖入 SVG 看曲线形态

### 三个对照实验工作流

第 1 周周一-周二要做三段 prompt 的对照。建议：

```
week01_A/
├── test_bezier.cpp          ← 当前正在测试的版本
├── ai_versions/
│   ├── bezier_A.cpp.txt     ← prompt A 的 AI 输出（仅函数体）
│   ├── bezier_B.cpp.txt     ← prompt B 的 AI 输出
│   ├── bezier_C.cpp.txt     ← prompt C 的 AI 输出
│   └── notes.md             ← 三版的测试结果对比
```

逐版本粘进 `test_bezier.cpp` 跑一遍，记下数值误差和 SVG 截图。

---

## SVG 查看

生成的 `.svg` 文件直接用任意浏览器打开（拖进 Chrome / Edge 即可）。

或者在 VS Code 装 **"SVG Preview"** 插件，编辑器里直接预览。

VS 内部不预览 SVG，需要外部工具。

---

## 常见问题

| 现象 | 原因 | 解决 |
|---|---|---|
| 控制台中文乱码 | Windows 控制台默认 GBK | `enable_utf8_console()` 已自动调用；若 cmd 仍乱码，先 `chcp 65001` |
| 编译报 `C4819` 警告 | 源文件被识别为非 UTF-8 | CMake 已设 `/utf-8`；若手动 cl 编译需加此标志 |
| 编译报 `M_PI` 未定义 | MSVC `<cmath>` 默认不导出 | CMake 已设 `_USE_MATH_DEFINES`；若手动编译需加 `/D_USE_MATH_DEFINES` |
| `min/max` 报错 | windows.h 定义了同名宏 | CMake 已设 `NOMINMAX`，问题不会出现 |
| F5 时 SVG 不知去哪了 | VS 调试工作目录默认在源码目录 | CMakeLists.txt 已设 `VS_DEBUGGER_WORKING_DIRECTORY` 为 exe 所在目录 |
| 想编译单个 target | | `cmake --build build --target test_bezier --config Release` |
| 想清理 | | 删除 `build/` 文件夹即可 |
| Linux/macOS 上也想跑 | | CMakeLists.txt 已有 GCC/Clang 备用编译选项，直接 `cmake -S . -B build && cmake --build build` |

---

## 编辑约定

- **不要修改 `geo.h`**：整个第 1 周保持不变
- **不要修改 `int main()`**：测试逻辑是金标准，改了就失去客观性
- **只在 PASTE ZONE 内替换函数实现**
- AI 输出的代码里如果重复定义了 `Point3D`，**删掉它的**，用 `geo.h` 的

---

## 验证标准速查

| 测试 | 通过线 | 失败说明 |
|---|---|---|
| Bezier Test 1 | 误差 < 1e-9 | AI 用了 Bernstein 展开且 t 接近 0/1 时精度差 |
| Bezier Test 2 | 严格相等 | 端点处理错（不应做插值，应直接返回控制点） |
| Bezier Test 3 | 严格相等 | 退化处理错 |
| B-spline Test 1 | 误差 < 1e-9 | 递推公式实现错 |
| B-spline Test 2 | 0 violation | 不应该有负值 |
| NURBS Test 1 | 误差 < 1e-6 | 权重或归一化错（不在单位圆上）|
| NURBS Test 2 | 误差 < 1e-6 | 端点插值错 |
