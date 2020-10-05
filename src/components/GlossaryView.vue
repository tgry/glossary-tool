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
            <b-form-textarea ref="name_textarea" placeholder="名前" rows="8" @change="changeNameTextArea" v-model="name_convert"></b-form-textarea>
          </b-col>
          <b-col class="convert-arrow-col">
            <b-icon-arrow-right font-scale="3"></b-icon-arrow-right>
          </b-col>
          <b-col>
            <b-form-textarea ref="value_textarea" placeholder="値" rows="8" v-model="value_convert"></b-form-textarea>
          </b-col>
        </b-row>
      </b-container>
    </b-collapse>
    <div><b-button block variant="primary" v-b-toggle.word-list>一覧</b-button></div>
    <b-collapse id="word-list" visible>
      <table class="word-table">
        <thead>
          <th>単語</th>
          <th>値</th>
          <th>説明</th>
          <th>編集</th>
          <th>削除</th>
        </thead>
        <tbody>
          <tr v-for=" word in words" :key="word.name">
            <td>{{word.name}}</td>
            <td>{{word.value}}</td>
            <td>{{word.description}}</td>
            <td><b-button @click="clickEditWordButton(word)" title="編集" v-b-tooltip.hover><b-icon-pen font-scale="1"></b-icon-pen></b-button></td>
            <td><b-button @click="clickRemoveWordButton(word)" title="削除" v-b-tooltip.hover><b-icon-trash font-scale="1"></b-icon-trash></b-button></td>
          </tr>
        </tbody>
      </table>
    </b-collapse>

    <!-- ダイアログ類 -->

    <!-- 設定 -->
    <b-modal id="glossary-config-dialog" title="個別設定">
      <div id="glossary-config-button-area">
        <b-button @click="clickImportGlossaryButton" title="ファイルからインポートします(既存の単語は削除されます)" v-b-tooltip.hover>インポート</b-button>
        <b-button @click="exportGlossaryFile" title="表示されている用語集をファイルへ保存します" v-b-tooltip.hover>エクスポート</b-button>
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

    <!-- エクスポート インポート用 -->
    <div id="glossary-file-select">
      <input type="file" ref="glossary_import_file">
      <a ref="glossary_export_link" href="#" download="glossarydata.js">export</a>
    </div>
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
      value_convert:'',
      words:[]
    }
  },
  methods: {
    setGlossary: function(doc) {
      // 用語集の設定
      this.glossary = doc ? {"_id":doc._id, "words":doc.words}: null;
      this.words = this.glossary.words;
      this.name_convert = '';
      this.value_convert = '';
    },
    clickImportGlossaryButton: function() {
      // インポートボタン押下
      let self = this;
      this.$refs.glossary_import_file.onchange = function() {
        self.importGlossaryFile();
      }
      this.$refs.glossary_import_file.click();
    },
    importGlossaryFile: function() {
      // インポート処理
      let file = this.$refs.glossary_import_file.files[0];
      let reader = new FileReader();
      let self = this;

      reader.onload = function() {
        let data = JSON.parse(reader.result);

        self.glossary.words = data[0].words;
        self.words = data[0].words;

        // 親に送る
        self.$emit('update-glossary', self.glossary);
      }

      reader.readAsText(file, 'utf-8');
      // ダイアログは閉じておく
      this.$bvModal.hide('app-config-dialog');
    },
    exportGlossaryFile: function() {
      // エクスポート実行
      let self = this;
      this.$pouch.get(
        this.glossary._id,
        {include_docs: true},
        'pouch_glossary'
      ).then(function(result) {
        let data = [];
        data.push({ "_id":result._id, "words":result.words });

        const reader = new FileReader();
        reader.onload = function() {
          // データを設定
          self.$refs.glossary_export_link.href='data:text/plain;charset=UTF-8;base64,' + reader.result.split(',')[1];
          // ダウンロードさせる
          self.$refs.glossary_export_link.click();
        }

        reader.readAsDataURL(new Blob([JSON.stringify(data)]));
      }).catch(function (err) {
        console.log(err);
      });

    },
    handleWordEdit: function() {
      let array = [];
      let self = this;

      // 単語追加変更
      if (this.current_word) {
        // 変更
        this.glossary.words.forEach(function(word){
          if (word.name == self.current_word.name) {
            word.name = self.edit_word_dialog_name;
            word.value = self.edit_word_dialog_value;
            word.description = self.edit_word_dialog_description;
          }
          array.push(word);
        });
      } else {
        // 追加
        this.glossary.words.forEach(function(word){
          array.push(word);
        });

        array.push(
          {
            name: self.edit_word_dialog_name,
            value: self.edit_word_dialog_value,
            description: self.edit_word_dialog_description
          }
        );
      }

      this.glossary.words = array;
      this.words = array;

      // ダイアログを閉じる
      this.$nextTick(function() {
        this.$bvModal.hide('edit-word-dialog');
      });
    
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
      this.words = array;

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

#glossary-config-button-area {
  width: 100%;
  text-align: center;
}

.convert-arrow-col {
  text-align: center;
  max-width: 5em;
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

/* 単語一覧テーブル */
.word-table {
  width: 100%;
}

.word-table thead {
  border-bottom:solid rgb(53, 57, 78) 1px;
}

.word-table thead th:nth-child(1) {
  width: 30%;
}

.word-table thead th:nth-child(2) {
  width: 30%;
}

.word-table thead th:nth-child(3) {
  width: 30%;
}

.word-table thead th:nth-child(4) {
  width: 5%;
}

.word-table thead th:nth-child(5) {
  width: 5%;
}

</style>