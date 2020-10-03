<template>
  <b-container>
    <b-row class="side-area">ツリー</b-row>
    <b-row cols="8">メイン</b-row>
  </b-container>
</template>

<script>
export default {
  name: 'GlossaryTool',
  props: {
    msg: String
  }
}
</script>

<style scoped>

.side-area {background-color: lightslategrey;}

</style>