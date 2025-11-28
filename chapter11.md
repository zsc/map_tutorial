# 第 11 章：室内地图构建与 BIM 融合

**——打通“最后一百米”：从建筑模型到空间导航**

## 1. 开篇段落

在前面的章节中，我们的视角穿越了数千公里的卫星轨道，俯瞰了城市尺度的路网与街景。然而，现代人类 80% 以上的时间是在室内度过的。当用户驾车通过我们的室外导航到达大型机场、复杂的购物中心或综合医院时，基于 GPS 的导航服务往往会因为信号遮挡和缺乏内部细节而戛然而止。

**室内地图（Indoor Mapping）** 被称为地理信息服务的“最后一百米”。这不仅仅是室外地图的比例尺放大，而是一个全新的领域：坐标系从全球经纬度变为局部笛卡尔坐标，定位源从卫星为 WiFi/蓝牙/UWB，数据结构从平面拓扑变为分层立体拓扑。

本章将带你深入室内地图的内核。我们将以建筑行业的通用标准 **BIM (Building Information Modeling)** 为起点，解构如何通过 ETL 流程提取空间要素，如何利用仿射变换实现室内外坐标的无缝对齐，以及如何构建支持跨楼层（Multi-floor）路径规划的导航网格。

---

## 2. 核心原理与论述

### 2.1 室内空间数据的“世界观”

构建室内地图的最大挑战在于思维模式的转换。室外地图假设空间是连续的，而室内地图的空间是高度离散和受限的。

#### 2.1.1 空间模型对比

| 特性 | 室外 GIS (Outdoor) | 室内 GIS (Indoor) |
| :--- | :--- | :--- |
| **参考系** | WGS84 / Web Mercator (绝对坐标) | 建筑局部坐标系 (相对坐标) |
| **Z 轴含义** | 海拔高程 (Elevation) | **楼层 (Level/Floor)** + 相对高度 |
| **连通性** | 平面路网为主，立交桥为辅 | 垂直交通 (电梯/楼梯) 是核心枢纽 |
| **定位源** | GNSS (GPS/Beidou/Galileo) | WiFi 指纹, iBeacon, UWB, 视觉 SLAM |
| **边界** | 开放空间，软边界 (行政区划) | 封闭空间，硬边界 (墙体、门禁) |

> **Rule of Thumb (经验法则)**：
> 在设计室内地图数据结构时，**永远不要把“楼层”仅仅当做一个属性字段**。楼层应该是一个物理维度的过滤器。在 Web 地图中，通常建议将每一层作为独立的 Vector Tile（矢量瓦片）图层源，或者使用 `Level` 字段进行强过滤，避免一次性加载整栋楼的所有数据导致渲染崩溃。

### 2.2 数据源：BIM 与 IFC 标准

**BIM (Building Information Modeling)** 是建筑学与工程领域的数字化标准。对于 GIS 工程师而言，BIM 是最丰富的室内数据金矿。我们主要关注通用的数据交换格式 **IFC (Industry Foundation Classes)**。

一个 IFC 文件不仅包含了几何形状（Mesh），还包含了语义（Semantics）。构建室内地图的质是从 BIM 中进行**降维（Dimensionality Reduction）**和**语义映射**。

#### 2.2.1 关键 IFC 实体的 GIS 映射

在解析 IFC 文件时，我们需要关注以下核心实体：

*   **`IfcProject` / `IfcSite`**: 提供项目的基准点和地理参考信息。
*   **`IfcBuildingStorey`**: 对应 GIS 中的 **楼层 (Level)**。
*   **`IfcSpace`**: 对应 GIS 中的 **房间 (Polygon)**。这是室内地图最重要的展示要素。
*   **`IfcWall` / `IfcColumn`**: 对应 GIS 中的 **障碍物/墙体 (Polygon/Line)**。
*   **`IfcDoor`**: 对应 GIS 中的 **通行节点 (Point/Line)**，连接两个 Space。
*   **`IfcSlab`**: 对应 **地板**，通常用于生成楼层的背景轮廓。

### 2.3 几何处理流水线 (The Geometry Pipeline)

直接将 3D BIM 导入 WebGL 地图引擎通常是不可行的（数据量过大、细节过多）。我们需要通过切片（Slicing）技术生成平面图。

#### 2.3.1 “切片”算法逻辑

想象一把水平的“激光刀”，在一层楼的特定高度横扫而过：

1.  **确定切片高度**：通常选择距楼板 **1.2米 至 1.5米** 处。这个高度可以避开低矮家具，同时又能切到窗户和门。
2.  **布尔运算**：计算切平面与 3D 构件（墙、柱）的交集（Intersection）。
3.  **轮廓提取**：将交集结果转化为二维多边形。

**ASCII 图解：切片原理**

```text
    [ 3D BIM 模型截面 ]              [ 生成的 2D GIS 地图 ]

       |           |
       |   墙体    |                +---------+
   ----|-----------|---- < 切割线   |         |
       |           |     (1.5m)     |  房间   |
    ___|___     ___|___             | Polygon |
   |       |   |       |            |         |
   | 地板  |   | 地板  |            +----  ---+
===+=======+===+=======+===              ||
   IfcSlab      IfcSlab                 门(Door)
```

### 2.4 坐标系对齐：仿射变换数学

BIM 模型通常建立在工程坐标系中（例如 $(0,0,0)$ 是建筑主入口的地面），单位通常是毫米（mm）。而 Web 地图使用的是 WGS84 或 Web Mercator（米）。

将 BIM 坐标 $(x, y)$ 转换为 GIS 投影坐标 $(E, N)$ 需要进行**仿射变换（Affine Transformation）**。这通常包含缩放（Scale）、旋转（Rotation）和平移（Translation）。

设：
*   $S$ 为缩放因子（例如 $0.001$，将毫米转为米）。
*   $\alpha$ 为旋转角度（BIM 正北与地理正北的夹角，Azimuth）。
*   $(T_x, T_y)$ 为平移量（BIM 原点在 GIS 投影坐标系下的位置）。

变换公式如下：

$$
\begin{bmatrix} E \\ N \end{bmatrix} =
S \cdot
\begin{bmatrix}
\cos \alpha & -\sin \alpha \\
\sin \alpha & \cos \alpha
\end{bmatrix}
\cdot
\begin{bmatrix} x \\ y \end{bmatrix}
+
\begin{bmatrix} T_x \\ T_y \end{bmatrix}
$$

**注意**：在 Web Mercator 投影中，高纬度地区的长度变形严重。严谨的做法是先将 GIS 底图坐标投影到适合当地的平面坐标系（如 UTM），进行匹配后再转回经纬度；或者在换矩阵中引入非均匀缩放因子。

### 2.5 导航网格与拓扑构建

有了房间和墙壁的图形，我们还需要构建**导航图（Navigation Graph）**才能实现寻路。

#### 2.5.1 平面拓扑生成
常用的自动生成算法有：
1.  **中轴变换 (Medial Axis Transform)**：提取房间多边形的“骨架”，生成位于走廊中心的路径线。
2.  **可见性图 (Visibility Graph)**：连接互为可见的关键点（如门到门），适合开阔的大厅。

#### 2.5.2 垂直连接 (Vertical Connectivity)
這是室内导航的灵明。不同楼层是独立的平面图（Sub-graph），通过“连接器（Connector）”缝合。

定义一个图 $G = (V, E)$，其中垂直边 $E_{vertical}$ 的构建规则如下：

$$
\text{Link}(u, v) \iff \text{Type}(u)=\text{Type}(v) \in \{\text{Stair}, \text{Elevator}\} \land |\text{Level}(u) - \text{Level}(v)| = 1
$$

即：如果两个节点类型相同（都是楼梯A的节点），且楼层相邻，则建立连接。

**ASCII 图解：跨楼层拓扑**

```text
Level 2:   [Room A] --- (Door) --- [Corridor] --- [Elevator Node 2]
                                                         |
                                                         | (垂直边 Weight=5s)
                                                         |
Level 1:   [Room B] --- (Door) --- [Corridor] --- [Elevator Node 1]
```

### 2.6 数据标准：IMDF 与 IndoorGML

为了解决室内地图格式混乱的问题，了解主流标准非常重要：

1.  **IMDF (Indoor Mapping Data Format)**: Apple 推出的基于 GeoJSON 的标准。
    *   特点：结构清晰，商业化应用最广，被 OGC 采纳为标准。
    *   核心要素：`Unit` (房间), `Level` (楼层), `Facility` (设施), `Anchor` (定位点)。
2.  **IndoorGML**: OGC 的一种基于 XML 的标准。
    *   特点：侧重于拓扑空间关系和导航（Node-Relation Graph），而非几何外观。

---

## 3. 本章小结

*   **从实体到空间**：BIM 关注的是“墙和板”（实体），GIS 关注的是“房间和走廊”（空间）。制作室内地图的过程就是从实体中提取负空间（Negative Space）的过程。
*   **坐标转换核心**：必须掌握二维仿射变换矩阵，处理好 BIM 局部坐标与 GIS 全球坐标的旋转、缩放和平移关系。
*   **分层治理**：楼层（Level）是室内数据的核心维度。数据存储和渲染都应以楼层为单位进行组织。
*   **拓扑连接**：室内导航依赖于“节点-边”图论模型，其中楼梯和电梯是连接不同楼层子图的关键“虫洞”。

---

## 4. 常见陷阱与错误 (Gotchas)

### 4.1 “几乎闭合”的几何体 (Leakage)
*   **现象**：在 BIM 中，墙体之间可能有微小的肉眼不可见的缝隙（例如 1mm）。
*   **后果**：在 GIS 中进行多边形化（Polygonization）或生成导航网格时，算法会认为房间是漏的，导致无法生成封闭的房间面，或者导航路径“漏”到墙外。
*   **调试技巧**： ETL 流程中引入 **Snap（捕捉）** 容差机制（例如设定 2cm 的容差），强制闭合微小的缝隙。

### 4.2 坐标精度抖动 (The Large Coordinate Problem)
*   **现象**：WebGL 引擎通常使用 32 位浮点数。如果直接使用 UTM 坐标（如 $x=500000, y=4000000$），加上小数位，有效数字可能会溢出。
*   **后果**：地图模型在浏览时出现剧烈的抖动或顶点闪烁。
*   **调试技巧**：使用 **RTC (Relative To Center)** 技术。在加载数据时，将所有顶点的坐标减去建筑中心的坐标，只传输相对值给 GPU，在 Shader 中再动态加上中心点坐标。

### 4.3 楼层命名混乱
*   **现象**：BIM 文件中标记为 "Level 1"，在实际运营中可能是 "L1"、"F1"、"Ground Floor" 甚至 "Level 0"。
*   **后果**：用户看到的楼层与电梯按钮不符；数据无法与 POI 匹配。
*   **调试技巧**：建立 **`ordinal` (物理层序)** 和 **`display_name` (显示名称)** 的映射表。
    *   Level 0 -> ordinal: 0, display: "G"
    *   Level 1 -> ordinal: 1, display: "F1"
    *   Basement -> ordinal: -1, display: "B1"

### 4.4 玻璃与透明材质
*   **现象**：激光雷达扫描或自动切片算法往往会忽略玻璃墙（因为透明或被视为窗户）。
*   **后果**：导航规划出的路径穿过了商场的玻璃护栏，导致严重的可用性问题。
*   **调试技巧**：在 BIM 导出阶段，必须显式地将 `IfcCurtainWall`（幕墙）、`IfcRailing`（栏杆）和 `IfcWindow`（如果是落地窗）标记为 **不可通行障碍物 (Obstacles)**。
