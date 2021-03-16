Title: Three.js 阴影
Description: 在Three.js中使用阴影
TOC: 阴影

本文是关于 three.js 的系列文章的一部分。第一篇文章是[Three.js 基础](threejs-fundamentals.html)。
如果你还没有读过这篇文章，而且是 three.js 的新手，那么你可以考虑从第一篇文章开始。
上一篇文章是关于[相机](threejs-cameras.html)的，在阅读这篇文章之前最好是先读过这篇文章以及关于
[灯光](threejs-lights.html)的文章。

计算机中的阴影是一个复杂的话题。有各种各样的解决方案，所有这些方案都有各自的优缺点，包括 three.js 中提供的解决方案。

Three.js 默认使用 *阴影贴图*。阴影贴图的工作原理是，对于每一个产生投射阴影的光线，所有标记产生阴影的物体都从光线的
角度进行渲染。
**再读一遍**。让它沉淀下去。

换句话说，如果你有20个物体，5个灯光，所有20个物体都投射阴影，所有5个灯光都造成阴影，那么你的整个场景将被绘制6次。
所有20个物体将绘经由light#1绘制。然后是light#2 然后light#3 以此类推。最后实际的场景将由之前的5个渲染数据绘制而成。

更糟糕的是,如果你有一个点光源会造成阴影的场景，该场景也必须绘制6次。只因为那个光！

由于这些原因，寻找其他的解决方案是很常见的，而不是让一堆光产生阴影。一个常见的解决方案是有多个
灯光，但只有一个方向的光产生阴影。

另一个解决方案是使用光照贴图或环境遮挡贴图来预先计算离线照明的效果。这导致静态照明或静态照明提
示，但至少它的快速。我们将在另一篇文章中讨论这两个问题。

另一个解决方案是使用假阴影。制造一个平面，在这个平面上放置一个近似阴影的灰度纹理，把它画在对象下
方的地面上。

例如，让我们使用这个纹理作为一个假的阴影

<div class="threejs_center"><img src="../resources/images/roundshadow.png"></div>

我们将使用[前一篇文章](threejs-cameras.html)中的一些代码。

让我们设置背景颜色为白色。

```js
const scene = new THREE.Scene();
+scene.background = new THREE.Color('white');
```

然后我们将设置相同的棋盘格地面，但这次它使用 `MeshBasicMaterial`，因为我们不需要地面照明。

```js
+const loader = new THREE.TextureLoader();

{
  const planeSize = 40;

-  const loader = new THREE.TextureLoader();
  const texture = loader.load('resources/images/checker.png');
  texture.wrapS = THREE.RepeatWrapping;
  texture.wrapT = THREE.RepeatWrapping;
  texture.magFilter = THREE.NearestFilter;
  const repeats = planeSize / 2;
  texture.repeat.set(repeats, repeats);

  const planeGeo = new THREE.PlaneGeometry(planeSize, planeSize);
  const planeMat = new THREE.MeshBasicMaterial({
    map: texture,
    side: THREE.DoubleSide,
  });
+  planeMat.color.setRGB(1.5, 1.5, 1.5);
  const mesh = new THREE.Mesh(planeGeo, planeMat);
  mesh.rotation.x = Math.PI * -.5;
  scene.add(mesh);
}
```

注意，我们将颜色设置为`1.5,1.5,1.5`。这将棋盘纹理的颜色乘以`1.5,1.5,1.5`。由于纹理的颜色是
0x808080和0xC0C0C0，这是中灰色和浅灰色，乘以1.5将给我们一个白色和浅灰色棋盘。

让我们加载这个阴影纹理

```js
const shadowTexture = loader.load('resources/images/roundshadow.png');
```

创建一个数组来记住每个球体和相关对象。

```js
const sphereShadowBases = [];
```

然后我们做一个球体

```js
const sphereRadius = 1;
const sphereWidthDivisions = 32;
const sphereHeightDivisions = 16;
const sphereGeo = new THREE.SphereGeometry(sphereRadius, sphereWidthDivisions, sphereHeightDivisions);
```

和一个平面来放假阴影

```js
const planeSize = 1;
const shadowGeo = new THREE.PlaneGeometry(planeSize, planeSize);
```

现在我们要做一堆的球体。对于每个球我们创建一个`THREE.Object3D`作为基础，同时创建阴影平面网格
和球面网格。这样，如果我们移动`THREE.Object3D`，球体和阴影都会移动。我们需要把阴影稍微地面以上
防止z方向的穿透。我们还将 depthWrite 设置为 false，这样阴影就不会彼此混淆。我们将在[另一篇文章]
(threejs-transparency.html)中讨论这两个问题。阴影是一个 `MeshBasicMaterial`，因为它不
需要照明。

我们使每个球体具有不同的色调，然后保存基体，球体网格，阴影网格和每个球体的初始 y 位置。

```js
const numSpheres = 15;
for (let i = 0; i < numSpheres; ++i) {
  // make a base for the shadow and the sphere
  // so they move together.
  const base = new THREE.Object3D();
  scene.add(base);

  // add the shadow to the base
  // note: we make a new material for each sphere
  // so we can set that sphere's material transparency
  // separately.
  const shadowMat = new THREE.MeshBasicMaterial({
    map: shadowTexture,
    transparent: true,    // so we can see the ground
    depthWrite: false,    // so we don't have to sort
  });
  const shadowMesh = new THREE.Mesh(shadowGeo, shadowMat);
  shadowMesh.position.y = 0.001;  // so we're above the ground slightly
  shadowMesh.rotation.x = Math.PI * -.5;
  const shadowSize = sphereRadius * 4;
  shadowMesh.scale.set(shadowSize, shadowSize, shadowSize);
  base.add(shadowMesh);

  // add the sphere to the base
  const u = i / numSpheres;   // goes from 0 to 1 as we iterate the spheres.
  const sphereMat = new THREE.MeshPhongMaterial();
  sphereMat.color.setHSL(u, 1, .75);
  const sphereMesh = new THREE.Mesh(sphereGeo, sphereMat);
  sphereMesh.position.set(0, sphereRadius + 2, 0);
  base.add(sphereMesh);

  // remember all 3 plus the y position
  sphereShadowBases.push({base, sphereMesh, shadowMesh, y: sphereMesh.position.y});
}
```

我们设置了两个灯，一个是半球灯，亮度设置为2，这样可以让东西更亮一些。

```js
{
  const skyColor = 0xB1E1FF;  // light blue
  const groundColor = 0xB97A20;  // brownish orange
  const intensity = 2;
  const light = new THREE.HemisphereLight(skyColor, groundColor, intensity);
  scene.add(light);
}
```

另一个是`DirectionalLight`，这样球体就有了一些定义

```js
{
  const color = 0xFFFFFF;
  const intensity = 1;
  const light = new THREE.DirectionalLight(color, intensity);
  light.position.set(0, 10, 5);
  light.target.position.set(-5, 0, 0);
  scene.add(light);
  scene.add(light.target);
}
```

它会渲染成现在的样子，但是让我们使小球们动起来。对于每个球体，阴影，基体，我们在xz平面中移动
基体，我们使用 Math.abs (Math.sin (time))来上下移动球体，这会给我们一个弹性动画。并且，
我们还设置了阴影材质的不透明度，这样当每个球体的高度越高，它的阴影就会渐渐消失。

```js
function render(time) {
  time *= 0.001;  // convert to seconds

  ...

  sphereShadowBases.forEach((sphereShadowBase, ndx) => {
    const {base, sphereMesh, shadowMesh, y} = sphereShadowBase;

    // u is a value that goes from 0 to 1 as we iterate the spheres
    const u = ndx / sphereShadowBases.length;

    // compute a position for the base. This will move
    // both the sphere and its shadow
    const speed = time * .2;
    const angle = speed + u * Math.PI * 2 * (ndx % 1 ? 1 : -1);
    const radius = Math.sin(speed - ndx) * 10;
    base.position.set(Math.cos(angle) * radius, 0, Math.sin(angle) * radius);

    // yOff is a value that goes from 0 to 1
    const yOff = Math.abs(Math.sin(time * 2 + ndx));
    // move the sphere up and down
    sphereMesh.position.y = y + THREE.MathUtils.lerp(-2, 2, yOff);
    // fade the shadow as the sphere goes up
    shadowMesh.material.opacity = THREE.MathUtils.lerp(1, .25, yOff);
  });

  ...
```

这里有15种弹跳球。

{{{example url="../threejs-shadows-fake.html" }}}

在一些应用程序中，通常对所有东西都使用圆形或椭圆形阴影，当然你也可以使用不同形状的阴影纹理。你也
可以给阴影一个更硬的边缘。使用这种类型的阴影的一个很好的例子是[动物之森](https://www.google.com/search?tbm=isch&q=animal+crossing+pocket+camp+screenshots)，在这里你可以看到每个角色
都有一个简单的圆形阴影。这是有效和‘廉价’的。[纪念碑谷](https://www.google.com/search?q=monument+valley+screenshots&tbm=isch)似乎也用这种阴影作为主角。

So, moving on to shadow maps, there are 3 lights which can cast shadows. The `DirectionalLight`,
the `PointLight`, and the `SpotLight`.

Let's start with the `DirectionalLight` with the helper example from [the lights article](threejs-lights.html).

The first thing we need to do is turn on shadows in the renderer.

```js
const renderer = new THREE.WebGLRenderer({canvas});
+renderer.shadowMap.enabled = true;
```

Then we also need to tell the light to cast a shadow

```js
const light = new THREE.DirectionalLight(color, intensity);
+light.castShadow = true;
```

We also need to go to each mesh in the scene and decide if it should
both cast shadows and/or receive shadows.

Let's make the plane (the ground) only receive shadows since we don't
really care what happens underneath.

```js
const mesh = new THREE.Mesh(planeGeo, planeMat);
mesh.receiveShadow = true;
```

For the cube and the sphere let's have them both receive and cast shadows

```js
const mesh = new THREE.Mesh(cubeGeo, cubeMat);
mesh.castShadow = true;
mesh.receiveShadow = true;

...

const mesh = new THREE.Mesh(sphereGeo, sphereMat);
mesh.castShadow = true;
mesh.receiveShadow = true;
```

And then we run it.

{{{example url="../threejs-shadows-directional-light.html" }}}

What happened? Why are parts of the shadows missing?

The reason is shadow maps are created by rendering the scene from the point
of view of the light. In this case there is a camera at the `DirectionalLight`
that is looking at its target. Just like [the camera's we previously covered](threejs-cameras.html)
the light's shadow camera defines an area inside of which
the shadows get rendered. In the example above that area is too small.

In order to visualize that area we can get the light's shadow camera and add
a `CameraHelper` to the scene.

```js
const cameraHelper = new THREE.CameraHelper(light.shadow.camera);
scene.add(cameraHelper);
```

And now you can see the area for which shadows are cast and received.

{{{example url="../threejs-shadows-directional-light-with-camera-helper.html" }}}

Adjust the target x value back and forth and it should be pretty clear that only
what's inside the light's shadow camera box is where shadows are drawn.

We can adjust the size of that box by adjusting the light's shadow camera.

Let's add some GUI setting to adjust the light's shadow camera box. Since a
`DirectionalLight` represents light all going in a parallel direction, the
`DirectionalLight` uses an `OrthographicCamera` for its shadow camera.
We went over how an `OrthographicCamera` works in [the previous article about cameras](threejs-cameras.html).

Recall an `OrthographicCamera` defines
its box or *view frustum* by its `left`, `right`, `top`, `bottom`, `near`, `far`,
and `zoom` properties.

Again let's make a helper class for the dat.GUI. We'll make a `DimensionGUIHelper`
that we'll pass an object and 2 properties. It will present one property that dat.GUI
can adjust and in response will set the two properties one positive and one negative.
We can use this to set `left` and `right` as `width` and `up` and `down` as `height`.

```js
class DimensionGUIHelper {
  constructor(obj, minProp, maxProp) {
    this.obj = obj;
    this.minProp = minProp;
    this.maxProp = maxProp;
  }
  get value() {
    return this.obj[this.maxProp] * 2;
  }
  set value(v) {
    this.obj[this.maxProp] = v /  2;
    this.obj[this.minProp] = v / -2;
  }
}
```

We'll also use the `MinMaxGUIHelper` we created in the [camera article](threejs-cameras.html)
to adjust `near` and `far`.

```js
const gui = new GUI();
gui.addColor(new ColorGUIHelper(light, 'color'), 'value').name('color');
gui.add(light, 'intensity', 0, 2, 0.01);
+{
+  const folder = gui.addFolder('Shadow Camera');
+  folder.open();
+  folder.add(new DimensionGUIHelper(light.shadow.camera, 'left', 'right'), 'value', 1, 100)
+    .name('width')
+    .onChange(updateCamera);
+  folder.add(new DimensionGUIHelper(light.shadow.camera, 'bottom', 'top'), 'value', 1, 100)
+    .name('height')
+    .onChange(updateCamera);
+  const minMaxGUIHelper = new MinMaxGUIHelper(light.shadow.camera, 'near', 'far', 0.1);
+  folder.add(minMaxGUIHelper, 'min', 0.1, 50, 0.1).name('near').onChange(updateCamera);
+  folder.add(minMaxGUIHelper, 'max', 0.1, 50, 0.1).name('far').onChange(updateCamera);
+  folder.add(light.shadow.camera, 'zoom', 0.01, 1.5, 0.01).onChange(updateCamera);
+}
```

We tell the GUI to call our `updateCamera` function anytime anything changes.
Let's write that function to update the light, the helper for the light, the
light's shadow camera, and the helper showing the light's shadow camera.

```js
function updateCamera() {
  // update the light target's matrixWorld because it's needed by the helper
  light.target.updateMatrixWorld();
  helper.update();
  // update the light's shadow camera's projection matrix
  light.shadow.camera.updateProjectionMatrix();
  // and now update the camera helper we're using to show the light's shadow camera
  cameraHelper.update();
}
updateCamera();
```

And now that we've given the light's shadow camera a GUI we can play with the values.

{{{example url="../threejs-shadows-directional-light-with-camera-gui.html" }}}

Set the `width` and `height` to about 30 and you can see the shadows are correct
and the areas that need to be in shadow for this scene are entirely covered.

But this brings up the question, why not just set `width` and `height` to some
giant numbers to just cover everything? Set the `width` and `height` to 100
and you might see something like this

<div class="threejs_center"><img src="resources/images/low-res-shadow-map.png" style="width: 369px"></div>

What's going on with these low-res shadows?!

This issue is yet another shadow related setting to be aware of.
Shadow maps are textures the shadows get drawn into.
Those textures have a size. The shadow camera's area we set above is stretched
across that size. That means the larger area you set, the more blocky your shadows will
be.

You can set the resolution of the shadow map's texture by setting `light.shadow.mapSize.width`
and `light.shadow.mapSize.height`. They default to 512x512.
The larger you make them the more memory they take and the slower they are to compute so you want
to set them as small as you can and still make your scene work. The same is true with the
light's shadow camera area. Smaller means better looking shadows so make the area as small as you
can and still cover your scene. Be aware that each user's machine has a maximum texture size
allowed which is available on the renderer as [`renderer.capabilities.maxTextureSize`](WebGLRenderer.capabilities).

<!--
Ok but what about `near` and `far` I hear you thinking. Can we set `near` to 0.00001 and far to `100000000`
-->

Switching to the `SpotLight` the light's shadow camera becomes a `PerspectiveCamera`. Unlike the `DirectionalLight`'s shadow camera
where we could manually set most its settings, `SpotLight`'s shadow camera is controlled by the `SpotLight` itself. The `fov` for the shadow
camera is directly connected to the `SpotLight`'s `angle` setting.
The `aspect` is set automatically based on the size of the shadow map.

```js
-const light = new THREE.DirectionalLight(color, intensity);
+const light = new THREE.SpotLight(color, intensity);
```

and we added back in the `penumbra` and `angle` settings
from our [article about lights](threejs-lights.html).

{{{example url="../threejs-shadows-spot-light-with-camera-gui.html" }}}

<!--
You can notice, just like the last example if we set the angle high
then the shadow map, the texture is spread over a very large area and
the resolution of our shadows gets really low.

div class="threejs_center"><img src="resources/images/low-res-shadow-map-spotlight.png" style="width: 344px"></div>

You can increase the size of the shadow map as mentioned above. You can
also blur the result

{{{example url="../threejs-shadows-spot-light-with-shadow-radius" }}}
-->


And finally there's shadows with a `PointLight`. Since a `PointLight`
shines in all directions the only relevant settings are `near` and `far`.
Otherwise the `PointLight` shadow is effectively 6 `SpotLight` shadows
each one pointing to the face of a cube around the light. This means
`PointLight` shadows are much slower since the entire scene must be
drawn 6 times, one for each direction.

Let's put a box around our scene so we can see shadows on the walls
and ceiling. We'll set the material's `side` property to `THREE.BackSide`
so we render the inside of the box instead of the outside. Like the floor
we'll set it only to receive shadows. Also we'll set the position of the
box so its bottom is slightly below the floor so the floor and the bottom
of the box don't z-fight.

```js
{
  const cubeSize = 30;
  const cubeGeo = new THREE.BoxGeometry(cubeSize, cubeSize, cubeSize);
  const cubeMat = new THREE.MeshPhongMaterial({
    color: '#CCC',
    side: THREE.BackSide,
  });
  const mesh = new THREE.Mesh(cubeGeo, cubeMat);
  mesh.receiveShadow = true;
  mesh.position.set(0, cubeSize / 2 - 0.1, 0);
  scene.add(mesh);
}
```

And of course we need to switch the light to a `PointLight`.

```js
-const light = new THREE.SpotLight(color, intensity);
+const light = new THREE.PointLight(color, intensity);

....

// so we can easily see where the point light is
+const helper = new THREE.PointLightHelper(light);
+scene.add(helper);
```

{{{example url="../threejs-shadows-point-light.html" }}}

Use the `position` GUI settings to move the light around
and you'll see the shadows fall on all the walls. You can
also adjust `near` and `far` settings and see just like
the other shadows when things are closer than `near` they
no longer receive a shadow and they are further than `far`
they are always in shadow.

<!--
self shadow, shadow acne
-->

