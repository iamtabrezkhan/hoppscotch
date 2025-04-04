<template>
  <SmartModal v-if="show" @close="hideModal">
    <div slot="header">
      <div class="row-wrapper">
        <h3 class="title">{{ $t("import_export") }} {{ $t("environments") }}</h3>
        <div>
          <v-popover>
            <button class="tooltip-target icon" v-tooltip.left="$t('more')">
              <i class="material-icons">more_vert</i>
            </button>
            <template slot="popover">
              <div>
                <button class="icon" @click="readEnvironmentGist" v-close-popover>
                  <i class="material-icons">assignment_returned</i>
                  <span>{{ $t("import_from_gist") }}</span>
                </button>
              </div>
              <div
                v-tooltip.bottom="{
                  content: !fb.currentUser
                    ? $t('login_with_github_to') + $t('create_secret_gist')
                    : fb.currentUser.provider !== 'github.com'
                    ? $t('login_with_github_to') + $t('create_secret_gist')
                    : null,
                }"
              >
                <button
                  :disabled="
                    !fb.currentUser ? true : fb.currentUser.provider !== 'github.com' ? true : false
                  "
                  class="icon"
                  @click="createEnvironmentGist"
                  v-close-popover
                >
                  <i class="material-icons">assignment_turned_in</i>
                  <span>{{ $t("create_secret_gist") }}</span>
                </button>
              </div>
            </template>
          </v-popover>
          <button class="icon" @click="hideModal">
            <i class="material-icons">close</i>
          </button>
        </div>
      </div>
    </div>
    <div slot="body" class="flex flex-col">
      <div class="flex flex-col items-start p-2">
        <span
          v-tooltip="{
            content: !fb.currentUser ? $t('login_first') : $t('replace_current'),
          }"
        >
          <button :disabled="!fb.currentUser" class="icon" @click="syncEnvironments">
            <i class="material-icons">folder_shared</i>
            <span>{{ $t("import_from_sync") }}</span>
          </button>
        </span>
        <button
          class="icon"
          @click="openDialogChooseFileToReplaceWith"
          v-tooltip="$t('replace_current')"
        >
          <i class="material-icons">create_new_folder</i>
          <span>{{ $t("replace_json") }}</span>
          <input
            type="file"
            @change="replaceWithJSON"
            style="display: none"
            ref="inputChooseFileToReplaceWith"
            accept="application/json"
          />
        </button>
        <button
          class="icon"
          @click="openDialogChooseFileToImportFrom"
          v-tooltip="$t('preserve_current')"
        >
          <i class="material-icons">folder_special</i>
          <span>{{ $t("import_json") }}</span>
          <input
            type="file"
            @change="importFromJSON"
            style="display: none"
            ref="inputChooseFileToImportFrom"
            accept="application/json"
          />
        </button>
      </div>
      <div v-if="showJsonCode" class="row-wrapper">
        <textarea v-model="environmentJson" rows="8" readonly></textarea>
      </div>
    </div>
    <div slot="footer">
      <div class="row-wrapper">
        <span>
          <SmartToggle :on="showJsonCode" @change="showJsonCode = $event">
            {{ $t("show_code") }}
          </SmartToggle>
        </span>
        <span>
          <button class="icon" @click="hideModal">
            {{ $t("cancel") }}
          </button>
          <button class="icon primary" @click="exportJSON" v-tooltip="$t('download_file')">
            {{ $t("export") }}
          </button>
        </span>
      </div>
    </div>
  </SmartModal>
</template>

<script>
import { fb } from "~/helpers/fb"
import { getSettingSubject } from "~/newstore/settings"

export default {
  data() {
    return {
      fb,
      showJsonCode: false,
    }
  },
  subscriptions() {
    return {
      SYNC_ENVIRONMENTS: getSettingSubject("syncEnvironments")
    }
  },
  props: {
    show: Boolean,
  },
  computed: {
    environmentJson() {
      return JSON.stringify(this.$store.state.postwoman.environments, null, 2)
    },
  },
  methods: {
    async createEnvironmentGist() {
      await this.$axios
        .$post(
          "https://api.github.com/gists",
          {
            files: {
              "hoppscotch-environments.json": {
                content: this.environmentJson,
              },
            },
          },
          {
            headers: {
              Authorization: `token ${fb.currentUser.accessToken}`,
              Accept: "application/vnd.github.v3+json",
            },
          }
        )
        .then(({ html_url }) => {
          this.$toast.success(this.$t("gist_created"), {
            icon: "done",
          })
          window.open(html_url)
        })
        .catch((error) => {
          this.$toast.error(this.$t("something_went_wrong"), {
            icon: "error",
          })
          console.log(error)
        })
    },
    async readEnvironmentGist() {
      let gist = prompt(this.$t("enter_gist_url"))
      if (!gist) return
      await this.$axios
        .$get(`https://api.github.com/gists/${gist.split("/").pop()}`, {
          headers: {
            Accept: "application/vnd.github.v3+json",
          },
        })
        .then(({ files }) => {
          let environments = JSON.parse(Object.values(files)[0].content)
          this.$store.commit("postwoman/replaceEnvironments", environments)
          this.fileImported()
          this.syncToFBEnvironments()
        })
        .catch((error) => {
          this.failedImport()
          console.log(error)
        })
    },
    hideModal() {
      this.$emit("hide-modal")
    },
    openDialogChooseFileToReplaceWith() {
      this.$refs.inputChooseFileToReplaceWith.click()
    },
    openDialogChooseFileToImportFrom() {
      this.$refs.inputChooseFileToImportFrom.click()
    },
    replaceWithJSON() {
      let reader = new FileReader()
      reader.onload = ({ target }) => {
        let content = target.result
        let environments = JSON.parse(content)
        this.$store.commit("postwoman/replaceEnvironments", environments)
      }
      reader.readAsText(this.$refs.inputChooseFileToReplaceWith.files[0])
      this.fileImported()
      this.syncToFBEnvironments()
      this.$refs.inputChooseFileToReplaceWith.value = ""
    },
    importFromJSON() {
      let reader = new FileReader()
      reader.onload = ({ target }) => {
        let content = target.result
        let importFileObj = JSON.parse(content)
        if (
          importFileObj["_postman_variable_scope"] === "environment" ||
          importFileObj["_postman_variable_scope"] === "globals"
        ) {
          this.importFromPostman(importFileObj)
        } else {
          this.importFromPostwoman(importFileObj)
        }
      }
      reader.readAsText(this.$refs.inputChooseFileToImportFrom.files[0])
      this.syncToFBEnvironments()
      this.$refs.inputChooseFileToImportFrom.value = ""
    },
    importFromPostwoman(environments) {
      let confirmation = this.$t("file_imported")
      this.$store.commit("postwoman/importAddEnvironments", {
        environments,
        confirmation,
      })
    },
    importFromPostman({ name, values }) {
      let environment = { name, variables: [] }
      values.forEach(({ key, value }) => environment.variables.push({ key, value }))
      let environments = [environment]
      this.importFromPostwoman(environments)
    },
    exportJSON() {
      let text = this.environmentJson
      text = text.replace(/\n/g, "\r\n")
      let blob = new Blob([text], {
        type: "text/json",
      })
      let anchor = document.createElement("a")
      anchor.download = "hoppscotch-environment.json"
      anchor.href = window.URL.createObjectURL(blob)
      anchor.target = "_blank"
      anchor.style.display = "none"
      document.body.appendChild(anchor)
      anchor.click()
      document.body.removeChild(anchor)
      this.$toast.success(this.$t("download_started"), {
        icon: "done",
      })
    },
    syncEnvironments() {
      this.$store.commit("postwoman/replaceEnvironments", fb.currentEnvironments)
      this.fileImported()
    },
    syncToFBEnvironments() {
      if (fb.currentUser !== null && this.SYNC_ENVIRONMENTS) {
        fb.writeEnvironments(JSON.parse(JSON.stringify(this.$store.state.postwoman.environments)))
      }
    },
    fileImported() {
      this.$toast.info(this.$t("file_imported"), {
        icon: "folder_shared",
      })
    },
  },
}
</script>
