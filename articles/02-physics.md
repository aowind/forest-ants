# 让蚂蚁真的爬进洞里：3D 场景里的高度场物理

第一版做完时，蚁穴只是一个黑色的圆盘，蚂蚁走到它附近就**缩小消失**。

看起来像「穿过去了」。

问题很直白：蚂蚁的位置来自一条二维贝塞尔曲线，`y` 永远是地面高度。而蚁穴是一个凸起的土丘，顶端有洞。蚂蚁走的是一条穿过土丘内部的直线，视觉上当然是穿模。

这篇记录我是怎么把它改成「必须爬上去才能下去」的。

![](/assets/uploads/u_d6b927997e91b241.jpg)

## 一、把蚁穴变成一个高度场

核心想法很简单：**蚁穴不再是一个装饰物，它是地形的一部分**。

给地表高度加一个函数：

```js
function surfaceHeight(x, z) {
  return terrainHeight(x, z) + moundHeight(holeDist(x, z));
}
```

`holeDist` 是到洞口中心的水平距离，`moundHeight(r)` 返回该半径处土丘高出地面多少。

它的形状由两段剖面定义——土丘外表面和洞内漏斗：

```js
const MOUND_PROFILE = [
  [0.60, 0.66], [0.78, 1.02], [1.05, 1.05],
  [1.45, 0.82], [1.95, 0.48], [2.55, 0.16], [3.05, -0.05]
];
const FUNNEL_PROFILE = [
  [0.60, 0.66], [0.56, 0.42], [0.48, 0.02],
  [0.40, -0.38], [0.32, -0.78], [0.27, -1.05]
];
```

格式是 `[半径, 高度]`。土丘从洞口边缘（0.60 m）向外升到 1.05 m 的丘顶，再缓降到 3.05 m 处与地面相接；洞内则从 0.66 m 一路下探到 -1.05 m。

```js
function moundHeight(r) {
  if (r >= 3.05) return 0;
  if (r <= 0.60) return lerpProfile(FUNNEL_PROFILE, r, true);
  return lerpProfile(MOUND_PROFILE, r, false);
}
```

因为剖面点是线性连接的，插值也用线性，两者完全贴合。

## 二、几何必须和高度场说同一件事

高度场只是数学。玩家看到的是几何。如果两者不一致，蚂蚁就会悬浮或陷进土里。

所以土丘用 `LatheGeometry` 旋转这两段剖面生成，**用的就是同一组数字**：

```js
const prof = [
  new THREE.Vector2(3.05, -0.05),
  new THREE.Vector2(2.55, 0.16),
  // …
  new THREE.Vector2(0.60, 0.66)
];
const moundGeo = new THREE.LatheGeometry(prof, 52);
```

这里踩了一个很值得记的坑。

**剖面点的顺序决定法线方向。**

我最初按「内 → 外」写（0.60 在前、3.05 在后），生成出来的土丘整片是纯黑的。用脚本把顶点法线打出来一看：

```json
{"r": 1.05, "y": 1.05, "ny": -0.95}
```

`ny = -0.95`——法线朝下。土丘顶面全都背光，所以是黑的。

一开始我以为是 `computeVertexNormals()` 的问题，去掉它、加上它、翻转它，都没用。真正的解法是把剖面点**倒过来**（外 → 内），绕序自然就反了：

```js
const prof = [
  new THREE.Vector2(3.05, -0.05),   // 外
  new THREE.Vector2(2.55, 0.16),
  // …
  new THREE.Vector2(0.60, 0.66)     // 内
];
```

土丘立刻恢复正常受光。

顺带还有第二个坑：土丘外缘（半径 3.05 处）如果只是固定高度，会和起伏的地面之间出现一圈环形沟。解决是**让外缘顶点贴合地形**：

```js
for (let i = 0; i < p.count; i++) {
  const r = Math.hypot(v.x, v.z);
  if (r < 1.8) continue;
  const blend = smooth(1.8, 3.05, r);          // 内圈保持剖面，外圈过渡到地面
  const groundLocal = terrainHeight(v.x + HOLE.x, v.z + HOLE.z) - baseY + 0.02;
  p.setY(i, v.y * (1 - blend) + groundLocal * blend);
}
```

## 三、蚂蚁开始爬坡

有了高度场，蚂蚁的位置就变成：

```js
p.y = surfaceHeight(p.x, p.z) + 0.05;   // 0.05 是让腿末端刚好落在表面
```

它自己就会翻过土丘。实测轨迹：

| 到洞口距离 | 高度 | 状态 |
|---|---|---|
| 4.64 m | 0.26 m | 平地行走 |
| 3.64 m | 0.28 m | 接近土丘 |
| 2.65 m | 0.42 m | 踏上外坡 |
| 1.66 m | 0.94 m | 翻过洞口唇边 |
| 0.03 m | -0.84 m | 已入洞，缩放 0.13 |

进洞是最后一步：当曲线参数 `t > 0.962`，蚂蚁的位置开始向洞心收拢，同时沿漏斗壁下潜：

```js
if (tt > 0.962) {
  const k = smooth(0.962, 1.0, tt);
  p.x += (HOLE.x - p.x) * k * 0.985;
  p.z += (HOLE.z - p.z) * k * 0.985;
  p.y = surfaceHeight(p.x, p.z) + 0.05;   // 这里已经是漏斗内部了
  scl = 1 - k * 0.93;                     // 边下行边缩小
  pitch = Math.atan2(-0.9, 1.0) * k;      // 低头钻洞
}
```

因为 `surfaceHeight` 在 `r < 0.60` 时走的是漏斗剖面，蚂蚁会**贴着洞壁**往下滑，而不是垂直掉落。缩放让它看起来是「钻进去」而不是「缩成一个点」。

## 四、身体要跟着坡度俯仰

只改高度会显得僵硬——蚂蚁像电梯一样上下平移。真实的上坡会前倾。

每帧在蚂蚁前方 0.28 米采一次高度，算倾角：

```js
const d = 0.28;
const aheadY = surfaceHeight(p.x + tan.x * d, p.z + tan.z * d);
const hereY  = surfaceHeight(p.x, p.z);
const pitch  = Math.atan2(aheadY - hereY, d) * 0.85;
body.rotation.z = pitch + Math.sin(time * 5.5 + u.phase) * 0.038;
```

`Math.atan2` 而不是 `Math.atan`，因为要区分上坡和下坡的正负。最后那项是行走时的轻微摆动，让俯仰不至于太机械。

## 五、顺带解决「不重叠」

之前每只蚂蚁的速度是 `0.038 × random(0.94, 1.06)`。跑几分钟后，快的追上了慢的，队列开始叠在一起。

改成**全体同速**，只靠初始的曲线参数间隔维持队形：

```js
const ANT_COUNT = 18;
const ANT_SPACING = 0.050;   // ≈0.95 世界单位
const ANT_SPEED = 0.038;
```

蚂蚁体长 0.55 米，间距 0.95 米，留了 0.4 米余量——加上它头顶扛的食物前伸约 0.25 米，刚好不碰前一只的尾部。

同时把地面物件对路径的避让半径全线加大（草 1.0→1.25，苔藓 0.75→0.95，枯枝 0.85→1.05），蚁穴周围 3.4 米内不长草，否则洞口会被草挡得看不见。

## 六、为什么不用物理引擎

有人会问：为什么不直接接 Rapier 或 Cannon？

因为这里需要的不是刚体动力学，是**一个确定性的高度查询**。蚂蚁不需要被推、被碰撞、被摩擦，它只需要知道「我脚下有多高」。

`surfaceHeight` 是一次纯函数调用，零成本；物理引擎要为 18 只蚂蚁各建一个胶囊体，还要处理与地形网格的连续碰撞检测，开销大两个数量级，而结果是一样的。

选对抽象层级，比选对库重要。

![](/assets/uploads/u_2429823a753fe40a.jpg)

---

场景在线：<https://aowind.github.io/forest-ants/>
源码：<https://github.com/aowind/forest-ants>
