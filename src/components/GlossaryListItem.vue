<template>
  <div class="list-item">
    <!-- フォルダ用 -->
    <b-button v-b-toggle="'child-collapse-' + glossary.path" class="item-button" v-if="glossary.doc_type == 'folder'" variant="secondary">
      <b-icon-folder class="type-icon"></b-icon-folder>
      {{ glossary.name }}
    </b-button>
    <!-- 用語集用 -->
    <b-button v-if="glossary.doc_type == 'glossary'" class="item-button" variant="primary" @click="selectGlossary">
      <b-icon-book class="type-icon"></b-icon-book>
      {{ glossary.name }}
    </b-button>
    <b-button v-if="glossary.doc_type == 'glossary'" variant="primary" class="glossary-remove-button" @click="removeGlossary">
      <b-icon-trash></b-icon-trash>
    </b-button>
    <b-collapse :id="'child-collapse-' + glossary.path" class="mt-2" v-if="glossary.doc_type == 'folder'" visible>
      <glossary-list-item v-for="child in glossary.child" :key="child.path" :glossary="child" v-on:remove-glossary="sendRemoveGlossary" v-on:select-glossary="sendSelectGlossary"></glossary-list-item>
    </b-collapse>
  </div>
</template>

<script>
export default {
  name: "GlossaryListItem",
  props: {
    glossary: Object,
  },
  methods:{
    removeGlossary:function() {
      let self = this;
      // 確認ダイアログ
      this.$bvModal.msgBoxConfirm(this.glossary.path + 'を削除します。よろしいですか？').then(function(value) {
        if (!value) {
          return;
        } 
        // 親に伝える
        self.$emit('remove-glossary', self.glossary.doc);
      });
    },
    sendRemoveGlossary:function(doc) {
      // 親に伝える
      this.$emit('remove-glossary', doc);
    },
    sendSelectGlossary:function(doc) {
      // 親に伝える
      this.$emit('select-glossary', doc);
    },
    selectGlossary:function() {
      // 親に伝える
      this.$emit('select-glossary', this.glossary.doc);
    }
  }
};
</script>

<style scoped>
.list-item {
  padding-left: 30px;
  margin: 5px;
}

.type-icon {
  margin-right: 10px
}

.item-button {
  min-width: 200px;
}

.glossary-remove-button {
  margin-left: 10px;
}
</style>