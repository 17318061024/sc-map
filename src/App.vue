<template>
  <div id="app-32-map" class="is-full"></div>
</template>

<script>
// 3D地图核心模块（封装Three.js基础功能）
import Map3d from '@/utils/Map3d.js';
// 补间动画库（用于创建平滑过渡动画）
import TWEEN from '@tweenjs/tween.js';
// Three.js核心库（WebGL 3D渲染引擎）
import * as THREE from 'three';
// 可视化调试面板（用于实时调整场景参数）
import { GUI } from 'three/examples/jsm/libs/lil-gui.module.min.js';

// Vue3生命周期钩子
import { onBeforeUnmount, onMounted } from 'vue';
// 工具函数库（包含随机数生成等方法）
import { random } from '@/utils';

// 自定义hooks👇
// 文件加载器（处理GeoJSON数据加载）
import useFileLoader from '@/hooks/useFileLoader.js';
// 国家边界处理模块（生成国界线几何体）
import useCountry from '@/hooks/useCountry.js';
// 坐标转换模块（WGS84 ↔ 平面坐标）
import useCoord from '@/hooks/useCoord.js';
// 地理数据标准化模块（统一不同来源的GeoJSON格式）
import useConversionStandardData from '@/hooks/useConversionStandardData.js';
// 光柱特效模块（创建地图标记点光柱效果）
import useMapMarkedLightPillar from '@/hooks/map/useMapMarkedLightPillar';
// 序列帧动画模块（处理粒子动画序列）
import useSequenceFrameAnimate from '@/hooks/useSequenceFrameAnimate';
// 2D标签渲染模块（实现CSS与WebGL混合渲染）
import useCSS2DRender from '@/hooks/useCSS2DRender';
import { linesData } from '../public/data/map/lines-data';

let centerXY = [106.59893798828125, 26.918846130371094];

export default {
  name: '3dMap30',
  setup() {
    let baseEarth = null;

    // 重置
    const resize = () => {
      baseEarth.resize();
    };

    const { requestData } = useFileLoader();
    const { transfromGeoJSON } = useConversionStandardData();
    const { getBoundingBox } = useCoord();
    const { createCountryFlatLine } = useCountry();
    const { initCSS2DRender, create2DTag } = useCSS2DRender();
    const { createLightPillar } = useMapMarkedLightPillar({
      scaleFactor: 1.2,
    });
    // 序列帧
    const { createSequenceFrame } = useSequenceFrameAnimate();

    const texture = new THREE.TextureLoader();
    // const textureMap = texture.load("/data/map/gz-map.jpg")
    const texturefxMap = texture.load('/data/map/gz-map-fx.jpg');
    const rotatingApertureTexture = texture.load(
      '/data/map/rotatingAperture.png'
    );
    const rotatingPointTexture = texture.load('/data/map/rotating-point2.png');
    const circlePoint = texture.load('/data/map/circle-point.png');
    const sceneBg = texture.load('/data/map/scene-bg2.png');

    // 创建Phong高光材质（用于3D模型顶部表面）
    const topFaceMaterial = new THREE.MeshPhongMaterial({
      color: 0x162537, // 基础颜色
      combine: THREE.MultiplyOperation, // 材质混合模式（颜色相乘增强层次感）
      transparent: true, // 启用透明度支持
      opacity: 1, // 完全不透明（1=100% 不透明）
    });
    const sideMaterial = new THREE.MeshLambertMaterial({
      color: 0x5c7699,
      transparent: true,
      opacity: 1,
    });
    const bottomZ = -0.2;
    // 可视化调试面板初始化函数
    const initGui = () => {
      // 创建GUI调试面板实例
      const gui = new GUI();

      // 调试参数配置对象
      const guiParams = {
        topColor: '#b40000', // 模型顶部颜色初始值
        sideColor: '#120000', // 模型侧面颜色初始值
        scale: 0.1, // 纹理缩放比例初始值
      };

      // 添加顶部颜色调节控件
      gui.addColor(guiParams, 'topColor').onChange((val) => {
        // 实时更新顶部材质颜色（HEX颜色值转换）
        topFaceMaterial.color = new THREE.Color(val);
      });

      // 添加侧面颜色调节控件
      gui.addColor(guiParams, 'sideColor').onChange((val) => {
        // 实时更新侧面材质颜色
        sideMaterial.color = new THREE.Color(val);
      });

      // 添加纹理缩放比例滑动条（范围0-1）
      gui.add(guiParams, 'scale', 0, 1).onChange((val) => {
        // 同步更新主纹理和特效纹理的重复比例
        textureMap.repeat.set(val, val); // 基础纹理
        texturefxMap.repeat.set(val, val); // 特效纹理
      });
    };
    // 初始化旋转光圈
    const initRotatingAperture = (scene, width) => {
      let plane = new THREE.PlaneBufferGeometry(width, width);
      let material = new THREE.MeshBasicMaterial({
        map: rotatingApertureTexture,
        transparent: true,
        opacity: 1,
        depthTest: true,
      });
      // 创建旋转光圈平面模型
      // 参数说明：plane=几何体，material=纹理材质
      let mesh = new THREE.Mesh(plane, material);

      // 设置光圈位置到地图中心点
      // 参数：...centerXY=地图中心点坐标，0=Z轴高度（地表层级）
      mesh.position.set(...centerXY, 0);

      // 缩放光圈尺寸（1.1倍于地图宽度）
      // 参数：x=1.1, y=1.1, z=1.1（XYZ轴等比缩放）
      mesh.scale.set(1.1, 1.1, 1.1);

      // 将光圈添加到3D场景中（使其可见）
      scene.add(mesh);
      return mesh;
    };
    // 初始化旋转点
    const initRotatingPoint = (scene, width) => {
      let plane = new THREE.PlaneBufferGeometry(width, width);
      let material = new THREE.MeshBasicMaterial({
        map: rotatingPointTexture,
        transparent: true,
        opacity: 1,
        depthTest: true,
      });
      let mesh = new THREE.Mesh(plane, material);
      mesh.position.set(...centerXY, bottomZ - 0.02);
      mesh.scale.set(1.1, 1.1, 1.1);
      scene.add(mesh);
      return mesh;
    };
    // 初始化背景
    const initSceneBg = (scene, width) => {
      let plane = new THREE.PlaneBufferGeometry(width * 4, width * 4);
      let material = new THREE.MeshPhongMaterial({
        // color: 0x061920,
        color: 0x383267,
        // map: sceneBg,
        transparent: true,
        opacity: 1,
        depthTest: true,
      });

      let mesh = new THREE.Mesh(plane, material);
      mesh.position.set(...centerXY, bottomZ - 0.2);
      scene.add(mesh);
    };
    // 初始化原点
    const initCirclePoint = (scene, width) => {
      let plane = new THREE.PlaneBufferGeometry(width, width);
      let material = new THREE.MeshPhongMaterial({
        color: 0x00ffff,
        map: circlePoint,
        transparent: true,
        opacity: 1,
        // depthTest: false,
      });
      let mesh = new THREE.Mesh(plane, material);
      mesh.position.set(...centerXY, bottomZ - 0.1);
      // let mesh2 = mesh.clone()
      // mesh2.position.set(...centerXY, bottomZ - 0.001)
      scene.add(mesh);
    };
    // 初始化粒子
    const initParticle = (scene, bound) => {
      // 获取中心点和中间地图大小
      let { center, size } = bound;
      // 构建范围，中间地图的2倍
      let minX = center.x - size.x;
      let maxX = center.x + size.x;
      let minY = center.y - size.y;
      let maxY = center.y + size.y;
      let minZ = -6;
      let maxZ = 6;

      let particleArr = [];
      for (let i = 0; i < 16; i++) {
        const particle = createSequenceFrame({
          image: './data/map/上升粒子1.png',
          width: 180,
          height: 189,
          frame: 9,
          column: 9,
          row: 1,
          speed: 0.5,
        });
        let particleScale = random(5, 10) / 1000;
        particle.scale.set(particleScale, particleScale, particleScale);
        particle.rotation.x = Math.PI / 2;
        let x = random(minX, maxX);
        let y = random(minY, maxY);
        let z = random(minZ, maxZ);
        particle.position.set(x, y, z);
        particleArr.push(particle);
      }
      scene.add(...particleArr);
      return particleArr;
    };
    // 创建顶部底部边线
    const initBorderLine = (data, mapGroup) => {
      let lineTop = createCountryFlatLine(
        data,
        {
          color: 0xffffff,
          linewidth: 0.001,
          transparent: true,
          depthTest: false,
        },
        'Line2'
      );
      lineTop.position.z += 8;
      // let lineBottom = createCountryFlatLine(
      //   data,
      //   {
      //     color: 0x99b2d3,
      //     linewidth: 0.2,
      //     // transparent: true,
      //     depthTest: false,
      //   },
      //   'Line2'
      // );
      // lineBottom.position.z -= 0.1905;
      //  添加边线
      mapGroup.add(lineTop);
      // mapGroup.add(lineBottom);
    };
    // 创建光柱
    const initLightPoint = (properties, mapGroup) => {
      if (!properties.centroid && !properties.center) {
        return false;
      }
      // 创建光柱
      let heightScaleFactor = 0.4 + random(1, 5) / 5;
      let lightCenter = properties.centroid || properties.center;
      let light = createLightPillar(...lightCenter, heightScaleFactor);
      light.position.z = 0.31;
      mapGroup.add(light);
    };
    // 创建标签
    const initLabel = (city, scene) => {
      if (!city.length) {
        return false;
      }
      city.forEach((item) => {
        // 设置标签的显示内容和位置
        var label = create2DTag('标签', 'map-32-label');
        scene.add(label);
        let labelCenter = item.value; //centroid || properties.center
        label.show(item.name, new THREE.Vector3(...labelCenter, 8));
      });
    };
    onMounted(async () => {
      // 四川数据
      let provinceData = await requestData('./data/map/四川省.json');
      provinceData = transfromGeoJSON(provinceData);
      //世界数据
      let worldData = await requestData('./data/map/world-map.json');
      worldData = transfromGeoJSON(worldData);
      // worldMap = transfromGeoJSON(worldMap);

      class CurrentMap3d extends Map3d {
        constructor(props) {
          super(props);
        }
        initCamera() {
          let { width, height } = this.options;
          let rate = width / height;
          // 设置45°的透视相机,更符合人眼观察
          this.camera = new THREE.PerspectiveCamera(45, rate, 0.001, 90000000);
          this.camera.up.set(0, 0, 1);
          // 贵州
          // this.camera.position.set(105.96420078859111, 20.405756412693812, 5.27483892390678) //相机在Three.js坐标系中的位置
          // 四川
          this.camera.position.set(0, -10, 250); //相机在Three.js坐标系中的位置
          this.camera.lookAt(...centerXY, 0);
        }
        initModel() {
          try {
            // 创建组
            this.mapGroup = new THREE.Group();
            // 标签 初始化
            this.css2dRender = initCSS2DRender(this.options, this.container);

            worldData.features.forEach((elem, index) => {
              // 定一个省份对象
              const city = new THREE.Object3D();
              // 坐标
              const coordinates = elem.geometry.coordinates;
              // city 属性
              const properties = elem.properties;

              // 循环坐标
              coordinates.forEach((multiPolygon) => {
                multiPolygon.forEach((polygon) => {
                  const shape = new THREE.Shape();
                  // 绘制shape
                  for (let i = 0; i < polygon.length; i++) {
                    let [x, y] = polygon[i];
                    if (i === 0) {
                      shape.moveTo(x, y);
                    }
                    shape.lineTo(x, y);
                  }
                  // 拉伸设置
                  const extrudeSettings = {
                    depth: 8, // 控制3D模型垂直拉伸高度（单位：坐标系单位）
                    bevelEnabled: true, // 启用边缘倒角效果（使模型边缘更圆润）
                    bevelSegments: 1, // 倒角分段数（值越大边缘越平滑）
                    bevelThickness: 0.1, // 倒角厚度（控制边缘斜面宽度）
                  };
                  const geometry = new THREE.ExtrudeGeometry(
                    shape,
                    extrudeSettings
                  );
                  const mesh = new THREE.Mesh(geometry, [
                    topFaceMaterial,
                    sideMaterial,
                  ]);

                  city.add(mesh);
                });
              });
              this.mapGroup.add(city);
              // 创建标点和标签
              // initLightPoint(properties, this.mapGroup);
              initLabel(linesData.city, this.scene);
            });
            // 创建上边框
            initBorderLine(worldData, this.mapGroup);

            let earthGroupBound = getBoundingBox(this.mapGroup);
            centerXY = [earthGroupBound.center.x, earthGroupBound.center.y];
            let { size } = earthGroupBound;
            let width = size.x < size.y ? size.y + 1 : size.x + 1;
            // 添加背景，修饰元素
            // this.rotatingApertureMesh = initRotatingAperture(this.scene, width);
            // this.rotatingPointMesh = initRotatingPoint(this.scene, width - 2);
            // initCirclePoint(this.scene, width);
            initSceneBg(this.scene, width);

            // 将组添加到场景中
            this.scene.add(this.mapGroup);
            // this.particleArr = initParticle(this.scene, earthGroupBound);
            initGui();
          } catch (error) {
            console.log(error);
          }
        }
        getDataRenderMap() {}

        destroy() {}
        initControls() {
          super.initControls();
          this.controls.target = new THREE.Vector3(...centerXY, 0);
        }
        initLight() {
          //   平行光1
          let directionalLight1 = new THREE.DirectionalLight(0x7af4ff, 1);
          directionalLight1.position.set(...centerXY, 30);
          //   平行光2
          let directionalLight2 = new THREE.DirectionalLight(0x7af4ff, 1);
          directionalLight2.position.set(...centerXY, 30);
          // 环境光
          let ambientLight = new THREE.AmbientLight(0x7af4ff, 1);
          // 将光源添加到场景中
          this.addObject(directionalLight1);
          this.addObject(directionalLight2);
          this.addObject(ambientLight);
        }
        initRenderer() {
          super.initRenderer();
          // this.renderer.outputEncoding = THREE.sRGBEncoding
        }
        loop() {
          this.animationStop = window.requestAnimationFrame(() => {
            this.loop();
          });
          // 这里是你自己业务上需要的code
          this.renderer.render(this.scene, this.camera);
          // 控制相机旋转缩放的更新
          if (this.options.controls.visibel && this.controls) {
            // this.controls.target.set(...centerXY, 0)
            this.controls.update();
          }
          // 统计更新
          if (this.options.statsVisibel) this.stats.update();
          if (this.rotatingApertureMesh) {
            this.rotatingApertureMesh.rotation.z += 0.0005;
          }
          if (this.rotatingPointMesh) {
            this.rotatingPointMesh.rotation.z -= 0.0005;
          }
          // 渲染标签
          if (this.css2dRender) {
            this.css2dRender.render(this.scene, this.camera);
          }
          // 粒子上升
          if (this.particleArr?.length) {
            for (let i = 0; i < this.particleArr.length; i++) {
              this.particleArr[i].updateSequenceFrame();
              this.particleArr[i].position.z += 0.01;
              if (this.particleArr[i].position.z >= 6) {
                this.particleArr[i].position.z = -6;
              }
            }
          }
          TWEEN.update();
          // console.log(this.camera.position)
        }
        resize() {
          // super.resize();
          // 这里是你自己业务上需要的code
          this.renderer.render(this.scene, this.camera);
          this.renderer.setPixelRatio(window.devicePixelRatio);

          if (this.css2dRender) {
            this.css2dRender.setSize(this.options.width, this.options.height);
          }
        }
      }
      baseEarth = new CurrentMap3d({
        container: '#app-32-map',
        axesVisibel: true,
        controls: {
          enableDamping: true, // 阻尼
          maxPolarAngle: (Math.PI / 2) * 0.98,
        },
      });
      baseEarth.run();
      window.addEventListener('resize', resize);
    });
    onBeforeUnmount(() => {
      window.removeEventListener('resize', resize);
    });
  },
};
</script>
<style>
html,
body,
#app,
.is-full {
  width: 100%;
  height: 100%;
  overflow: hidden;
}
.map-32-label {
  font-size: 10px;
  color: #fff;
}
</style>
