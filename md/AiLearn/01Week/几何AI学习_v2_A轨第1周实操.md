# 第 1 周配套：代码验证闭环（最小工作台）

> 配套：`几何AI学习_v2_A轨日常效率版.md`
> 目的：解决"AI 生成的 C++ 代码到底对不对"——通过**一个零依赖的最小工作台**，把 AI 输出粘进去就能编译、跑测试、看 SVG 可视化。
> **整套工具不超过 200 行 C++，单个 `g++` 命令编译，不需要 CMake、不需要任何库**。

---

## 设计哲学

| 决策 | 理由 |
|---|---|
| 不用 Python plotting / matplotlib | 多一个依赖少一份动力 |
| 不用 OpenGL / Qt | 杀鸡用牛刀 |
| 不用第三方库（Eigen 都不用） | 第 1 周专注 prompt 训练，不混入工具学习 |
| 输出 SVG（不是 PNG） | 浏览器直接打开，矢量缩放清晰，**纯文本可 diff** |
| 单文件单测试 | 每个算法独立编译，不互相干扰 |
| 在终端打印数值 + 同时画图 | 数值供 assertion，图形供直觉判断 |

---

## 一次性 Setup（5 分钟）

打开终端，**复制粘贴一整段**：

```bash
mkdir -p ~/geo-week01-lab && cd ~/geo-week01-lab && echo "Setup ready at $(pwd)"
```

然后照下面把三个文件创建出来（用你顺手的编辑器；下面会贴全文）。

---

## 文件 1：`geo.h`（公共头：Point3D + SVG writer）

这是**唯一的公共依赖**。一份 60 行的头，包含：
- `Point3D` 结构体（带运算符重载）
- `SVG` writer（自动 y 轴上翻，画线、画点、写文字）

```cpp
// geo.h — Week 1 minimal lab utilities
#pragma once
#include <cmath>
#include <fstream>
#include <string>
#include <vector>

struct Point3D {
    double x = 0, y = 0, z = 0;
    Point3D operator+(const Point3D& o) const { return {x+o.x, y+o.y, z+o.z}; }
    Point3D operator-(const Point3D& o) const { return {x-o.x, y-o.y, z-o.z}; }
    Point3D operator*(double s) const { return {x*s, y*s, z*s}; }
};
inline Point3D operator*(double s, const Point3D& p) { return p * s; }

class SVG {
    std::ofstream f;
public:
    SVG(const std::string& path, double x0, double y0, double x1, double y1) {
        f.open(path);
        double w = x1 - x0, h = y1 - y0;
        f << "<svg xmlns='http://www.w3.org/2000/svg' viewBox='"
          << x0 << " " << -y1 << " " << w << " " << h
          << "' width='800' height='600' style='background:white'>\n"
          << "<g transform='matrix(1 0 0 -1 0 0)'>\n";  // flip y axis
    }
    void polyline(const std::vector<Point3D>& pts, const std::string& color, double width) {
        if (pts.empty()) return;
        f << "<polyline points='";
        for (auto& p : pts) f << p.x << "," << p.y << " ";
        f << "' fill='none' stroke='" << color << "' stroke-width='" << width << "'/>\n";
    }
    void dots(const std::vector<Point3D>& pts, const std::string& color, double r) {
        for (auto& p : pts)
            f << "<circle cx='" << p.x << "' cy='" << p.y << "' r='" << r
              << "' fill='" << color << "'/>\n";
    }
    void line(double x0, double y0, double x1, double y1, const std::string& color, double width) {
        f << "<line x1='" << x0 << "' y1='" << y0 << "' x2='" << x1 << "' y2='" << y1
          << "' stroke='" << color << "' stroke-width='" << width << "'/>\n";
    }
    ~SVG() { f << "</g></svg>\n"; }
};
```

把这段保存为 `~/geo-week01-lab/geo.h`。**整个第 1 周不需要修改这个文件**。

---

## 文件 2：`test_bezier.cpp`（对照实验配套）

**这就是周一-周二实验的工作台**。AI 写的 Bezier 函数粘到标记位置之间，编译运行即可。

```cpp
// test_bezier.cpp — Bezier 求值的验证闭环
#include "geo.h"
#include <cassert>
#include <cmath>
#include <iostream>

// ╔══════════════════════════════════════════════════════════╗
// ║  ↓↓↓ 把 AI 生成的 bezier_cubic 函数粘到这里 ↓↓↓        ║
// ╚══════════════════════════════════════════════════════════╝

Point3D bezier_cubic(const Point3D& P0, const Point3D& P1,
                     const Point3D& P2, const Point3D& P3, double t) {
    // —— 占位实现（手写 De Casteljau，仅用于 setup 后冒烟测试） ——
    // 替换为 AI 生成的版本后再跑
    auto lerp = [](const Point3D& a, const Point3D& b, double s) {
        return a * (1 - s) + b * s;
    };
    Point3D q0 = lerp(P0, P1, t);
    Point3D q1 = lerp(P1, P2, t);
    Point3D q2 = lerp(P2, P3, t);
    Point3D r0 = lerp(q0, q1, t);
    Point3D r1 = lerp(q1, q2, t);
    return lerp(r0, r1, t);
}

// ╔══════════════════════════════════════════════════════════╗
// ║  ↑↑↑ 粘贴区域结束 ↑↑↑                                    ║
// ╚══════════════════════════════════════════════════════════╝

// 验证用的解析值：使用闭式 Bernstein 在 t=0.5 算一次（独立路径，不调用上面的函数）
Point3D bernstein_ref(const Point3D& P0, const Point3D& P1,
                      const Point3D& P2, const Point3D& P3, double t) {
    double u = 1 - t;
    return P0 * (u*u*u) + P1 * (3*u*u*t) + P2 * (3*u*t*t) + P3 * (t*t*t);
}

int main() {
    bool all_pass = true;
    auto check = [&](const std::string& name, const Point3D& a, const Point3D& b, double tol = 1e-9) {
        double err = std::sqrt((a.x-b.x)*(a.x-b.x) + (a.y-b.y)*(a.y-b.y) + (a.z-b.z)*(a.z-b.z));
        bool ok = err < tol;
        std::cout << (ok ? "  [PASS] " : "  [FAIL] ") << name
                  << "  got=(" << a.x << "," << a.y << "," << a.z << ")"
                  << "  expect=(" << b.x << "," << b.y << "," << b.z << ")"
                  << "  err=" << err << "\n";
        if (!ok) all_pass = false;
    };

    Point3D P0{0,0,0}, P1{1,4,0}, P2{4,4,0}, P3{5,0,0};

    std::cout << "=== Test 1: Normal curve (vs Bernstein reference) ===\n";
    for (double t : {0.0, 0.25, 0.5, 0.75, 1.0}) {
        Point3D got = bezier_cubic(P0, P1, P2, P3, t);
        Point3D ref = bernstein_ref(P0, P1, P2, P3, t);
        check("t=" + std::to_string(t), got, ref);
    }

    std::cout << "\n=== Test 2: Endpoint interpolation ===\n";
    check("P(0)=P0", bezier_cubic(P0, P1, P2, P3, 0.0), P0);
    check("P(1)=P3", bezier_cubic(P0, P1, P2, P3, 1.0), P3);

    std::cout << "\n=== Test 3: Coincident control points ===\n";
    Point3D Q{1,1,1};
    check("4-point-coincident", bezier_cubic(Q,Q,Q,Q, 0.5), Q);

    std::cout << "\n=== Test 4: Out-of-range t (degenerate) ===\n";
    Point3D low = bezier_cubic(P0, P1, P2, P3, -0.5);
    Point3D high = bezier_cubic(P0, P1, P2, P3, 1.5);
    std::cout << "  P(-0.5) = (" << low.x << "," << low.y << "," << low.z << ")\n";
    std::cout << "  P( 1.5) = (" << high.x << "," << high.y << "," << high.z << ")\n";
    std::cout << "  Question: AI 的代码是夹紧 / 外推 / 抛异常？符合你的 prompt 约束吗？\n";

    // —— SVG 可视化 ——
    SVG svg("bezier_output.svg", -1, -1, 6, 5);
    svg.polyline({P0, P1, P2, P3}, "#cccccc", 0.03);
    svg.dots({P0, P1, P2, P3}, "red", 0.08);
    std::vector<Point3D> curve;
    for (int i = 0; i <= 100; ++i)
        curve.push_back(bezier_cubic(P0, P1, P2, P3, i/100.0));
    svg.polyline(curve, "#2266ff", 0.04);

    std::cout << "\n=== SVG ===\nbezier_output.svg 已生成。在浏览器打开看曲线形态。\n";
    std::cout << (all_pass ? "\n✓ All numeric checks passed.\n" : "\n✗ Some checks failed!\n");
    return all_pass ? 0 : 1;
}
```

### 编译运行（**一行命令**）

```bash
cd ~/geo-week01-lab && g++ -std=c++17 -O2 -Wall test_bezier.cpp -o bezier && ./bezier && open bezier_output.svg
```

如果不是 macOS，把最后的 `open` 改成 `xdg-open`（Linux）或在浏览器里手动打开。

---

## 文件 3：`test_bspline.cpp`（周三 Cox-de Boor 配套）

B 样条基函数是**标量函数**，可视化方式不同：画 $N_{i,p}(u)$ 关于 $u$ 的曲线，几条不同的 $i$ 用不同颜色。

```cpp
// test_bspline.cpp — Cox-de Boor 基函数的验证闭环
#include "geo.h"
#include <cmath>
#include <iostream>
#include <vector>

// ╔══════════════════════════════════════════════════════════╗
// ║  ↓↓↓ 把 AI 生成的 basis 函数粘到这里 ↓↓↓                ║
// ╚══════════════════════════════════════════════════════════╝

double basis(int i, int p, double u, const std::vector<double>& knots) {
    // —— 占位实现 ——
    if (p == 0)
        return (u >= knots[i] && u < knots[i+1]) ? 1.0 : 0.0;
    double d1 = knots[i+p] - knots[i];
    double d2 = knots[i+p+1] - knots[i+1];
    double a = (d1 > 0) ? (u - knots[i]) / d1 * basis(i, p-1, u, knots) : 0.0;
    double b = (d2 > 0) ? (knots[i+p+1] - u) / d2 * basis(i+1, p-1, u, knots) : 0.0;
    return a + b;
}

// ╔══════════════════════════════════════════════════════════╝
// ║  ↑↑↑ 粘贴区域结束 ↑↑↑                                    ║
// ╚══════════════════════════════════════════════════════════╝

int main() {
    // 经典 cubic clamped knot vector
    std::vector<double> knots = {0, 0, 0, 0, 0.25, 0.5, 0.75, 1, 1, 1, 1};
    int p = 3;
    int n_ctrl = knots.size() - p - 1;  // = 7

    bool all_pass = true;

    std::cout << "=== Test 1: Partition of unity (sum should = 1) ===\n";
    for (double u : {0.0, 0.1, 0.3, 0.5, 0.7, 0.99}) {
        double sum = 0;
        for (int i = 0; i < n_ctrl; ++i) sum += basis(i, p, u, knots);
        bool ok = std::abs(sum - 1.0) < 1e-9;
        std::cout << (ok ? "  [PASS] " : "  [FAIL] ") << "u=" << u
                  << "  sum=" << sum << "  err=" << std::abs(sum - 1.0) << "\n";
        if (!ok) all_pass = false;
    }

    std::cout << "\n=== Test 2: Non-negativity ===\n";
    int violations = 0;
    for (double u = 0; u < 1.0; u += 0.01)
        for (int i = 0; i < n_ctrl; ++i)
            if (basis(i, p, u, knots) < -1e-12) violations++;
    std::cout << (violations == 0 ? "  [PASS] " : "  [FAIL] ")
              << "negative-value violations: " << violations << "\n";
    if (violations > 0) all_pass = false;

    std::cout << "\n=== Test 3: Boundary behavior at u=1.0 ===\n";
    double sum_at_one = 0;
    for (int i = 0; i < n_ctrl; ++i) sum_at_one += basis(i, p, 1.0, knots);
    std::cout << "  sum at u=1.0 = " << sum_at_one << "\n";
    std::cout << "  注意: 标准 Cox-de Boor 在 u=1.0 会返回 sum=0（半开区间问题）。\n";
    std::cout << "  AI 是否处理了这个 corner case？\n";

    // —— SVG 可视化：每个 N_{i,3}(u) 一条曲线 ——
    SVG svg("bspline_output.svg", -0.05, -0.1, 1.05, 1.2);
    // 坐标轴
    svg.line(0, 0, 1, 0, "#888", 0.005);
    svg.line(0, 0, 0, 1, "#888", 0.005);
    // knot 位置
    for (double k : knots) svg.line(k, 0, k, 0.05, "#aaa", 0.005);

    const char* colors[] = {"#e41a1c","#377eb8","#4daf4a","#984ea3","#ff7f00","#a65628","#f781bf"};
    for (int i = 0; i < n_ctrl; ++i) {
        std::vector<Point3D> curve;
        for (int k = 0; k <= 500; ++k) {
            double u = k / 500.0;
            curve.push_back({u, basis(i, p, u, knots), 0});
        }
        svg.polyline(curve, colors[i % 7], 0.008);
    }

    std::cout << "\nbspline_output.svg 已生成。每条彩色曲线是一个 N_{i,3}(u)。\n";
    std::cout << "可视判断: 曲线是否光滑？是否非负？是否在每个 knot span 内只有 4 条非零？\n";
    return all_pass ? 0 : 1;
}
```

### 编译运行

```bash
g++ -std=c++17 -O2 -Wall test_bspline.cpp -o bspline && ./bspline && open bspline_output.svg
```

---

## 文件 4：`test_nurbs.cpp`（周四 NURBS 配套）

NURBS 曲线复用 De Boor 思路，但带权重。可视化方式：**画两条曲线对比**——所有权重 = 1 的版本（退化为 B 样条）vs 修改某个权重的版本，能直观看到权重对曲线的"拉拽"。

```cpp
// test_nurbs.cpp — NURBS 求值的验证闭环
#include "geo.h"
#include <cmath>
#include <iostream>
#include <vector>

// 如果 test_bspline.cpp 还在同目录，可以 #include 它的 basis；
// 这里独立实现一份占位，避免链接冲突
static double bspline_basis(int i, int p, double u, const std::vector<double>& knots) {
    if (p == 0)
        return (u >= knots[i] && u < knots[i+1]) ? 1.0 : 0.0;
    double d1 = knots[i+p] - knots[i];
    double d2 = knots[i+p+1] - knots[i+1];
    double a = (d1 > 0) ? (u - knots[i]) / d1 * bspline_basis(i, p-1, u, knots) : 0.0;
    double b = (d2 > 0) ? (knots[i+p+1] - u) / d2 * bspline_basis(i+1, p-1, u, knots) : 0.0;
    return a + b;
}

// ╔══════════════════════════════════════════════════════════╗
// ║  ↓↓↓ 把 AI 生成的 nurbs_eval 函数粘到这里 ↓↓↓           ║
// ╚══════════════════════════════════════════════════════════╝

Point3D nurbs_eval(double u,
                   const std::vector<Point3D>& ctrl,
                   const std::vector<double>& weights,
                   const std::vector<double>& knots,
                   int degree) {
    // —— 占位实现 ——
    Point3D num{0,0,0};
    double den = 0;
    for (size_t i = 0; i < ctrl.size(); ++i) {
        double N = bspline_basis(i, degree, u, knots);
        double w = weights[i];
        num = num + ctrl[i] * (N * w);
        den += N * w;
    }
    return num * (1.0 / den);
}

// ╔══════════════════════════════════════════════════════════╝
// ║  ↑↑↑ 粘贴区域结束 ↑↑↑                                    ║
// ╚══════════════════════════════════════════════════════════╝

int main() {
    // 经典半圆 NURBS：3 个控制点，权重 (1, sqrt(2)/2, 1) 表示标准圆弧
    std::vector<Point3D> ctrl = {{1,0,0}, {1,1,0}, {0,1,0}};
    std::vector<double> w_circle = {1.0, std::sqrt(2.0)/2.0, 1.0};
    std::vector<double> w_uniform = {1.0, 1.0, 1.0};
    std::vector<double> knots = {0, 0, 0, 1, 1, 1};
    int degree = 2;

    bool all_pass = true;
    auto check = [&](const std::string& name, double got, double expect, double tol = 1e-6) {
        bool ok = std::abs(got - expect) < tol;
        std::cout << (ok ? "  [PASS] " : "  [FAIL] ") << name
                  << "  got=" << got << "  expect=" << expect << "\n";
        if (!ok) all_pass = false;
    };

    std::cout << "=== Test 1: Quarter-circle NURBS, points should lie on unit circle ===\n";
    for (double u : {0.0, 0.25, 0.5, 0.75, 1.0}) {
        Point3D p = nurbs_eval(u, ctrl, w_circle, knots, degree);
        double r = std::sqrt(p.x*p.x + p.y*p.y);
        check("|P(u=" + std::to_string(u) + ")|=1", r, 1.0);
    }

    std::cout << "\n=== Test 2: Endpoint interpolation ===\n";
    Point3D p0 = nurbs_eval(0.0, ctrl, w_circle, knots, degree);
    Point3D p1 = nurbs_eval(1.0 - 1e-9, ctrl, w_circle, knots, degree);
    check("P(0).x", p0.x, 1.0);
    check("P(0).y", p0.y, 0.0);
    check("P(1).x", p1.x, 0.0);
    check("P(1).y", p1.y, 1.0);

    std::cout << "\n=== Test 3: Uniform weights → degenerates to B-spline (NOT a circle) ===\n";
    Point3D mid_uniform = nurbs_eval(0.5, ctrl, w_uniform, knots, degree);
    double r_uniform = std::sqrt(mid_uniform.x*mid_uniform.x + mid_uniform.y*mid_uniform.y);
    std::cout << "  |P(0.5)| with uniform weights = " << r_uniform
              << "  (should NOT be 1.0 — about 0.85)\n";

    // —— SVG: 圆弧（红） + 均匀权重（蓝） + 参考单位圆（灰虚线效果用密集点） ——
    SVG svg("nurbs_output.svg", -0.2, -0.2, 1.4, 1.4);
    // 单位圆参考
    std::vector<Point3D> ref_circle;
    for (int k = 0; k <= 90; ++k) {
        double a = k * (M_PI / 2) / 90;
        ref_circle.push_back({std::cos(a), std::sin(a), 0});
    }
    svg.polyline(ref_circle, "#cccccc", 0.01);

    // NURBS 圆弧
    std::vector<Point3D> arc, arc_uniform;
    for (int k = 0; k <= 200; ++k) {
        double u = k / 200.0;
        arc.push_back(nurbs_eval(u, ctrl, w_circle, knots, degree));
        arc_uniform.push_back(nurbs_eval(u, ctrl, w_uniform, knots, degree));
    }
    svg.polyline(arc, "#e41a1c", 0.015);          // 真圆弧 - 红
    svg.polyline(arc_uniform, "#377eb8", 0.015);  // 均匀权重 - 蓝
    svg.polyline(ctrl, "#999999", 0.008);         // 控制多边形
    svg.dots(ctrl, "black", 0.025);

    std::cout << "\nnurbs_output.svg 已生成。\n";
    std::cout << "可视判断:\n";
    std::cout << "  灰色 = 真正的单位圆\n";
    std::cout << "  红色 = NURBS（权重对的，应贴合灰色）\n";
    std::cout << "  蓝色 = 退化为 B 样条（权重全 1，应偏离灰色）\n";
    std::cout << "  如果红色不贴合灰色，AI 的 NURBS 求值有问题。\n";
    return all_pass ? 0 : 1;
}
```

### 编译运行

```bash
g++ -std=c++17 -O2 -Wall test_nurbs.cpp -o nurbs && ./nurbs && open nurbs_output.svg
```

---

## 一键命令汇总

把这两行写进 `~/.config/fish/functions/geotest.fish`（你用 fish shell）：

```fish
function geotest --description "Compile & run a Week1 lab test"
    cd ~/geo-week01-lab
    set name $argv[1]
    g++ -std=c++17 -O2 -Wall test_$name.cpp -o $name; and ./$name; and open $name\_output.svg
end
```

之后只要 `geotest bezier` / `geotest bspline` / `geotest nurbs`。

---

## 完整工作流（融合进第 1 周日程）

### 周一-周二（对照实验）

1. 在 Gemini 上跑 3 个对照 prompt → 拿到 3 段 Bezier 代码
2. 各自粘进 `test_bezier.cpp` 的标记区域（**保留三份副本**：`test_bezier_A.cpp` / `_B.cpp` / `_C.cpp`）
3. 各自编译运行
4. 对比观察：
   - 数值测试：哪几版能 PASS 所有 5 个 t 值？
   - 端点测试：哪几版 P(0)、P(1) 严格精确？
   - 退化测试：哪几版能扛住四点重合？
   - SVG：曲线形态是否合理？有无尖角、突变、跑飞的点？

**这一步把"主观判断 prompt 谁好"变成"客观数据**——A 版可能在 t=0.999 处偏离 ref 1e-3，B 版偏离 1e-6，C 版 1e-15。**亲眼看到这个差距，prompt 工程的价值就立起来了**。

### 周三（Cox-de Boor）

1. 拿 AI 生成的 `basis` 函数粘进 `test_bspline.cpp`
2. 跑 `geotest bspline`
3. 三个核心验证：
   - **Partition of unity**：基函数之和 = 1（任何 u）
   - **Non-negativity**：所有 N ≥ 0
   - **Boundary**：u=1.0 处的行为
4. SVG 里看 7 条彩色曲线是否光滑、形态对称

**如果 AI 的代码在前两个数学性质上 FAIL，说明它递推公式实现错了**。这是无可辩驳的客观证据，比让 AI 自己"声称我的代码是对的"强 100 倍。

### 周四（NURBS）

1. AI 生成的 `nurbs_eval` 粘进 `test_nurbs.cpp`
2. 跑 `geotest nurbs`
3. **NURBS 最严苛的客观测试**：用 1/4 圆的标准权重，求值后 $\sqrt{x^2 + y^2}$ 必须 = 1
4. SVG 里红色弧线必须**严丝合缝**贴在灰色参考圆上

如果 AI 的 NURBS 求值有任何隐性 bug（权重归一化、节点处理、De Boor 实现等），**圆弧测试一定会暴露**。这是几何代码验证最经典的金标准——**圆性 (circularity test)**。

### 周末（综合）

把三份测试都跑过的代码归档成你的"AI 代码验证基线"：

```
~/geo-week01-lab/
├── geo.h
├── test_bezier.cpp        ← AI 最佳版本固化在这里
├── test_bspline.cpp       ← AI 最佳版本固化在这里
├── test_nurbs.cpp         ← AI 最佳版本固化在这里
├── bezier_output.svg
├── bspline_output.svg
└── nurbs_output.svg
```

下周开始新算法时，沿用这个范式：**先写测试 harness，再让 AI 写代码，跑测试直到 PASS**。

---

## 这套工具的"杠杆"在哪里

1. **从"主观判断"到"客观度量"**
   `prompt A 不太好` ≠ `prompt A 在 t=0.999 处误差 3.2e-4，prompt C 在同样位置误差 1.1e-15`。后者会让你深刻记住 prompt 工程的价值。

2. **从"AI 自夸"到"测试打脸"**
   AI 经常说"我的代码处理了所有边界情况"。圆性测试 / partition-of-unity 测试一跑，AI 的牛皮就破了。这是你建立**对 AI 的判断力**的基础。

3. **从"看代码"到"看图"**
   人脑读 100 行 C++ 比看一张曲线图慢 100 倍。SVG 让你 1 秒发现"这条曲线为什么在中间凸了一下"。

4. **后续 11 周的复用**
   半边数据结构、ARAP、网格平滑 ... 这些算法都可以套同样模式：**harness + paste-zone + numeric checks + SVG**。你周末攒下的这套工作流，将贯穿整 12 周。

---

## 常见踩坑

| 现象 | 原因 | 应对 |
|---|---|---|
| 编译报错 `Point3D` 重定义 | AI 在它的代码块里也定义了 Point3D | 删掉 AI 那份，用 harness 里的 |
| 编译报错 `std::vector` 未声明 | AI 没 include `<vector>` | 在 harness 已经 include，删 AI 重复的 |
| SVG 打开是空白 | 坐标范围不对，曲线在 viewBox 外 | 检查 SVG 的 viewBox 是否包含你的数据范围 |
| 测试一个都没 PASS | AI 误解了输入约定（如 row-major vs column-major） | 拿 1-2 个具体输入手算，对照 AI 输出哪一步偏离 |
| 圆弧测试明显偏离 | NURBS 权重处理错 / 端点处理错 / De Boor span 错 | 看 SVG，从哪个 u 开始偏离就是 bug 起点 |
| `open` 命令找不到 | 不是 macOS | Linux 用 `xdg-open`，Windows 用 `start` |

---

## 进阶（可选，**周末再做**）

### 给 prompt 加测试约束

下周可以在你的 prompt 模板里加上：

```
代码必须能通过以下测试：
1. P(0) 严格等于 P0（精度 1e-15）
2. P(1) 严格等于 P3（精度 1e-15）
3. 任意 t ∈ [0,1] 时，bezier_cubic 与 sum_{i=0}^3 C(3,i) (1-t)^(3-i) t^i P_i 的差 < 1e-10
4. 4 个控制点全部相等时，对任意 t 返回该点
请确保你的实现通过以上每一条。
```

这让 AI 的目标从"写一个看起来对的函数"变成"**写一个能通过这些 assertion 的函数**"——后者**质量陡升一个台阶**。

---

## 最后一句

> 没有验证闭环的 AI 输出，**只是文本**；
> 有了验证闭环的 AI 输出，才是**可信代码**。
> 第 1 周的 200 行 harness，会让你之后 11 周对 AI 产出的判断力，从"看起来对"升级到"测试过了"。
