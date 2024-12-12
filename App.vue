<template>
  <div>
    <VueMarkdown :content="msg" />
    <button hover-class="button-hover" @click="test">sdsadas</button>
  </div>
</template>
<script>
import VueMarkdown from './lib/index.vue'
export default {
  name: 'App',
  components: {
    VueMarkdown,
  },
  data() {
    return {
      msg: '### Hello, world! \n\n ## This is a test',
    }
  },
  methods: {
    test() {
      console.log('test')
      this.msg = '### Hello! \n\n ## This is a test'
    },
  },
}
</script>
