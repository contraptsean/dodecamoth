<template>
<!--ui help buttons-->
<p :class="{'d-none': closeHelp}"><button type="button" class="btn" data-bs-toggle="modal" data-bs-target="#exampleModal">
  <i class="bi bi-question-octagon fs-1"></i>
</button><button type="button" class="btn-close btn-close-white" aria-label="Close" @click="closeHelper"></button></p>
  <!--canvas rendering-->
  <Renderer ref="renderer" resize :orbit-ctrl="{ enableDamping: true, dampingFactor: 0.05 }" pointer @click="randomColors">
    <Camera :position="{ z: 300 }" />
    <Scene>
      <PointLight ref="light" color="#ffc0c0" />

      <InstancedMesh ref="imesh" :count="NUM_INSTANCES">
        <DodecahedronGeometry :radius="5" />
        <SubSurfaceMaterial :props="{ vertexColors: true }" />
      </InstancedMesh>
    </Scene>
    <EffectComposer>
      <RenderPass />
      <UnrealBloomPass :strength="0.5" />
      <FXAAPass />
    </EffectComposer>
  </Renderer>
  <!--modal-->
  <!-- Modal -->
<div class="modal fade" id="exampleModal" tabindex="-1" aria-labelledby="exampleModalLabel" aria-hidden="true">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title text-gradient-blue" id="exampleModalLabel">It's dangerous to go alone! Take these moths..</h5>
        <button type="button" class="btn-close btn-close-white" data-bs-dismiss="modal" aria-label="Close"></button>
      </div>
      <div class="modal-body text-gradient-red">
        <span class="d-none d-md-block"><strong>Desktop Instructions</strong>
          <br /><i class="bi bi-mouse2"></i> The Moths will Swarm your Cursor
          <br /><i class="bi bi-mouse2"></i> Click to Change Color
          <br /><i class="bi bi-mouse2"></i> Drag to Pan
          <br /><i class="bi bi-mouse2"></i> Scroll to Zoom
        </span>
        <span class="d-block d-md-none"><strong>Mobile Instructions</strong>
          <br /><i class="bi bi-hand-index-fill"></i> Tap to Change Color
          <br /><i class="bi bi-hand-index-fill"></i> Tap to Set Light Point
          <br /><i class="bi bi-hand-index-fill"></i> Drag to Pan
          <br /><i class="bi bi-hand-index-fill"></i> Pinch to Zoom
          </span>


      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-gradient-border" data-bs-dismiss="modal">Close</button>
      </div>
    </div>
  </div>
</div>
</template>

<script>
import { ref } from 'vue'
import { Color, InstancedBufferAttribute , MathUtils, Object3D, Vector3 } from 'three';
import {
  Camera,
  DodecahedronGeometry,
  EffectComposer,
  FXAAPass,
  InstancedMesh,
  PointLight,
  Renderer,
  RenderPass,
  SubSurfaceMaterial,
  Scene,
  UnrealBloomPass,
} from 'troisjs';
import chroma from 'chroma-js';
const { randFloat: rnd, randFloatSpread: rndFS } = MathUtils;
export default {
  components: {
    Camera,
    DodecahedronGeometry,
    EffectComposer,
    FXAAPass,
    InstancedMesh,
    PointLight,
    Renderer,
    RenderPass,
    SubSurfaceMaterial,
    Scene,
    UnrealBloomPass,
  },
  setup() {
    const NUM_INSTANCES = 2000;
    const instances = [];
    const cscale = chroma.scale(['#dd3e1b', '#0b509c']);
    const target = new Vector3();
    const dummyO = new Object3D();
    const dummyV = new Vector3();
    for (let i = 0; i < NUM_INSTANCES; i++) {
      instances.push({
        position: new Vector3(rndFS(200), rndFS(200), rndFS(200)),
        scale: rnd(0.2, 1),
        velocity: new Vector3(rndFS(2), rndFS(2), rndFS(2)),
        attraction: 0.0025 + rnd(0, 0.01),
        vlimit: 0.3 + rnd(0, 0.2),
      });
    var closeHelp = ref(false);
    }
    return {
      NUM_INSTANCES,
      instances,
      cscale,
      target,
      dummyO,
      dummyV,
      closeHelp
    };
  },
  mounted() {
    this.renderer = this.$refs.renderer;
    this.imesh = this.$refs.imesh.mesh;
    this.light = this.$refs.light.light;
    this.init();
  },
  methods: {
    init() {
      // init instanced mesh matrix
      for (let i = 0; i < this.NUM_INSTANCES; i++) {
        const { position, scale } = this.instances[i];
        this.dummyO.position.copy(position);
        this.dummyO.scale.set(scale, scale, scale);
        this.dummyO.updateMatrix();
        this.imesh.setMatrixAt(i, this.dummyO.matrix);
      }
      this.updateColors();
      this.imesh.instanceMatrix.needsUpdate = true;
      // animate
      this.renderer.onBeforeRender(this.animate);
    },
    animate() {
      const { pointer } = this.renderer.three;
      this.target.copy(pointer.positionV3)
      this.light.position.copy(this.target);
      for (let i = 0; i < this.NUM_INSTANCES; i++) {
        const { position, scale, velocity, attraction, vlimit } = this.instances[i];
        this.dummyV.copy(this.target).sub(position).normalize().multiplyScalar(attraction);
        velocity.add(this.dummyV).clampScalar(-vlimit, vlimit);
        position.add(velocity);
        this.dummyO.position.copy(position);
        this.dummyO.scale.set(scale, scale, scale);
        this.dummyO.lookAt(this.dummyV.copy(position).add(velocity));
        this.dummyO.updateMatrix();
        this.imesh.setMatrixAt(i, this.dummyO.matrix);
      }
      this.imesh.instanceMatrix.needsUpdate = true;
    },
    randomColors() {
      const c1 = chroma.random(), c2 = chroma.random();
      this.cscale = chroma.scale([c1, c2]);
      this.updateColors();
    },
    updateColors() {
      const colors = [];
      for (let i = 0; i < this.NUM_INSTANCES; i++) {
        const color = new Color(this.cscale(rnd(0, 1)).hex());
        colors.push(color.r, color.g, color.b);
      }
      this.imesh.geometry.setAttribute('color', new InstancedBufferAttribute(new Float32Array(colors), 3));
    },
    closeHelper() {
      this.closeHelp = !this.closeHelp;
    }
  },
};
</script>
<style>
body {
  margin: 0;
}
canvas {
  display: block;
  min-height: 100vh;
  max-width: 100vw;

}
p {
  position:absolute;
  z-index: 1;
  top:1rem;
  right:5rem;
  display:flex;
  align-items:center;
  justify-content:space-around;
}
p .btn {
  color:#ffc0c0;
}
p .btn:hover {
  color:#bbb;
}
p .btn:active,
p .btn:focus {
  outline:none;
  box-shadow: none;
  border-bottom: 3px solid #ddd;
}
.modal-content {
  background-color:#1c2333;
  color:#eee;
}
.text-gradient-red {
  background: linear-gradient(to right, #ffc0c0, #F27121);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}
.text-gradient-blue {
  background: linear-gradient(to right,  #00c0c0, #ffc0c0);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}
.btn-gradient-border {
  color: #fff;
  border: 2px double transparent;
  background-image: linear-gradient(rgb(13, 14, 33), rgb(13, 14, 33)), radial-gradient(circle at left top, rgb(1, 110, 218), rgb(255, 100, 192));
  background-origin: border-box;
  background-clip: padding-box, border-box;
}

.btn-gradient-border:hover {
  color:#ccc;
}
</style>