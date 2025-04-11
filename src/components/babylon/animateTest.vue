<template>
  <div class="animateTest-container">
    <canvas class="bbl-canvas" tabindex="1"></canvas>
  </div>
</template>
<script setup>
import BBL from '@/components/babylon/createBBL.js'
import * as BABYLON from 'babylonjs'
import { onMounted } from 'vue'

const create3d = () => {
  // 创建BBL实例
  const bbl = new BBL('.animateTest-container')
  // bbl.createAxis()
  // const ground = bbl.createGround(20, 20)
  // const ball = bbl.createBall(64, 1)
  // ball.position.y = 5
  // 灯光
  // bbl.addLight(0, 20, 0, 0.5)
  // bbl.addLight(20, 20, 20, 0.5)
  // bbl.addLight(-20, 20, -20, 0.5)
  // 动画
  // const boxAni = new BABYLON.Animation('ani', 'rotation.y', 30, BABYLON.Animation.ANIMATIONTYPE_FLOAT, BABYLON.Animation.ANIMATIONLOOPMODE_CYCLE)
  // const aniKeys = [
  //   { frame: 0, value: 0 },
  //   { frame: 60, value: 2 * Math.PI }
  // ]
  // boxAni.setKeys(aniKeys)
  // box.animations = [boxAni]
  // bbl.scene.beginAnimation(box, 0, 60, true)
  // 材质
  // bbl.createGroundMat(ground)
  // bbl.createMetalMat(box)
  // bbl.createMetalMat(ball)
  // bbl.createAsphalt(ground)
  // bbl.createAsphalt(ball)
  bbl.createEnv()
  // 模型导入
  bbl.createBarrel('wall', 1)

  // 创建半球形光线
  bbl.createHemiLight(1, false)
  // 创建点光源
  // bbl.createPointLight(true)
  // 创建平行光
  // const pointLight = bbl.createDirectLight(true)
  // pointLight.parent = ball
  // 创建聚光灯
  // const spotLight = bbl.createSpotLight({ px: 0, py: 10, pz: 0 }, { dx: 0, dy: -1, dz: 0 }, Math.PI / 2, 10, true, [ball, ground])
  // 交互行为
  // bbl.createActions(ball)
  // 碰撞模拟，物理引擎cannon
  bbl.createImpostors()
}
onMounted(() => {
  create3d()
  console.log('master分支修改')
})
</script>

<style lang="scss" scoped>
.animateTest-container {
  .bbl-canvas {
    // transform: scale(1.04);
  }
}
</style>
