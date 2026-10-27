# 零贴图的 3D 森林：用 Canvas 2D 在运行时生成全部材质

做这个森林场景时，我给自己定了一条硬约束：**整个作品是一个 HTML 文件**，不能有任何外部贴图、模型、字体文件。

理由很实际——单文件意味着你可以把它丢进任何静态服务器、发邮件、塞进一个 gist，都能跑；也意味着没有 CORS、没有加载等待、没有「图丢了」的 404。

代价是：地面、树皮、天空、蘑菇、苔藓，全部要在运行时用代码画出来。

![](/assets/uploads/u_972368c1a5739c1a.jpg)

## 一、地面：可平铺噪声纹理

森林地面不是一种颜色，它是「深褐土壤 + 苔藓斑块 + 枯叶碎屑 + 高频颗粒」的叠加。

第一版我直接画了随机圆斑，结果一平铺，接缝处全是硬边。因为每个笔触画到画布边缘就被截断了，平铺时相邻两块拼不上。

解决办法是**环绕绘制**：每个笔触都在 3×3 的偏移位置重复落笔，越界的那一份会在对面重现。

```js
const wrapFill = (cx, cy, r, fill) => {
  for (const dx of [-size, 0, size]) {
    for (const dy of [-size, 0, size]) {
      const px = cx + dx, py = cy + dy;
      // 只画与画布相交的那几份，其余跳过
      if (px + r < 0 || px - r > size || py + r < 0 || py - r > size) continue;
      ctx.fillStyle = fill;
      ctx.beginPath();
      ctx.arc(px, py, r, 0, 6.2832);
      ctx.fill();
    }
  }
};
```

有了它，2560 个笔触（1500 个泥土色块、760 个苔藓斑、300 个枯叶碎屑）画完之后，纹理在任意方向上都能无缝拼接。

最后再叠一层逐像素的高频噪声：

```js
const img = ctx.getImageData(0, 0, size, size);
const d = img.data;
for (let i = 0; i < d.length; i += 4) {
  const n = (Math.random() - 0.5) * 26;
  d[i] += n; d[i + 1] += n; d[i + 2] += n;
}
ctx.putImageData(img, 0, 0);
```

这张图会被用两次：一次当 `map`（颜色），一次当 `bumpMap`（凹凸），`repeat` 设成 30×30。

## 二、天空：一份数据同时当背景和环境光

天空用等距柱状投影（equirectangular）的垂直渐变 canvas：顶部深蓝，地平线附近淡米色，底部暖褐。太阳位置画一圈径向渐变光晕，再撒几道稀薄云带。

关键在于这张图**用了两次**：

```js
const skyTex = makeSkyTexture();          // CanvasTexture
skyTex.mapping = THREE.EquirectangularReflectionMapping;

scene.background = skyTex;                // ① 直接当天空盒

const pmrem = new THREE.PMREMGenerator(renderer);
scene.environment = pmrem.fromEquirectangular(skyTex).texture;  // ② 预滤波成 IBL
```

① 让你看见天空，② 让所有 `MeshStandardMaterial` 都能反射它——露珠的镜面高光、蚂蚁甲壳的清漆反光、金属感石头的环境色，全部来自这一张渐变图。

配合 `scene.environmentIntensity = 0.55`，环境光不会盖过主光源，保持清晨该有的明暗对比。

## 三、树皮：竖条纹 + 沟壑

树皮纹理是一张 256×512 的图，画法是「亮暗竖条 + 深色沟壑 + 苔藓附着」：

```js
// 430 条亮度在 0.78~1.22 之间波动的竖条
for (let i = 0; i < 430; i++) {
  const cx = Math.random() * 256, w = rr(1.5, 7), h = rr(90, 420);
  const l = rr(0.78, 1.22);
  ctx.fillStyle = `rgba(${84*l|0},${64*l|0},${46*l|0},${rr(0.10,0.26)})`;
  ctx.fillRect(cx, cy, w, h);
  if (cy + h > 512) ctx.fillRect(cx, cy - 512, w, h);   // 纵向接缝处理
}
```

这里有个踩过的坑：最初亮度范围取 0.55~1.35，对比太强，平铺 4 次后树干上出现明显的一圈圈横向亮带，像斑马。把范围收到 0.78~1.22、条数加到 430 之后，纹理变得均匀，只剩细腻的树皮颗粒感。

`bumpScale` 也不能给大。一开始设 0.12，在 25 米高、直径 6 米的树干上（LatheGeometry 40 段）产生了明显的带状伪影——几何精度不足以支撑那么强的凹凸。降到 0.05 就干净了。

## 四、噪声：地形和顶点扰动的地基

所有「自然感」的底层是一个 value noise：

```js
function hash2(x, y) {
  const s = Math.sin(x * 127.1 + y * 311.7) * 43758.5453123;
  return s - Math.floor(s);
}

function vnoise(x, y) {
  const xi = Math.floor(x), yi = Math.floor(y);
  const xf = x - xi, yf = y - yi;
  const u = xf * xf * (3 - 2 * xf);      // smoothstep 插值
  const v = yf * yf * (3 - 2 * yf);
  const a = hash2(xi, yi), b = hash2(xi+1, yi);
  const c = hash2(xi, yi+1), d = hash2(xi+1, yi+1);
  return (a*(1-u) + b*u)*(1-v) + (c*(1-u) + d*u)*v;
}
```

`fbm` 叠 4 个八度，喂给三个地方：

1. **地形高度场**——中心平缓、远处隆起的林地
2. **树干顶点**——`(vnoise - 0.5) * 0.14` 让旋转体表面不平整
3. **菌盖顶点**——同样的手法，让半球菌盖不那么「光滑的塑料感」

## 五、为什么值得

如果这个场景用了外部贴图，它大概需要 5 张 JPG（地面、树皮、天空、叶片、噪点），合计 2~4 MB。现在这 5 张图在页面加载后约 **180 ms** 内生成完毕，体积 0。

换来的是：
- 单文件分发，`git clone` 下来直接能开
- 改一个参数（比如让苔藓更多），整片地面跟着变，不用回 Photoshop
- 没有一张贴图会因为 CDN 挂掉而 404

代价是首次生成有约 0.2 秒的卡顿，所以我加了一个 loading 遮罩，生成完成后再淡出。

![](/assets/uploads/u_06e3b2ebf2d6ba15.jpg)

---

场景在线：<https://aowind.github.io/forest-ants/>
源码：<https://github.com/aowind/forest-ants>
