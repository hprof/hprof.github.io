---
layout: post
title: 在地图上均匀生成单位
excerpt:  在地图内需要随机掉落指定数量的宝箱, 两个宝箱之间的距离不能小于k
---

问题可以简化为, 在长为x, 宽为y的二维平面, 随机生成n个点, 两点之间的距离不能小于k

### 随机试错法

直接在二维平面上随机生成坐标, 生成出来的点与结果集里的所有点计算欧几里得距离, 如果低于k则跳过, 大于等于k则加入到结果集里.

```java
public static List<Point> generateRandomPoints(int x, int y, int n, double k) {
    List<Point> points = new ArrayList<>();
    Random random = new Random();
    // 最大尝试次数，防止无限循环
    int maxAttempts = 1000;
    // 估算地图是否能容纳 n 个点
    double area = x * y;
    double minAreaPerPoint = Math.PI * (k / 2) * (k / 2); // 每个点的“占用”圆形面积
    if (n * minAreaPerPoint > area) {
        System.out.println("警告：地图面积可能不足以容纳 " + n + " 个点，最小距离为 " + k);
    }
    int attempts = 0;
    while (points.size() < n && attempts < maxAttempts) {
        // 随机生成点的坐标
        double px = random.nextDouble() * x;
        double py = random.nextDouble() * y;
        Point newPoint = new Point(px, py);
        // 检查是否满足最小距离约束
        if (isValidPoint(newPoint, points, k)) {
            points.add(newPoint);
            attempts = 0; // 重置尝试次数
        } else {
            attempts++;
        }
    }
    if (points.size() < n) {
        System.out.println("只生成了 " + points.size() + " 个点，可能由于距离约束或地图太小");
    }
    return points;
}
```

该算法实现简单, 在点不多的情况下还可以接受, 但在点数多的情况下试错成本过高.

### 泊松盘采样法

直接随机法最大的问题是每生成一个点就要与之前生成的所有点做距离判断. 直观上我们认为生成的点只需要与附近的点做距离判断即可. 

我们可以将地图划分为多个网格来加速距离检查, 网格单元大小为 k/√2, 确保点位于格子内的任意位置时, 其半径都可以将格子覆盖到.  维护一个活跃点列表, 从一个随机点开始.  对每个活跃点, 在其周围环形区域（半径范围 [k, 2k]）内生成若干候选点. 检查候选点附近格子内的点是否满足最小距离约束, 若满足则加入点集和活跃列表.  如果活跃点无法生成新点, 则将其移除. 继续直到生成 n 个点或活跃列表为空.

```java
public static ArrayList<Point> generatePoissonDiskPoints(int x, int y, int n, double k) {
    ArrayList<Point> points = new ArrayList<>();
    ArrayList<Point> activeList = new ArrayList<>();
    Random random = new Random();
    // 初始化背景网格，单元大小为 k/√2
    double cellSize = k / Math.sqrt(2);
    int gridWidth = (int) Math.ceil(x / cellSize);
    int gridHeight = (int) Math.ceil(y / cellSize);
    Point[][] grid = new Point[gridHeight][gridWidth]; // 存储点的引用
    // 生成第一个随机点
    Point firstPoint = new Point(random.nextDouble() * x, random.nextDouble() * y);
    points.add(firstPoint);
    activeList.add(firstPoint);
    // 将第一个点放入网格
    int gridX = (int) (firstPoint.x / cellSize);
    int gridY = (int) (firstPoint.y / cellSize);
    grid[gridY][gridX] = firstPoint;
    // 参数：每个活跃点尝试生成候选点的次数
    int maxAttemptsPerPoint = 30;
    while (!activeList.isEmpty() && points.size() < n) {
        // 随机选择一个活跃点
        int activeIndex = random.nextInt(activeList.size());
        Point activePoint = activeList.get(activeIndex);
        boolean found = false;
        // 在活跃点周围环形区域 [k, 2k] 内生成候选点
        for (int i = 0; i < maxAttemptsPerPoint; i++) {
            // 生成随机角度和半径
            double theta = random.nextDouble() * 2 * Math.PI;
            double r = k + random.nextDouble() * k; // 半径在 [k, 2k]
            double px = activePoint.x + r * Math.cos(theta);
            double py = activePoint.y + r * Math.sin(theta);
            // 检查候选点是否在地图范围内
            if (px < 0 || px >= x || py < 0 || py >= y) {
                continue;
            }
            Point candidate = new Point(px, py);
            int candGridX = (int) (px / cellSize);
            int candGridY = (int) (py / cellSize);
            // 检查候选点是否满足最小距离约束
            boolean valid = true;
            // 检查候选点所在网格及周围 5x5 网格内的点
            for (int dy = -2; dy <= 2 && valid; dy++) {
                for (int dx = -2; dx <= 2; dx++) {
                    int nx = candGridX + dx;
                    int ny = candGridY + dy;
                    if (nx >= 0 && nx < gridWidth && ny >= 0 && ny < gridHeight) {
                        Point neighbor = grid[ny][nx];
                        if (neighbor != null && candidate.distance(neighbor) < k) {
                            valid = false;
                            break;
                        }
                    }
                }
            }
            // 如果候选点有效，加入点集和活跃列表
            if (valid) {
                points.add(candidate);
                activeList.add(candidate);
                grid[candGridY][candGridX] = candidate;
                found = true;
                if (points.size() >= n) {
                    break;
                }
            }
        }
        // 如果当前活跃点无法生成新点，移除它
        if (!found) {
            activeList.remove(activeIndex);
        }
    }
    if (points.size() < n) {
        System.out.println("只生成了 " + points.size() + " 个点，可能由于距离约束或地图太小");
    }
    return points;
}
```

该算法可以使每生成一个点的距离检查时间复杂度从O(n)降到O(1), 更适合大量点生成.
