<template>
  <div id="filthy-rich-vue-document">

    <div
      @click="showValidators = !showValidators"
      class="document-header"
    >
      <div v-if="showValidators">Editor + validation</div>
      <div v-else>Editor</div>
    </div>

    <div class="wrap">
      <FilthyRichVue
        :content="content"
        @save="contentChanged"
      />

      <p v-if="markedText">
        Markert tekst ved klikk på save: "{{ markedText }}"
      </p>
    </div>

    <template v-if="showValidators">
      <div class="wrap">
        <h2>Actual HTML</h2>
        <textarea
          v-model="content"
          style="width: 100%; height: 200px"
        />
      </div>

      <div class="wrap">
        <h2>Errors</h2>
        <div style="background: #fcc; padding: 1rem;">{{ errors.length ? errors: 'No errors' }}</div>

        <h2>Converted to JSON</h2>
        <pre style="background: #cfc; max-height: 600px; padding: 1rem; overflow-y: auto;">{{ nodes }}</pre>

        <h2>Converted back to HTML</h2>
        <pre style="background: #ccf; max-height: 600px; padding: 1rem; overflow-y: auto;">{{ contentAsHtml }}</pre>

        <h2>Result: Content {{ content === contentAsHtml ? 'is' : 'is NOT' }} identical</h2>
      </div>
    </template>

  </div>
</template>

<script>
import * as html from "@leizm/html-parser"
import FilthyRichVue from './components/FilthyRichVue.vue'

export default {
  name: 'App',
  components: {
    FilthyRichVue
  },
  data: () => ({
    content: "<p><span>Test</span></p>",
    contentAsHtml: "",
    errors: "",
    nodes: "",
    markedText: "",
    showValidators: false
  }),
  methods: {
    reportSelected () {
      let text = "";
      if (typeof window.getSelection != "undefined") {
        text = window.getSelection().toString()
      } else if (typeof document.selection != "undefined" && document.selection.type == "Text") {
        text = document.selection.createRange().text
      }
      this.markedText = text
    },
    format (command) {
      if (command === 'h2' || command === 'h3' || command === 'p') {
        document.execCommand('formatBlock', false, command);
      } else {
        document.execCommand(command, false, null)
      }
    },
    insertLink () {
      const url = prompt('Enter the link here: ','https:\/\/')
      document.execCommand('createlink', false, url)
    },
    washArray (arr) {
      const result = []
      const copy = JSON.parse(JSON.stringify(arr))
      copy.forEach((item) => {
        result.push(this.washObject(item))
      })
      return result
    },
    washObject (obj) {
      const result = {}
      const copy = JSON.parse(JSON.stringify(obj))

      Object.keys(copy).forEach((key) => {
        if (key !== 'properties') {
          let value = JSON.parse(JSON.stringify(copy[key]))
          if (key === 'children') {
            value = this.washArray(value)
          }
          result[key] = value
        }
      })
      
      return result
    },
    parse () {
      const { errors, nodes } = html.parse(this.content)
      this.errors = errors
      this.nodes = nodes
      this.contentAsHtml = html.toString(this.nodes)
    },
    contentChanged ({ content }) {
      this.reportSelected()
      this.content = content
    }
  },
  watch: {
    content () {
      this.parse()
    }
  }
}
</script>

<style>
html, body, div {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

#filthy-rich-vue-document {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  font-size: 1.4rem;
}

.document-header {
  background: #000;
  color: #fff;
  margin-bottom: 60px;
  cursor: pointer;
  padding: 20px;
}

button {
  padding: 10px;
  background: #666;
  color: #fff;
  border: 0;
  cursor: pointer;
  border-radius: 3px;
}

button.save-button {
  font-weight: bold;
  font-size: 1.4rem;
  padding: 8px 20px;
  margin-top: 5px;
}

h2 {
  margin: 40px 0 20px 0;
}

textarea {
  font-size: 1.2rem;
  padding: 10px;
}

.wrap {
  margin: 100px;
}

.editor-wrapper {
  position: relative;
  background: #efe;
  padding: 10px;
}

.toolbar {
  text-align: right;
  position: absolute;
  top: 15px;
  right: 17px;
}

.toolbar .group {
  display: inline;
  margin-left: 1rem;
}

.editor {
  min-height: 150px;
  overflow: auto;
  padding: 2.5rem 1rem 1rem;
  resize: vertical;
  outline: none;
  border: 3px solid #9f9;
}

.history-controls {
  float: right;
  margin-top: 3px;
}
</style>
