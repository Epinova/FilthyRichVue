<template>
  <div class="editor-wrapper">
    <div class="toolbar">
      <div
        v-if="features.h2 || features.h3 || features.paragraph"
        class="group"
      >
        <button
          v-if="features.h2"
          @click='format("h2")'
          class="h2"
        >{{ i18n.h2 }}</button>

        <button
          v-if="features.h3"
          @click='format("h3")'
          class="h3"
        >{{ i18n.h3 }}</button>

        <button
          v-if="features.paragraph"
          @click='format("p")'
          class="p"
        >{{ i18n.paragraph }}</button>
      </div>

      <div
        v-if="features.bold || features.italic || features.underline"
        class="group"
      >
        <button
          v-if="features.bold"
          @click='format("bold")'
          :title="i18n.bold"
        ><i class='fa fa-bold'></i></button>

        <button
          v-if="features.italic"
          @click='format("italic")'
          :title="i18n.italic"
        ><i class='fa fa-italic'></i></button>

        <button
          v-if="features.underline"
          @click='format("underline")'
          :title="i18n.underline"
        ><i class='fa fa-underline'></i></button>
      </div>

      <div
        v-if="features.link || features.unlink"
        class="group"
      >
        <button
          v-if="features.link"
          @click='insertLink'
          :title="i18n.insertLink"
        ><i class='fa fa-link'></i></button>

        <button
          v-if="features.unlink"
          @click='format("unlink")'
          :title="i18n.removeLink"
        ><i class='fa fa-unlink'></i></button>
      </div>

      <div
        v-if="features.unorderedList || features.orderedList"
        class="group"
      >
        <button
          v-if="features.unorderedList"
          @click='format("insertUnorderedList")'
          :title="i18n.unorderedList"
        ><i class='fa fa-list-ul'></i></button>

        <button
          v-if="features.orderedList"
          @click='format("insertOrderedList")'
          :title="i18n.norderedList"
        ><i class='fa fa-list-ol'></i></button>
      </div>
    </div>

    <div
      v-html="internalContent"
      ref="editable"
      class="editor"
      contenteditable
    />

    <button
      @click="save"
      class="save-button"
    >
      {{ i18n.save }}
    </button>

    <span>{{ i18n.numWords }}: {{ wordCount }}</span>
    <span v-if="internalContent.length">({{ i18n.numChars }}: {{ charCount }})</span>

    <div
      v-if="features.undo || features.redo"
      class="history-controls"
    >
      <button
        v-if="features.undo && contentHasChanged"
        @click='navigate("undo")'
      ><i class='fa fa-undo'></i></button>

      <button
        v-if="features.redo && this.undos"
        @click='navigate("redo")'
      ><i class='fa fa-repeat'></i></button>
    </div>
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
    features: {
      type: Object,
      default: () => ({
        h2: true,
        h3: true,
        paragraph: true,
        bold: true,
        italic: true,
        underline: true,
        link: true,
        unlink: true,
        unorderedList: true,
        orderedList: true,
        undo: true,
        redo: true
      })
    },
    i18n: {
      type: Object,
      default: () => ({
        h2: "Heading 2",
        h3: "Heading 3",
        paragraph: "Brødtekst",
        bold: "Fet tekst",
        italic: "Italic",
        underline: "Understreket",
        insertLink: "Sett inn link",
        removeLink: "Fjern link",
        unnorderedList: "Uordnet liste",
        norderedList: "Ordnet liste",
        numWords: "Antall ord",
        numChars: "Antall tegn",
        save: "Lagre",
        enterLink: "Enter the link here: ",
      })
    },
    content: {
      type: String,
      default: ""
    }
  },
  methods: {
    format (command) {
      if (command === 'h2' || command === 'h3' || command === 'p') {
        document.execCommand('formatBlock', false, command);
      } else {
        document.execCommand(command, false, null)
      }
    },
    navigate (command) {
      document.execCommand(command, false, null)

      if (command === "undo") {
        this.undos = this.undos + 1
      }
      if (command === "redo" && this.undos > 0) {
        this.undos = this.undos - 1
      }
    },
    insertLink () {
      const url = prompt(this.i18n.enterLink,'https:\/\/')
      document.execCommand('createlink', false, url)
    },
    parse () {
      const { errors, nodes } = html.parse(this.internalContent)
      this.errors = errors
      this.nodes = nodes
      this.internalContent = html.toString(this.nodes)
    },
    save () {
      this.internalContent = JSON.parse(JSON.stringify(this.$refs['editable'].innerHTML))
      this.parse()
      this.undos = 0
      this.$emit('save', {
        content: this.internalContent
      })
    },
    syncronize () {
      this.internalContent = JSON.parse(JSON.stringify(this.content))
    },
    hasChanges () {
      return this.content !== this.getCurrentContent()
    },
    strippedText (str) {
      if (!document) return ''
      const wrapper = document.createElement("div")
      wrapper.innerHTML = JSON.parse(JSON.stringify(str))
      return wrapper.textContent || wrapper.innerText || ""
    },
    getCurrentContent () {
      if (!this.$refs['editable']) return ''
      return JSON.parse(JSON.stringify(this.$refs['editable'].innerHTML))
    },
    getWords () {
      const words = this.strippedText(this.getCurrentContent())
      return words ? words.match(/\b[-?(\w+)?]+\b/gi).length : 0
    },
    getChars () {
      return this.strippedText(this.getCurrentContent()).length
    }
  },
  watch: {
    content () {
      this.syncronize()
    }
  },
  beforeDestroy () {
    clearInterval(this.watcher)
  },
  created () {
    this.syncronize()
    this.monitor = setInterval(() => {
      this.contentHasChanged = this.hasChanges()
      this.wordCount = this.getWords()
      this.charCount = this.getChars()
    }, 200)
  }
}
</script>
