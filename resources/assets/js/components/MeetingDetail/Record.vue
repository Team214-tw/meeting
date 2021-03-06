<template>
  <div>
    <button
      class="uk-button uk-button-primary uk-button-small uk-align-right edit-button"
      v-if="canModify && !editingRecord"
      @click="startEdit">
      編輯
    </button>

    <div
      class="markdown-body uk-overflow-hidden"
      v-show="!editingRecord"
      v-html="rederedHtml">
    </div>

    <mavonEditor
      language="en"
      v-if="editingRecord"
      :externalLink="false"
      defaultOpen="edit"
      :autofocus="false"
      :ishljs="false"
      :toolbars="toolbars"
      :subfield="false"
      :boxShadow="false"
      v-model="meetingRecord"
      @save="saveRecord">
    </mavonEditor>

    <button
      class="uk-button uk-button-primary uk-align-right edit-button uk-margin-top"
      v-if="editingRecord"
      @click="endEdit">
      完成
    </button>

    <div
      ref="backup-modal"
      class="uk-modal-container"
      uk-modal="bg-close: false; esc-close: false;">
      <div class="uk-modal-dialog">
        <div class="uk-modal-header">
          <h4>
            以下為上次未完成的紀錄<br>
            是否回復紀錄？
          </h4>
        </div>
        <div class="uk-modal-body" uk-overflow-auto>
          <span class="pre-wrap">{{ backupRecord }}</span>
        </div>
        <div class="uk-modal-footer uk-text-right">
          <button class="uk-button uk-button-default uk-modal-close">
            放棄
          </button>
          <button class="uk-button uk-button-primary uk-modal-close" @click="restoreRecord">
            回復
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
  .v-note-wrapper {
    z-index: 10;
  }
</style>

<script>
import { mavonEditor } from 'mavon-editor';
import { mapState } from 'vuex';
import 'mavon-editor/dist/css/index.css';

export default {
  beforeRouteLeave(to, from, next) {
    if (this.editingRecord) {
      this.saveRecord(() => {
        this.editingRecord = false;
        next();
      });
    } else {
      next();
    }
  },
  props: ['meeting'],
  components: {
    mavonEditor,
  },
  computed: {
    canModify() {
      return (
        this.meeting.status <= this.$meetingStatus.End
        && this.meeting.owner_id === this.user.user_id);
    },
    meetingRecord: {
      set(record) {
        this.$set(this.meeting, 'record', record);
      },
      get() {
        return this.meeting.record ? this.meeting.record : '';
      },
    },
    ...mapState(['user', 'loading']),
  },
  watch: {
    meetingRecord() {
      localStorage.setItem(`${this.meeting.id}_record`, this.meetingRecord);
    },
  },
  data() {
    return {
      rederedHtml: '',
      backupRecord: null,
      editingRecord: false,
      toolbars: {
        bold: true,
        italic: true,
        header: true,
        underline: true,
        strikethrough: true,
        mark: true,
        superscript: true,
        subscript: true,
        quote: true,
        ol: true,
        ul: true,
        link: true,
        imagelink: true,
        code: true,
        table: true,
        fullscreen: true,
        htmlcode: true,
        undo: true,
        redo: true,
        trash: true,
        save: true,
        alignleft: true,
        aligncenter: true,
        alignright: true,
        subfield: true,
        preview: true,
      },
    };
  },
  methods: {
    startEdit() {
      this.editingRecord = true;
      this.backupRecord = localStorage.getItem(`${this.meeting.id}_record`);
      if (this.backupRecord !== null) {
        UIkit.modal(this.$refs['backup-modal']).show();
      }
    },
    endEdit() {
      this.saveRecord(() => {
        this.editingRecord = false;
      });
    },
    restoreRecord() {
      this.$set(this.meeting, 'record', this.backupRecord);
      UIkit.modal(this.$refs['backup-modal']).hide();
    },
    saveRecord(resolve) {
      this.$store.commit('startLoad');
      axios.put(`/api/meetings/${this.meeting.id}`, {
        record: this.meetingRecord,
      }).then(() => {
        localStorage.removeItem(`${this.meeting.id}_record`);
        this.$store.commit('endLoad');
        resolve();
      });
      this.rederedHtml = mavonEditor.getMarkdownIt().render(this.meetingRecord);
    },
  },
  mounted() {
    const self = this;
    this.rederedHtml = mavonEditor.getMarkdownIt().set({
      html: false,
      linkify: true,
      tocCallback(tocMarkdown, tocArray, tocHtml) {
        self.$emit('update:toc', tocHtml);
      },
    }).render(this.meetingRecord);
  },
};
</script>
