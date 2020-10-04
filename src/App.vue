<template>
  <div id="app">
    <GlossaryTool />
  </div>
</template>

<script>
import GlossaryTool from "./components/GlossaryTool.vue";
import Vue from "vue";

// bootstrap用
import { BootstrapVue, IconsPlugin } from "bootstrap-vue";

import "bootstrap/dist/css/bootstrap.css";
import "bootstrap-vue/dist/bootstrap-vue.css";

Vue.use(BootstrapVue);
Vue.use(IconsPlugin);

// PouchDB用
import PouchDB from "pouchdb-browser";
import PouchFind from "pouchdb-find";
import PouchLiveFind from "pouchdb-live-find";
import PouchAuthentication from "pouchdb-authentication";
import PouchVue from "pouch-vue";

PouchDB.plugin(PouchFind);
PouchDB.plugin(PouchLiveFind);
PouchDB.plugin(PouchAuthentication);

Vue.use(PouchVue, {
  pouch: PouchDB,
  defaultDB: "http://localhost:5984/glossary-tool",
  optionsDB: {
    fetch: function (url, opts) {
      opts.credentials = "include";
      return PouchDB.fetch(url, opts);
    },
  },
});

export default {
  name: "App",
  components: {
    GlossaryTool,
  },
};
</script>

<style>
#app {
  width: 100vw;
  height: 100vh;
}
</style>
