<template>
  <default-field
    :field="field"
    :errors="errors"
    :full-width-content="field.width"
  >
    <template slot="field">
        <a
            class="inline-block font-bold cursor-pointer mr-2 animate-text-color select-none border-primary mb-2"
            :class="{ 'text-60': localeKey !== currentLocale, 'text-primary border-b-2': localeKey === currentLocale }"
            :key="`a-${localeKey}`"
            v-for="(locale, localeKey) in field.locales"
            @click="changeTab(localeKey)"
        >
            {{ locale }}
        </a>

      <quill-editor
        :style="css"
        v-model="value[currentLocale]"
        ref="myQuillEditor"
        :options="editorOption"
        @blur="onEditorBlur($event)"
        @focus="onEditorFocus($event)"
        @ready="onEditorReady($event)"
        @keydown.tab="handleTab"
      ></quill-editor>
    </template>
  </default-field>
</template>

<script>
import { FormField, HandlesValidationErrors } from "laravel-nova";
import { quillEditor, Quill } from "vue-quill-editor";
import BlotFormatter from "quill-blot-formatter";
import { ImageExtend, QuillWatch } from "quill-image-extend-module";
import { VideoBlot } from "../../quilljs/VideoBlot";
import Tooltip from "quill/ui/tooltip";
import { CustomImageSpec } from "../../quilljs/CustomImageSpec";
import "quill/dist/quill.core.css";
import "quill/dist/quill.snow.css";
import "quill/dist/quill.bubble.css";
import { EventBus } from '../event-bus';

Quill.register({
  "modules/ImageExtend": ImageExtend,
  "modules/blotFormatter": BlotFormatter,
  "ui/tooltip": Tooltip,
  "formats/video": VideoBlot,
});
const Delta = Quill.import('delta')
export default {
  mixins: [FormField, HandlesValidationErrors],
  components: {
    quillEditor,
  },
  props: ["resourceName", "resourceId", "field"],
  data() {
    return {
      locales: Object.keys(this.field.locales),
      currentLocale: null,
      persisted: [],
      toolbarTips: this.field.tooltip,
      editorOption: {
        placeholder: this.field.placeholder,
        modules: {
          blotFormatter: {
            specs: [CustomImageSpec],
          },
          ImageExtend: {
            loading: true,
            size: this.field.maxFileSize ? this.field.maxFileSize : 2,
            name: "attachment",
            action: `/nova-vendor/quilljs/${this.resourceName}/upload/${this.field.attribute.split(this.field.split)[0]}`,
            response: (res) => {
              return res.url;
            },
            headers: (xhr) => {
              xhr.setRequestHeader(
                "X-CSRF-TOKEN",
                document.head.querySelector('meta[name="csrf-token"]').content
              );
            },
            sizeError: () => {
              this.$toasted.show(
                `Image size exceeds ${
                  this.field.maxFileSize ? this.field.maxFileSize : 2
                }MB`,
                { type: "error" }
              );
            },
            change: (xhr, formData) => {
              const draftId = this._uuid()
              formData.append("draftId", draftId)
              this.persisted.push(draftId)

            },
          },
          toolbar: {
            container: this.field.options,
            handlers: {
              image() {
                QuillWatch.emit(this.quill.id);
              },
              video(value) {
                this.quill.theme.tooltip.edit("video");
              },
            },
          },
        },
      },
    };
  },
  methods: {
    changeTab(locale, dontEmit) {
        if(this.currentLocale !== locale){
            if(!dontEmit){
                EventBus.$emit('localeChanged', locale);
            }

            this.currentLocale = locale;
        }
    },

    handleTab(e) {
        const currentIndex = this.locales.indexOf(this.currentLocale)
        if (!e.shiftKey) {
            if (currentIndex < this.locales.length - 1) {
                e.preventDefault();
                this.changeTab(this.locales[currentIndex + 1]);
            }
        } else {
            if (currentIndex > 0) {
                e.preventDefault();
                this.changeTab(this.locales[currentIndex - 1]);
            }
        }
    },
    _uuid() {
      var d = Date.now();
      if (
        typeof performance !== "undefined" &&
        typeof performance.now === "function"
      ) {
        d += performance.now(); //use high-precision timer if available
      }
      return "xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx".replace(/[xy]/g, function (
        c
      ) {
        var r = (d + Math.random() * 16) % 16 | 0;
        d = Math.floor(d / 16);
        return (c === "x" ? r : (r & 0x3) | 0x8).toString(16);
      });
    },
    /*
     * Set the initial, internal value for the field.
     */
      setInitialValue() {
          this.value = this.field.value || ''
      },

    /**
     * Fill the given FormData object with the field's internal value.
     */
    fill(formData) {
        Object.keys(this.value).forEach(locale => {
            formData.append(this.field.attribute + '[' + locale + ']', this.value[locale] || '')
        })
    },

    /**
     * Update the field's internal value.
     */
    handleChange(value) {
        this.value[this.currentLocale] = value
    },

    onEditorBlur(quill) {
      // console.log("editor blur!", quill);
    },
    onEditorFocus(quill) {
      // console.log("editor focus!", quill);
    },
    onEditorReady(quill) {
      // console.log("editor ready!", quill);
    },
    onEditorChange({ quill, html, text }) {
      this.content = html;
    },
    autotip() {
      if (this.toolbarTips) {
        for (let item of this.toolbarTips) {
          let tip = document.querySelector(".quill-editor " + item.Choice);
          if (!tip) continue;
          tip.setAttribute("title", item.title);
        }
      }
    }
  },
  computed: {
    editor() {
      return this.$refs.myQuillEditor.quill;
    },
    css() {
      return {
        height: this.field.height + 41 + "px",
        "padding-bottom": this.field.paddingBottom + 40 + "px",
      };
    },
  },
  mounted() {
    this.autotip()

     this.currentLocale = this.locales[0] || null;

     EventBus.$on('localeChanged', locale => {
         if(this.currentLocale !== locale) {
             this.changeTab(locale, true);
         }
     });
  },
};
</script>

<style>
.ql-editor p {
  margin-top: 18px;
  font-size: 18px;
}
.ql-video {
  width: 800px;
  height: 450px;
}
</style>
