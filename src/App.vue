<template>
  <Renderer ref="renderer" resize :orbit-ctrl="{ enableDamping: true, dampingFactor: 0.05 }" shadow>
    <Camera :position="{ y: 250, z: 250 }" />
    <Scene background="#08090d">
      
      <PointLight ref="light" :intensity="1" color="#ffff00" />
      <PointLight color="#ffffff" :intensity="0.3" :position="{ x: -LEN*INC**2, y: LEN*INC**2, z: LEN*INC**2 }" />
      <PointLight color="#ffffff" :intensity="0.3" :position="{ x: -LEN*INC**2, y: LEN*INC**2, z: -LEN*INC**2 }" />
      <InstancedMesh ref="imesh" :count="MAX**3">
        <BoxGeometry :size="SIZE" />
        <PhongMaterial />
      </InstancedMesh>
    </Scene>
    <EffectComposer>
      <RenderPass />
      <UnrealBloomPass :strength="0.5" :exposure="1" />
    </EffectComposer>
  </Renderer>
</template>

<script>
import { Object3D, Color, DynamicDrawUsage, Vector3 } from 'three';
import Stats from 'stats.js';
import * as dat from 'dat.gui';
import {
  Camera,
  EffectComposer,
  InstancedMesh,
  PhongMaterial,
  Renderer,
  RenderPass,
  BoxGeometry,
  PointLight,
  AmbientLight,
  Scene,
  UnrealBloomPass,
} from 'troisjs';
export default {
  components: {
    Camera,
    EffectComposer,
    InstancedMesh,
    PhongMaterial,
    Renderer,
    RenderPass,
    BoxGeometry,
    PointLight,
    AmbientLight,
    Scene,
    UnrealBloomPass,
  },
  setup() {
    let LEN = 25;
    
    let ARR = Array.from(Array(LEN), () => {
      return Array.from(Array(LEN), () => new Array(LEN).fill(false))
    })

    return {
      MAX: LEN,
      SIZE: 5,
      TIME: 1,
      SPEED: 1,
      INC: 10,
      INIT: false,
      LOOP: true,
      LEN,
      ARR
    };
  },
  mounted() {
    const gui = new dat.GUI();
    gui.add(this, 'LEN', 8, this.MAX, 1).name("Size").onChange(this.updateLen)
    gui.add(this, 'INC', 5, this.MAX, 1).name("Spacing").onChange(this.updateLen)
    gui.add(this, 'SPEED', 0, 10).name("Speed").onChange(this.updateSpeed)
    gui.add(this, 'LOOP').name("Loop").onChange(this.loop)
    
    this.stats = new Stats();
    this.stats.showPanel(0);
    document.body.appendChild( this.stats.dom );

    this.imesh = this.$refs.imesh.mesh;
    this.imesh.count = (this.MAX)**3
    this.imesh.instanceMatrix.setUsage( DynamicDrawUsage )
    
    this.renderer = this.$refs.renderer;
    this.renderer.onBeforeRender(this.animate);
    
    this.interval = window.setInterval(this.step, this.TIME*1000)
    this.start()
  },
  methods: {
    start() {
      window.clearInterval(this.interval);
      this.interval = setInterval(this.step, this.TIME*1000)
      this.ARR[4][5][4] = true;
      this.ARR[4][6][5] = true;
      this.ARR[5][6][4] = true;
      this.ARR[4][6][4] = true;
      this.ARR[3][6][4] = true;
      this.ARR[4][6][3] = true;
      this.ARR[4][7][4] = true;
      const dummy = new Object3D();
      let x,y,z;
      x = y = z = - (this.LEN / 2) * this.INC;
      this.imesh.count = (this.LEN)**3
      for (let i = 0; i < (this.LEN)**3; i++) {
        dummy.position.set(x, y, z);
        dummy.updateMatrix();
        this.imesh.setMatrixAt(i, dummy.matrix);
        x += this.INC;
        if (x == (this.LEN / 2) * this.INC) {
          x = - (this.LEN / 2) * this.INC;
          y += this.INC;
        }
        if (y == (this.LEN / 2) * this.INC) {
          y = - (this.LEN / 2) * this.INC;
          z += this.INC;
        }
      }
      this.imesh.instanceMatrix.needsUpdate = true;
    },
    neighbours(x, y, z) {
      x = parseInt(x);
      y = parseInt(y);
      z = parseInt(z);
      let n = [];
      for (let x1 = -1; x1 <= 1; x1 ++) {
        for (let y1 = -1; y1 <= 1; y1 ++) {
          for (let z1 = -1; z1 <= 1; z1 ++) {
            n.push(this.ARR[x + x1]?.[y + y1]?.[z + z1])
          }
        }
      }
      return n.filter(x => x).length;
    },
    animate() {
      this.stats.begin();

      let i = 0;
      for (let x in this.ARR) {
        for (let y in this.ARR) {
          for (let z in this.ARR) {
            if (this.ARR[x][y][z]) {
              this.imesh.setColorAt(i, new Color("#ffc400"))
            } else
              this.imesh.setColorAt(i, new Color("#7b10e6"))
            i ++
          }
        }
      }
      this.imesh.instanceColor.needsUpdate = true;
      this.stats.end();
    },
    step() {
      if (this.next) {
        this.next = false;
        this.start();
        return;
      }

      let todo = [];
      let count = 0;
      for (let x in this.ARR) {
        for (let y in this.ARR) {
          for (let z in this.ARR) {
            let n = this.neighbours(x, y, z);
            if (!this.ARR[x][y][z]) {
              if (n > 1 && n <= 2) todo.push([x, y, z])
            } else {
              if (n >= 3) {
                this.ARR[x][y][z] = false;
              }
            }
          }
        }
      }
      for (let [x, y, z] of todo) {
        this.ARR[x][y][z] = true;
        if (this.ARR[x][y][z]) count ++;
      }
      if (!count && this.LOOP) this.next = true;
    },
    updateLen() {
      this.ARR = Array.from(Array(this.LEN), () => {
        return Array.from(Array(this.LEN), () => new Array(this.LEN).fill(false))
      })
      this.start()
    },
    updateSpeed() {
      this.TIME = 1 / (this.SPEED + 0.1);
      window.clearInterval(this.interval);
      this.interval = setInterval(this.step, this.TIME*1000)
    }
  }
};
</script>

<style>
body, html {
  margin: 0;
  height: 100vh;
}
canvas {
  display: block;
  height: 100vh;
}
</style>
