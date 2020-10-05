<template>
  <div>
    <b-container class="app-area" fluid>
      <b-row class="app-area">
        <b-col id="list-area">
          <h1>Glossary-Tool</h1>

          <!-- 設定ボタン -->
          <b-button
            id="app-config-button"
            variant="outline-light"
            v-b-tooltip.hover
            title="全体設定"
            v-b-modal.app-config-dialog
          >
            <b-icon-gear font-scale="2"></b-icon-gear>
          </b-button>

          <!-- 追加ボタン -->
          <b-button
            id="add-glossary-button"
            variant="outline-light"
            v-b-tooltip.hover
            title="用語集を作成"
            v-b-modal.add-glossary-dialog
          >
            <b-icon-plus font-scale="2"></b-icon-plus>
          </b-button>

          <p v-if="!loaded">
            <b-spinner label="Loading" variant="light"></b-spinner> 読み込み中…
          </p>
          <p id="list-empty" v-if="glossary_list.length == 0 && loaded">
            作成済みの用語集がありません<br />
            <b-button v-b-modal.add-glossary-dialog>作成</b-button>
            <b-button @click="clickImportButton">インポート</b-button>
          </p>
          <p id="glossary-list" v-if="glossary_list.length != 0">
            <!-- リスト -->
            <glossary-list-item v-for="glossary in glossary_list" :key="glossary.path" :glossary="glossary" v-on:remove-glossary="removeGlossary" v-on:select-glossary="selectGlossary"></glossary-list-item>
          </p>
        </b-col>
        <b-col id="glossary-area" cols="9">
          <!-- 用語集の表示 -->
          <glossary-view ref="glossary_view" v-on:remove-glossary="removeGlossary" v-on:update-glossary="updateGlossary"></glossary-view>
        </b-col>
      </b-row>
    </b-container>

    <!-- ダイアログ類 -->

    <!-- 設定 -->
    <b-modal id="app-config-dialog" title="全体設定" @show="handleAppConfigDialogModalShow" @ok="handleAppConfigDialogModalOk">
      <div id="app-config-button-area">
        <b-button @click="clickImportButton" title="ファイルからインポートします(既存データは削除されます)" v-b-tooltip.hover>インポート</b-button>
        <b-button @click="exportAppFile" title="すべての用語集をファイルへ保存します" v-b-tooltip.hover>エクスポート</b-button>
      </div>
      <form ref="app_config_form">
        <b-form-group label="CouchDBのURL" label-for="app-config-form-url-input" >
          <b-form-input id="app-config-form-url-input" v-model="config_dialog_url" required></b-form-input>
        </b-form-group>
        <b-form-group label="CouchDBのユーザ" label-for="app-config-form-user-input">
          <b-form-input id="app-config-form-user-input" v-model="config_dialog_user" required></b-form-input>
        </b-form-group>
        <b-form-group label="CouchDBのパスワード" label-for="app-config-form-password-input" >
          <b-form-input id="app-config-form-password-input" v-model="config_dialog_password" required></b-form-input>
        </b-form-group>
      </form>
    </b-modal>

    <!-- 用語集追加 -->
    <b-modal id="add-glossary-dialog" title="用語集の追加" @show="resetGlossaryNameDialogModal" @hidden="resetGlossaryNameDialogModal" @ok="handleGlossaryNameDialogModalOk">
      <form ref="add_glossary_form" @submit.stop.prevent="handleGlossaryAdd" >
        <b-form-group :state="add_glossary_form_state" label="用語集の名前" label-for="add-glossary-form-input" invalid-feedback="名前の入力がないか、/が先頭にあります" >
          <b-form-input id="add-glossary-form-input" v-model="glossary_name_dialog" :state="add_glossary_form_state" required placeholder="/区切り"></b-form-input>
        </b-form-group>
      </form>
    </b-modal>

    <!-- エクスポート インポート用 -->
    <div id="app-file-select">
      <input type="file" ref="app_import_file">
      <a ref="export_link" href="#" download="appdata.js">export</a>
    </div>
  </div>
</template>

<script>
import GlossaryListItem from "./GlossaryListItem.vue";
import GlossaryView from "./GlossaryView.vue";

export default {
  name: "GlossaryTool",
  props: {
    msg: String,
  },
  data: function () {
    return {
      current_glossary: null,
      glossary_list: [],
      config: null,
      config_dialog_url:'',
      config_dialog_user:'',
      config_dialog_password:'',
      glossary_name_dialog: '',
      add_glossary_form_state: null,
      import_file: null,
      loaded: false
    };
  },
  components: {
    'glossary-list-item':GlossaryListItem,
    'glossary-view': GlossaryView
  },
  created: function () {
    // 初期化処理

    // 設定読み込み(設定されていない場合はデフォルトを使用)
    localStorage.glossary_tool_config =
      localStorage.glossary_tool_config ||
      '{"user":"admin", "password":"password", "couch_url":"http://localhost:5984/glossary-tool"}';
    
    this.config = JSON.parse(localStorage.glossary_tool_config);

    // PouchDB接続
    this.initPouch(this.config.couch_url);
  },
  pouch: {
    pouch_glossary: {
      /*empty selector*/
    },
  },
  methods: {
    initPouch: function (couch_url) {
      // PouchDB初期化
      this.$pouch
        .connect(this.config.user, this.config.password, couch_url)
        .then((res) => {
          if (res.status === 0) {
            // オフライン
            console.log('オフライン');
            this.reloadGlossaryList();
            return;
          } else if (res.error === 'unauthorized') {
            // 認証失敗
            console.log('認証失敗');
            this.reloadGlossaryList();
            return;
          }

          // ローカル⇒リモート
          this.$pouch.sync('pouch_glossary', couch_url);

          // リスト作成
          this.reloadGlossaryList();
        })
        .catch((error) => {
          console.error(error);
        });
    },
    disconnectPouch: function (couch_url) {
      // 同期切断
      this.$pouch.disconnect(couch_url);
    },
    recoonectPouch: function (prev_url, next_url) {
      // 設定変更後の再接続
      this.disconnectPouch(prev_url);
      this.initPouch(next_url);
    },
    handleGlossaryAdd: function() {
        // バリデーション
        let valid = this.$refs.add_glossary_form.checkValidity();
        if (!valid) {
          this.add_glossary_form_state = valid;
          return;
        } else if (this.glossary_name_dialog.startsWith('/')) {
          this.add_glossary_form_state = false;
          return;
        }

        // 用語集追加処理
        this.addGlossary(this.glossary_name_dialog);

        // ダイアログを閉じる
        this.$nextTick(function() {
          this.$bvModal.hide('add-glossary-dialog');
        });
    },
    resetGlossaryNameDialogModal: function() {
      this.glossary_name_dialog = '';
      this.add_glossary_form_state = null;
    },
    handleGlossaryNameDialogModalOk: function(bvModalEvt) {
      bvModalEvt.preventDefault();
      this.handleGlossaryAdd();
    },
    addGlossary: function(glossary_name) {
      let self = this;
      this.$pouch.put({"_id":glossary_name, "words":[]}, {}, 'pouch_glossary').then(function() {
        console.log('追加: ' + glossary_name);
        // リストを作り直す
        self.reloadGlossaryList();
      });
    },
    reloadGlossaryList: function() {
      // 全件取得して作り直し
      let self = this;
      this.$pouch.allDocs({
        include_docs: true
      },
      'pouch_glossary'
      ).then(function (result) {
        let glist = [];
        let gmap = {};

        result.rows.forEach(function(row) {
          let idArray = row.id.split('/');
          let path = '';
          idArray.forEach(function(name) {
            let nextPath = path.length == 0 ? name : path + '/' + name;
            let item = null;

            if (row.id == nextPath) {
              // 用語集
              item = {"doc_type":"glossary", "name": name, "doc":row.doc, path:nextPath};
              
            } else if (!gmap[nextPath]) {
              // リストにないフォルダ
              item = {"doc_type":"folder", "name": name, "child":[], path:nextPath};
            } else {
              // 既存フォルダは処理不要
              path = nextPath;
              return;
            }

            if (path) {
              // 親あり
              gmap[path].child.push(item);
              // ソート
              gmap[path].child.sort(function(a, b){
                if (a.doc_type != b.doc_type) {
                  return a.doc_type == 'folder' ? -1 : 1;
                } else if (a.name < b.name) {
                  return -1;
                }
                return 0
              });

            } else {
              // 親なし
              glist.push(item);
              // ソート
              glist.sort(function(a, b){
                if (a.doc_type != b.doc_type) {
                  return a.doc_type == 'folder' ? -1 : 1;
                } else if (a.name < b.name) {
                  return -1;
                }
                return 0
              });
            }

            gmap[nextPath] = item;
            path = nextPath;
          });
        });
        // プロパティを更新
        self.glossary_list = glist;
        self.loaded = true;

        }).catch(function (err) {
        console.log(err);
      });
    },
    removeGlossary:function(doc) {
      let self = this;
      console.log('削除：'+ doc._id);
      if (doc._id == this.current_glossary) {
        // 選択中の用語集を削除する場合はクリアする
        this.current_glossary = null;
        this.$refs.glossary_view.setGlossary(null);
      }
      this.$pouch.remove(doc, {}, 'pouch_glossary').then(function(){
        // リストを再生成
        self.reloadGlossaryList();
      });
    },
    selectGlossary:function(doc) {
      // 用語集を選択
      console.log('選択：'+ doc._id);

      // ビューに設定
      this.$refs.glossary_view.setGlossary(doc);

      // カレントを更新
      this.current_glossary = doc._id;
    },
    handleAppConfigDialogModalShow: function() {
      // 全体設定表示
        this.config_dialog_url = this.config.couch_url;
        this.config_dialog_user = this.config.user;
        this.config_dialog_password = this.config.password;
    },
    handleAppConfigDialogModalOk: function() {
      // 設定変更
      let config_data = {
        user: this.config_dialog_user,
        password: this.config_dialog_password,
        couch_url: this.config_dialog_url
      };

      localStorage.glossary_tool_config = JSON.stringify(config_data);
      this.config = config_data;

      // リロードの注意表示
      this.$bvToast.toast('設定変更は再表示後に有効になります', {
          title: '設定変更',
          variant: 'warning',
          toaster: 'b-toaster-top-left',
          solid: true
        });
    },
    exportAppFile: function() {
      let self = this;
      // 全件取得してエクスポート
      this.$pouch.allDocs(
        {include_docs: true},
        'pouch_glossary'
      ).then(function (result) {
        let data = [];
        result.rows.forEach(function(row){
          data.push({ "_id":row.doc._id, "words":row.doc.words });
        });

        const reader = new FileReader();
        reader.onload = function() {
          // データを設定
          self.$refs.export_link.href='data:text/plain;charset=UTF-8;base64,' + reader.result.split(',')[1];
          // ダウンロードさせる
          self.$refs.export_link.click();
        }

        reader.readAsDataURL(new Blob([JSON.stringify(data)]));
      });
    },
    importAppFile: function() {
      let file = this.$refs.app_import_file.files[0];
      let reader = new FileReader();
      let self = this;

      reader.onload = function() {
        let data = JSON.parse(reader.result);

        // 既存を削除して読み込んだデータを追加する
        self.$pouch.allDocs(
          {include_docs: true}, 
          'pouch_glossary'
        ).then(function(result){
          let deldata = [];
          result.rows.forEach(function(row) {
            let doc = row.doc;
            doc._deleted = true;
            deldata.push(doc);
          });

          if (deldata.length == 0) {
            // 削除データがなかったらそのまま追加
            self.$pouch.bulkDocs(data, {}, 'pouch_glossary').then(function(){
              // 終わったら再生成
              self.reloadGlossaryList();
            });
          } else {
            // 連続削除
            self.$pouch.bulkDocs(deldata, {}, 'pouch_glossary').then(function(){
              // 連続追加
              self.$pouch.bulkDocs(data, {}, 'pouch_glossary').then(function(){
                // 終わったら再生成
                self.reloadGlossaryList();
              });
            });
          }
        });

        // ダイアログは閉じておく
        self.$bvModal.hide('app-config-dialog');
      }

      reader.readAsText(file, 'utf-8');
    },
    clickImportButton: function() {
      let self = this;
      this.$refs.app_import_file.onchange = function() {
        self.importAppFile();
      }
      this.$refs.app_import_file.click();
    },
    updateGlossary: function(doc) {
      // 更新
      let self = this;
      this.$pouch.get(doc._id, {}, 'pouch_glossary').then(function(result){
        self.$pouch.put({"_id":doc._id, "_rev": result._rev, "words": doc.words}, {}, 'pouch_glossary').then(function(){
          console.log('更新: ' + doc._id);
        })
      }).catch(function(error) {
        console.log(error);
      });
    }
  },
};
</script>

<style scoped>
h1 {
  font-weight: bolder;
  color: darkblue;
  text-shadow: 1px 1px cornsilk;
  text-align: center;
}

#list-area {
  background-color: #94afdf;
  color: cornsilk;
}

.app-area {
  width: 100vw;
  height: 100vh;
}

#list-empty {
  text-align: center;
  margin: 30px;
}

#list-empty button {
  margin: 5px;
}

#app-config-button {
  position: absolute;
  top: 10px;
  right: 10px;
}

#add-glossary-button {
  position: absolute;
  top: 70px;
  right: 10px;
}

#glossary-list {
  margin-top: 50px;
}

#app-config-dialog button {
  margin: 30px;
}

#app-config-button-area {
  width: 100%;
  text-align: center;
}

#app-file-select {
  display: none;
}
</style>