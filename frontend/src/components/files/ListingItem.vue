<template>
  <div
    class="item"
    role="button"
    tabindex="0"
    :draggable="isDraggable"
    @dragstart="dragStart"
    @dragover="dragOver"
    @drop="drop"
    @click="itemClick"
    :data-thumbs-enabled="isThumbsEnabled"
    :data-dir="isDir"
    :data-type="type"
    :aria-label="name"
    :aria-selected="isSelected"
  >
    <div>
      <img
        v-if="hasThumbnailUrl"
        v-lazy="thumbnailUrl"
      />
      <i v-else class="material-icons"></i>
    </div>

    <div>
      <p class="name">{{ name }}</p>

      <p v-if="isDir" class="size" data-order="-1">&mdash;</p>
      <p v-else class="size" :data-order="humanSize()">{{ humanSize() }}</p>

      <p class="modified">
        <time :datetime="modified">{{ humanTime() }}</time>
      </p>
    </div>
  </div>
</template>

<script>
import { baseURL, enableThumbs } from "@/utils/constants";
import { mapMutations, mapGetters, mapState } from "vuex";
import filesize from "filesize";
import moment from "moment";
import { files as api } from "@/api";
import * as upload from "@/utils/upload";

export default {
  name: "item",
  data: function () {
    return {
      touches: 0,
    };
  },
  props: [
    "name",
    "isDir",
    "isThumbsEnabled",
    "url",
    "type",
    "size",
    "modified",
    "index",
    "readOnly",
  ],
  computed: {
    ...mapState(["user", "selected", "req", "jwt"]),
    ...mapGetters(["selectedCount"]),
    singleClick() {
      return this.readOnly == undefined && this.user.singleClick;
    },
    isSelected() {
      return this.selected.indexOf(this.index) !== -1;
    },
    isDraggable() {
      return this.readOnly == undefined && this.user.perm.rename;
    },
    canDrop() {
      if (!this.isDir || this.readOnly !== undefined) return false;

      for (let i of this.selected) {
        if (this.req.items[i].url === this.url) {
          return false;
        }
      }

      return true;
    },
    thumbnailUrl() {
      const path = this.url.replace(/^\/files\//, "");

      // reload the image when the file is replaced
      const key = Date.parse(this.modified);

      return `${baseURL}/api/preview/thumb/${path}?k=${key}&inline=true`;
    },
    hasThumbnailUrl() {
      return this.readOnly == undefined && this.isThumbsEnabled && enableThumbs;
    },
  },
  methods: {
    ...mapMutations(["addSelected", "removeSelected", "resetSelected"]),
    humanSize: function () {
      return filesize(this.size);
    },
    humanTime: function () {
      if (this.readOnly == undefined && this.user.dateFormat) {
        return moment(this.modified).format("L LT");
      }
      return moment(this.modified).fromNow();
    },
    dragStart: function () {
      if (this.selectedCount === 0) {
        this.addSelected(this.index);
        return;
      }

      if (!this.isSelected) {
        this.resetSelected();
        this.addSelected(this.index);
      }
    },
    dragOver: function (event) {
      if (!this.canDrop) return;

      event.preventDefault();
      let el = event.target;

      for (let i = 0; i < 5; i++) {
        if (!el.classList.contains("item")) {
          el = el.parentElement;
        }
      }

      el.style.opacity = 1;
    },
    drop: async function (event) {
      if (!this.canDrop) return;
      event.preventDefault();

      if (this.selectedCount === 0) return;

      let el = event.target;
      for (let i = 0; i < 5; i++) {
        if (el !== null && !el.classList.contains("item")) {
          el = el.parentElement;
        }
      }

      let items = [];

      for (let i of this.selected) {
        items.push({
          from: this.req.items[i].url,
          to: this.url + encodeURIComponent(this.req.items[i].name),
          name: this.req.items[i].name,
        });
      }

      // Get url from ListingItem instance
      let path = el.__vue__.url;
      let baseItems = (await api.fetch(path)).items;

      let action = (overwrite, rename) => {
        api
          .move(items, overwrite, rename)
          .then(() => {
            this.$store.commit("setReload", true);
          })
          .catch(this.$showError);
      };

      let conflict = upload.checkConflict(items, baseItems);

      let overwrite = false;
      let rename = false;

      if (conflict) {
        this.$store.commit("showHover", {
          prompt: "replace-rename",
          confirm: (event, option) => {
            overwrite = option == "overwrite";
            rename = option == "rename";

            event.preventDefault();
            this.$store.commit("closeHovers");
            action(overwrite, rename);
          },
        });

        return;
      }

      action(overwrite, rename);
    },
    itemClick: function (event) {
      if (this.singleClick && !this.$store.state.multiple) this.open();
      else this.click(event);
    },
    click: function (event) {
      if (!this.singleClick && this.selectedCount !== 0) event.preventDefault();

      setTimeout(() => {
        this.touches = 0;
      }, 300);

      this.touches++;
      if (this.touches > 1) {
        this.open();
      }

      if (this.$store.state.selected.indexOf(this.index) !== -1) {
        this.removeSelected(this.index);
        return;
      }

      if (event.shiftKey && this.selected.length > 0) {
        let fi = 0;
        let la = 0;

        if (this.index > this.selected[0]) {
          fi = this.selected[0] + 1;
          la = this.index;
        } else {
          fi = this.index;
          la = this.selected[0] - 1;
        }

        for (; fi <= la; fi++) {
          if (this.$store.state.selected.indexOf(fi) == -1) {
            this.addSelected(fi);
          }
        }

        return;
      }

      if (
        !this.singleClick &&
        !event.ctrlKey &&
        !event.metaKey &&
        !this.$store.state.multiple
      )
        this.resetSelected();
      this.addSelected(this.index);
    },
    open: function () {
      this.$router.push({ path: this.url });
    },
  },
};
</script>
