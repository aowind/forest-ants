# 清晨森林的光：实时 3D 场景的光照、雾与体积光管线

同一个场景，把光调对之前和调对之后，是两个作品。

这篇拆一下这个森林场景的光照管线——从一盏 45° 角的太阳，到最后那道被树干挡住的丁达尔光束。

![](/assets/uploads/u_ddf3d03fa137aa5e.jpg)

## 一、主光：低角度暖色平行光

清晨的光有两个特征：**角度低**、**色温暖**。

```js
const sun = new THREE.DirectionalLight(0xffc07a, 4.6);
sun.position.set(-15, 9.5, 2.5);
```

位置向量决定方向：`(-15, 9.5, 2.5)` 换算下来入射角约 32°，属于「太阳刚过树梢」的高度。颜色 `0xffc07a` 是偏橙的暖白，强度 4.6 比常规值高——因为配合了 ACES 色调映射，高光会被压回来，不给足强度画面会发灰。

阴影参数是这里面最容易翻车的地方：

```js
sun.shadow.mapSize.set(2048, 2048);
sun.shadow.camera.left = -16;
sun.shadow.camera.right = 16;
sun.shadow.camera.top = 16;
sun.shadow.camera.bottom = -16;
sun.shadow.bias = -0.0012;
sun.shadow.normalBias = 0.055;
```

**`normalBias` 必须给足**。低角度平行光打在近乎水平的地面上，阴影坐标的深度变化极小，默认的 bias 完全不够，会出现规律的水平条纹——我最初在蚁道附近看到一片片平行横纹，还以为是地面纹理的问题，查了很久才发现是 shadow acne。

0.055 是试出来的：再小条纹回来，再大地面的接触阴影会「漂浮」（peter-panning）。

顺便一个版本坑：Three.js r186 已经**移除**了 `PCFSoftShadowMap`，写它会被静默降级。现在要用 `PCFShadowMap` 配合 `shadow.radius`。

## 二、补光：天光 + 轮廓 + 冷填充

单靠一盏平行光，暗部会死黑。三盏辅助光分层填：

```js
// 天光：上方冷蓝，地面反弹暖褐
const hemi = new THREE.HemisphereLight(0xa8c6de, 0x6d5a3c, 1.45);

// 背光：勾出物体轮廓，让树干边缘有一圈亮边
const rim = new THREE.DirectionalLight(0xffe0b0, 0.55);
rim.position.set(14, 5, 16);

// 极弱的冷填充：让阴影里不是纯黑，而是偏蓝
const fill = new THREE.DirectionalLight(0x9dc0e0, 0.28);
fill.position.set(6, 8, -10);
```

冷暖对撞是这套光的关键：主光暖橙、阴影偏蓝，画面立刻有了「清晨 vs 正午」的区别。

## 三、环境光：PMREM 让露珠有内容

露珠用的是 `MeshPhysicalMaterial` 的透射：

```js
transmission: 0.96, thickness: 0.35, ior: 1.4,
clearcoat: 1.0, clearcoatRoughness: 0.02
```

但透射材质如果没有环境贴图，会渲染成一坨半透明的灰球——因为它折射的「背后」什么都没有。

所以天空那张等距柱状图除了当背景，还被预滤波成 IBL：

```js
scene.environment = pmrem.fromEquirectangular(skyTex).texture;
scene.environmentIntensity = 0.55;
```

露珠里于是能看见蓝天和暖色地平线。蚂蚁的甲壳同理——`clearcoat: 0.9` 配 `envMapIntensity: 1.6`，小小的身体上有一道天光反射，在深色地面上才「立」得起来。

## 四、雾：颜色必须和地平线对齐

```js
scene.fog = new THREE.Fog(0xe3cfa4, 22, 66);
```

雾色不是随便挑的暖米色。它是从天空纹理**地平线附近的颜色**取样来的。如果雾色和天空底部对不上，远处地面和天空之间会出现一条突兀的分界线，整个场景的「无限延伸感」就没了。

这一条改完，画面立刻从「一个方形地板」变成「一片林子」。

## 五、丁达尔光束：先解决穿模

体积光的常见做法是半透明圆柱 + 加法混合。第一版我用了 7 根 16–22 米长的光柱，结果：

**光穿过了树干。**

原因很物理——光柱斜射下来，水平跨度 14 米，必然横穿整片林子。而 `depthTest` 只能挡住「树干后面」的部分，「树干前面」的那段仍然会叠在树干上，看起来就像光直接穿过了实心木头。

修复分三步：

**① 给树干登记碰撞体**。建树时顺手 push 到全局数组：

```js
TREE_COLLIDERS.push({ x: baseX, z: baseZ, r: 2.4 * scale });
```

**② 生成光柱时沿轴线采样检测**：

```js
function beamHitsTree(cx, cy, cz, h) {
  for (let s = -0.5; s <= 0.5001; s += 0.07) {
    const px = cx + lightDir.x * h * s;
    const pz = cz + lightDir.z * h * s;
    for (const t of TREE_COLLIDERS) {
      if (Math.hypot(px - t.x, pz - t.z) < t.r + 0.35) return true;
    }
  }
  return false;
}
```

最多重试 90 次，实在找不到不撞树的位置就跳过这根。

**③ 缩短并减淡**。长度 16–22 → 6–10.5 米，底半径 0.7 → 0.24，不透明度 0.032 → 0.014–0.028。

光柱于是从「横贯画面的白带」变成「树干间隙里的局部柔光」——这才是它该有的样子。

## 六、地面光斑：加法混合的便宜魔法

树冠缝隙洒下来的光点，是用 26 个平放的 `PlaneGeometry` 做的：

```js
const mat = new THREE.MeshBasicMaterial({
  map: spotTex,                    // 径向渐变 canvas
  transparent: true,
  blending: THREE.AdditiveBlending,
  depthWrite: false,
  opacity: rr(0.13, 0.30)
});
m.rotation.x = -Math.PI / 2;
m.scale.set(s * rr(0.65, 1.4), s, 1);   // 拉成椭圆，避免「圆点」感
m.renderOrder = 2;
```

加法混合 + 大尺寸椭圆 + 低不透明度，是这三种参数组合让它看起来像「光」而不是「贴纸」。

## 七、后期：轻一点

```js
new UnrealBloomPass(new THREE.Vector2(w, h), 0.22, 0.55, 0.92)
```

强度只给 0.22、阈值 0.92。Bloom 一旦超过 0.4，白色菌盖上的斑点会开始发光，像 LED。

色调映射用 `ACESFilmicToneMapping`，曝光 1.08。ACES 会把高光滚降得很舒服，代价是中间调偏灰——所以前面主光强度才要给到 4.6。

## 八、最后两件事：浮尘和落叶

静止的场景再漂亮也像照片。两样东西让它活过来：

**浮尘**——620 个 `Points`，每个点用一个固定的随机种子做正弦位移：

```js
pos.setX(i, bx + Math.sin(time * 0.35 + s) * 0.55);
pos.setY(i, by + Math.sin(time * 0.22 + s * 1.7) * 0.45 + ((time * 0.10 + s * 3.1) % 1.2));
```

`% 1.2` 那项让它缓慢上升然后循环，配合加法混合，在光柱附近会自然变亮。

**落叶**——9 片叶子，各自有独立的转速和摆动相位，落到地面后重置到树冠高度。

## 结语

这套管线的成本很低：一盏带阴影的平行光、三盏无阴影补光、一根半球光、一张 PMREM 环境图、26 个加法混合平面、7 根光柱、620 个点。

真正花时间的不是「加什么」，而是**把每个东西的强度调到位**——尤其是那些会互相打架的部分：主光和天光的比例、雾色和地平线、Bloom 和材质亮度。

调光没有捷径，只有截图对比。

![](/assets/uploads/u_06e3b2ebf2d6ba15.jpg)

---

场景在线：<https://aowind.github.io/forest-ants/>
源码：<https://github.com/aowind/forest-ants>
