<template>
  <vueson-editor v-model="json" @json-change="onJsonChange"></vueson-editor>
</template>

<script>
import vuesonEditor from '../vueson-editor.vue'
export default {
  data () {
    return {
      json: [
        {"testfield": 123},
        {"testfield": 2323, "anotherfield": "test"}
      ]
    }
  },

  components: {
    vuesonEditor
  },

  methods: {
    onJsonChange (value) {
      console.log('value:', value);
    }
  }
}
</script>
