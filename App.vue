<template>
  <div>
    <VueMarkdown :content="msg" />
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
}
</script>
