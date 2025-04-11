<template>
  <div class="test-wrap w-[100vw] mt-[5vh] p-[5vw]">
    <div><img src="/" alt="" /></div>
    <!-- <ErButton /> -->
  </div>
</template>
<script setup>
import { onMounted, ref } from 'vue'
// import { ErButton } from 'toy-element2'
// import { ErButton } from '@eric-ui'

const canvasRef = ref(null)
const img = ref(null)
onMounted(() => {
  init()
  console.log('dev分支修改')
})
const init = () => {
  const res = encode({
    phone: '13657223213',
    real_name: '李林康',
    work_card: 'KK99054',
    redirect: '/home/chat/g/4d1760b4-c6ce-41f2-b837-7ecbf5a40527/s'
  })

  console.log('res', res)
}

const encode = (obj) => {
  for (const key in obj) {
    obj[key] = window.btoa(encodeURIComponent(obj[key]))
  }
  const str = `http://localhost:3000/eipLogin?phone=${obj.phone ?? ''}&real_name=${obj.real_name ?? ''}&work_card=${obj.work_card ?? ''}&redirect=${obj.redirect ?? ''}`
  return str
}
</script>

<style lang="scss" scoped>
.test-wrap {
  color: red;
  font-size: 20px;
  text-align: center;
}
</style>
