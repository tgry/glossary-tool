<template>
  <div id="glossary-view" v-if="glossary">
    <h2>{{glossary._id}}</h2>
    <!-- 設定ボタン -->
    <b-button
      id="glossary-config-button"
      variant="outline-dark"
      v-b-tooltip.hover
      title="個別設定"
      v-b-modal.glossary-config-dialog
    >
      <b-icon-gear font-scale="2"></b-icon-gear>
    </b-button>

    <!-- 追加ボタン -->
    <b-button
      id="add-word-button"
      variant="outline-dark"
      v-b-tooltip.hover
      title="単語を作成"
      @click="clickAddWordButton"
    >
      <b-icon-plus font-scale="2"></b-icon-plus>
    </b-button>

    <div><b-button block variant="primary"  v-b-toggle.convert-words>一括変換</b-button></div>
    <b-collapse id="convert-words" visible>
      <b-container fluid class="convert-container">
        <b-row>
          <b-col>
            <b-form-textarea ref="name_textarea" @change="changeNameTextArea" v-model="name_convert"></b-form-textarea>
          </b-col>
          <b-col class="convert-arrow-col">
            <b-icon-arrow-bar-right font-scale="3"></b-icon-arrow-bar-right>
          </b-col>
          <b-col>
            <b-form-textarea ref="value_textarea" v-model="value_convert"></b-form-textarea>
          </b-col>
        </b-row>
      </b-container>
    </b-collapse>
    <div><b-button block variant="primary" v-b-toggle.word-list>一覧</b-button></div>
    <b-collapse id="word-list" visible>
      <b-container>
        <b-row class="word-list-header">
          <b-col>単語</b-col>
          <b-col>値</b-col>
          <b-col>説明</b-col>
          <b-col>編集</b-col>
          <b-col>削除</b-col>
        </b-row>
        <b-row v-for=" word in glossary.words" :key="word.name">
          <b-col>{{word.name}}</b-col>
          <b-col>{{word.value}}</b-col>
          <b-col>{{word.description}}</b-col>
          <b-col><b-button @click="clickEditWordButton(word)" title="編集" v-b-tooltip.hover><b-icon-pen font-scale="1"></b-icon-pen></b-button></b-col>
          <b-col><b-button @click="clickRemoveWordButton(word)" title="削除" v-b-tooltip.hover><b-icon-trash font-scale="1"></b-icon-trash></b-button></b-col>
        </b-row>
      </b-container>
    </b-collapse>
    <!-- ダイアログ類 -->

    <!-- 設定 -->
    <b-modal id="glossary-config-dialog" title="個別設定">
      <b-button @click="clickImportGlossaryButton" title="ファイルからインポートします(既存データは削除されます)">インポート</b-button>
      <b-button @click="exportGlossaryFile" title="表示されている用語集をファイルへ保存します">エクスポート</b-button>
      <div id="glossary-file-select">
        <input type="file" ref="glossary_import_file">
        <a ref="glossary_export_link" href="#" download="glossarydata.js">export</a>
      </div>
    </b-modal>

    <!-- 単語編集 -->
    <b-modal id="edit-word-dialog" @ok="handleWordEditDialogModalOk">
      <form ref="edit_word_form" @submit.stop.prevent="handleWordEdit" >
        <b-form-group label="単語" label-for="edit-name-form-input">
          <b-form-input id="edit-name-form-input" v-model="edit_word_dialog_name" required placeholder="単語"></b-form-input>
        </b-form-group>
        <b-form-group label="値" label-for="edit-value-form-input">
          <b-form-input id="edit-value-form-input" v-model="edit_word_dialog_value" required placeholder="値"></b-form-input>
        </b-form-group>
        <b-form-group label="説明" label-for="edit-description-form-input" >
          <b-form-input id="edit-description-form-input" v-model="edit_word_dialog_description" placeholder="説明"></b-form-input>
        </b-form-group>
      </form>
    </b-modal>
  </div>
</template>

<script>

export default {
  name: "GlossaryView",
  data: function() {
    return {
      glossary:null,
      edit_word_dialog_name:'',
      edit_word_dialog_value:'',
      edit_word_dialog_description:'',
      current_word:null,
      name_convert:'',
      value_convert:''
    }
  },
  methods: {
    setGlossary: function(doc) {
      // 用語集の設定
      this.glossary = doc ? {"_id":doc._id, "words":doc.words}: null;
      this.name_convert = '';
      this.value_convert = '';
    },
    clickImportGlossaryButton: function() {
      // インポートボタン押下

    },
    exportGlossaryFile: function() {
      // エクスポート実行

    },
    handleWordEdit: function() {
      // 単語追加変更
      if (this.current_word) {
        // 変更
        this.current_word.name = this.edit_word_dialog_name;
        this.current_word.value = this.edit_word_dialog_value;
        this.current_word.description = this.edit_word_dialog_description;
      } else {
        // 追加
        this.glossary.words.push(
          {
            "name": this.edit_word_dialog_name,
            "value": this.edit_word_dialog_value,
            "description": this.edit_word_dialog_description}
          );
      }

      // ダイアログを閉じる
      this.$bvModal.hide('edit-word-dialog');

      // 親に送る
      this.$emit('update-glossary', this.glossary);

    },
    handleWordEditDialogModalOk: function(bvModalEvt) {
      // ダイアログのOK
      bvModalEvt.preventDefault();
      this.handleWordEdit();
    },
    clickAddWordButton: function() {
      // 初期化
      this.current_word = null;
      this.edit_word_dialog_name = '';
      this.edit_word_dialog_value = '';
      this.edit_word_dialog_description = '';

      // ダイアログを表示
      this.$bvModal.show('edit-word-dialog');
    },
    clickEditWordButton: function(word) {
      this.current_word = word;
      this.edit_word_dialog_name = word.name;
      this.edit_word_dialog_value = word.value;
      this.edit_word_dialog_description = word.description;

      // ダイアログを表示
      this.$bvModal.show('edit-word-dialog');
    },
    clickRemoveWordButton: function(word) {
      // 削除
      let array = this.glossary.words.filter(function(element){
        return element.name != word.name
      });

      this.glossary.words = array;

      this.$emit('update-glossary', this.glossary);
    },
    changeNameTextArea: function() {
      // 一括変換
      let list = this.name_convert.split('\n');
      let result = [];
      let self = this;

      list.forEach(function(element){
        let value = null;
        self.glossary.words.forEach(function(word){
          if (!value && element == word.name) {
            value = word.value;
          }
        });

        result.push(value || '');
      });

      this.value_convert = result.join('\n');
    }
  }
}

</script>

<style scoped>
h2 {
  font-weight: bolder;
  color: darkblue;
  font-family: sans-serif;
}

#glossary-view {
  width: 100%;
  height: 100%;
  padding-left: 5%;
  padding-right: 5%;
  padding-top: 2%;
}

.convert-container {
  padding:5px;
}

#glossary-config-dialog button {
  margin: 30px;
}

.convert-arrow-col {
  text-align: center;
}

#glossary-config-button {
  position: absolute;
  top: 10px;
  right: 10px;
}

#add-word-button {
  position: absolute;
  top: 70px;
  right: 10px;
}

#glossary-file-select {
  display: none;
}
</style>