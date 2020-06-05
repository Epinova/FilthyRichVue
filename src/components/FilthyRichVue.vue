<template>
  <div class="filthy-rich-wrapper">
    <slot class="filthy-rich-over" name="over">
      <div v-html="internalContent"
            ref="editable"
            class="filthy-rich-editor"
            contenteditable />
    <slot class="filthy-rich-under" name="under">
  </div>
</template>

<script>
import * as html from "@leizm/html-parser"

export default {
  data: () => ({
    internalContent: "",
    errors: "",
    nodes: "",
    undos: 0,
    contentHasChanged: false,
    wordCount: 0,
    charCount: 0,
    monitor: null
  }),
  props: {
    content: {
      type: String,
      default: ""
    }
  },
  methods: {
    command(command) {
      if (command === 'h2' || command === 'h3' || command === 'p') {
        document.execCommand('formatBlock', false, command);
      } else {
        document.execCommand(command, false, null)

        if (command === "undo") {
          this.undos = this.undos + 1
        } else if (command === "redo" && this.undos > 0) {
          this.undos = this.undos - 1
        }
      }
    },
    createlink(url) {
      document.execCommand('createlink', false, url)
    },
    parse() {
      const { errors, nodes } = html.parse(this.internalContent)
      this.errors = errors
      this.nodes = nodes
      this.internalContent = html.toString(this.nodes)
    },
    save() {
      this.internalContent = JSON.parse(JSON.stringify(this.$refs['editable'].innerHTML))
      this.parse()
      this.undos = 0
      this.$emit('save', {
        content: this.internalContent
      })
    },
    syncronize() {
      this.internalContent = JSON.parse(JSON.stringify(this.content))
    },
    hasChanges() {
      return this.content !== this.getCurrentContent()
    },
    strippedText(str) {
      if (!document) return ''
      const wrapper = document.createElement("div")
      wrapper.innerHTML = JSON.parse(JSON.stringify(str))
      return wrapper.textContent || wrapper.innerText || ""
    },
    getCurrentContent() {
      if (!this.$refs['editable']) return ''
      return JSON.parse(JSON.stringify(this.$refs['editable'].innerHTML))
    },
    getWords() {
      const words = this.strippedText(this.getCurrentContent())
      return words ? words.match(/\b[-?(\w+)?]+\b/gi).length : 0
    },
    getChars() {
      return this.strippedText(this.getCurrentContent()).length
    }
  },
  watch: {
    content() {
      this.syncronize()
    }
  },
  beforeDestroy() {
    clearInterval(this.watcher)
  },
  created() {
    this.syncronize()
    this.monitor = setInterval(() => {
      this.contentHasChanged = this.hasChanges()
      this.wordCount = this.getWords()
      this.charCount = this.getChars()
    }, 200)
  }
}
</script>
