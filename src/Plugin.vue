<template>
  <div>
    <div class="uk-form-row">
      <label>Language</label>
      <select v-model="model.language" class="uk-width-1-1">
        <option
          v-for="item in languages"
          :key="item.key"
          :value="item.key"
        >
          {{ item.title }}
        </option>
      </select>
    </div>

    <div class="uk-form-row">
      <label>Code Snippet</label>
      <prism-editor
        :key="editorKey"
        v-model="model.content"
        :language="model.language"
        :lineNumbers="model.lineNumbers"
        class="uk-width-1-1"
      />
    </div>

    <div class="uk-form-row">
      <label>Options</label>
      <div class="flex row flex--align-items-center flex--justify-content-space-between">
        <label>Toggle Line Numbers</label>
        <input
          type="checkbox"
          v-model="model.lineNumbers"
        />
      </div>
    </div>
  </div>
</template>

<script>
import PrismEditor from "vue-prism-editor"
import PrismComponents from 'prismjs/components'
import Prism from "prismjs";

export default {
  components: {
    PrismEditor 
  },
  mixins: [window.Storyblok.plugin],
  data: () => ({
    editorKey: 0,
    builtInLanguages: [
      {
        title: 'JavaScript',
        key: 'js'
      }, {
        title: 'HTML',
        key: 'html'
      }, {
        title: 'Markdown',
        key: 'md'
      }, {
        title: 'TypeScript',
        key: 'ts'
      }, {
        title: 'Vue.js',
        key: 'vue'
      }
    ]
  }),
  computed: {
    languages() {
      // gather prism languages from components
      const prismLanguages = []
      for (const key in PrismComponents.languages) {
        const language = { ...PrismComponents.languages[key] }
        if (language.title) {
          // add the key as attribute of the language object
          language.key = key
          prismLanguages.push(language)
        }
      }

      // merge with builtin languages by title
      const merge = (a, b, p) => a.filter( aa => ! b.find ( bb => aa[p] === bb[p]) ).concat(b);
      const languages = merge(prismLanguages, this.builtInLanguages, 'title')

      // sort alphabetically
      return languages.sort((a, b) => {
        return a.title.localeCompare(b.title)
      })
    }
  },
  watch: {
    'model': {
      handler(value) {
        this.$emit('changed-model', value);
      },
      deep: true
    },
    'model.language': {
      handler(value) {
        this.fetchLanguageIfNecessary(value)
      }
    }
  },
  mounted() {
    this.fetchLanguageIfNecessary(this.model.language)
  },
  methods: {
    initWith() {
      return {
        plugin: 'code',
        language: 'js',
        content: '// your code goes here',
        lineNumbers: true
      }
    },
    isBuiltIn(language) {
      return this.builtInLanguages.filter(l => l.key === language.key).length > 0
    },
    fetchLanguage(key) {
      return new Promise((resolve) => {
        this.$sb.getScript(`https://unpkg.com/prismjs/components/prism-${key}.min.js`, () => {
          resolve(true)
        })
      })
    },
    loadDependencies(key) {
      return new Promise((resolve, reject) => {
        // lets grap all language keys of dependent languages
        const dependencies = this.getLanguageDependenciesByKey(key)
        if (dependencies.length === 0) resolve(true)

        for (let i = 0; i < dependencies.length; i++) {
          const dependency = dependencies[i]
          // load the language
          this.loadLanguage(dependency)
            .then(r => {
              // resolve after last dependency
              if (i === dependencies.length - 1) {
                resolve(r)
              }
            })
            .catch(e => reject(e))
        }
      })
    },
    loadLanguage(key) {
      // we gonna load a single language
      // this language can be dependent on other languages
      // therefore we gonna look for all dependencies first
      return new Promise((resolve, reject) => {
        this.loadDependencies(key)
          .then(() => {
            if (Prism.languages[key]) {
              resolve(true)
            } else {
              // ... and then we fetch the requested language
              this.fetchLanguage(key)
                .then(r => resolve(r))
                .catch(e => reject(e))
            }
          })
          .catch(e => reject(e))
      })
    },
    fetchLanguageIfNecessary(key) {
      return new Promise((resolve) => {
        // grab the language
        const language = this.getLanguageByKey(key)

        // is it already built in and available?
        if (!language || this.isBuiltIn(language)) {
          resolve(language)
        }

        // if the language is not built in by vue prism editor
        // we gonna grab the component from official prism source through unpkg
        this.loadLanguage(language.key)
          .then(() => {
            // to make the language syntax highlighting visible
            // force rerendering of the editor
            this.rerenderEditor()
            resolve(true)
          })
      })
    },
    rerenderEditor() {
      // changing the key forces the component to be removed and readded
      this.editorKey += 1
    },
    getLanguageByKey(key) {
      const matches = this.languages.filter(l => l.key === key)
      return !matches || matches.length === 0 ? null : matches[0]
    },
    getLanguageMetaByKey(key) {
      if (this.prismLanguageExists(key)) {
        return PrismComponents.languages[key]
      }
      return null
    },
    getLanguageDependenciesByKey(key) {
      let meta = this.getLanguageMetaByKey(key) && this.getLanguageMetaByKey(key).require ? this.getLanguageMetaByKey(key).require : []
      if (!Array.isArray(meta)) {
        meta = [meta]
      }
      return meta
    },
    prismLanguageExists(key) {
      return Object.keys(PrismComponents.languages).some((k) => k === key)
    }
  }
}
</script>

<style scoped>
.flex {
  display: flex;
}
.flex.row {
  flex-direction: row;
}
.flex.flex--align-items-center {
  align-items: center;
}
.flex.flex--justify-content-space-between {
  justify-content: space-between;
}
</style>
