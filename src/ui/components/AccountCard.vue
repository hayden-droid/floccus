<template>
  <v-card
    :loading="Boolean(account.data.syncing)"
    color="light-blue-lighten-5">
    <template #progress>
      <v-progress-linear
        v-if="account.data.syncing"
        :value="account.data.syncing * 100 || 0" />
    </template>
    <v-container class="pa-4">
      <v-row
        no-gutters
        class="flex-column">
        <v-col>
          <v-row no-gutters>
            <v-col class="flex-grow-1">
              <div class="overline">
                {{ account.data.type }}
              </div>
              <div class="text-h6">
                <v-icon
                  v-if="account.data.localRoot === 'tabs'"
                  color="primary">
                  mdi-tab
                </v-icon>
                <v-icon
                  v-else
                  color="primary">
                  mdi-folder
                </v-icon> {{ folderName }}
              </div>
              <div class="caption">
                {{ uri }}
              </div>
            </v-col>
            <v-col
              class="align-end flex-grow-0"
              :style="{minWidth: 'max-content'}">
              <div class="pa-3 d-inline-block font-weight-light body-2">
                <v-icon
                  :color="statusColor"
                  :class="{spinning: account.data.syncing}">
                  {{ statusIcon }}
                </v-icon>
                <span :style="{color: statusColor}">{{ statusLabel }}</span>
              </div>
            </v-col>
          </v-row>
        </v-col>
        <v-col class="mt-3">
          <v-alert
            dense
            dark
            outlined
            :icon="false"
            :type="statusType"
            class="pa-2 text-caption">
            {{ statusDetail }} <v-btn
              v-if="account.data.error"
              :color="statusType"
              class="float-right"
              x-small
              @click="onGetLogs">
              {{ t('LabelDebuglogs') }}
            </v-btn>
          </v-alert>
          <v-alert
            v-if="legacyWarning"
            dense
            outlined
            :type="'warning'">
            {{ legacyWarning }}
          </v-alert>
          <v-alert
            v-if="!account.data.failsafe"
            dense
            outlined
            :type="'warning'">
            {{ t('StatusFailsafeoff') }}
          </v-alert>
        </v-col>
        <v-col>
          <v-row
            no-gutters
            class="mt-2">
            <v-col class="d-flex flex-row">
              <v-select
                v-model="account.data.strategy"
                dense
                :items="[
                  {text: strategyLabels['slave'], value: 'slave', icon: strategyIcons['slave']},
                  {text: strategyLabels['overwrite'], value: 'overwrite', icon: strategyIcons['overwrite']},
                  {text: strategyLabels['default'], value: 'default', icon: strategyIcons['default']},
                ]"
                @change="onChangeStrategy">
                <template #prepend-inner>
                  <v-icon>{{ strategyIcons[account.data.strategy] }}</v-icon>
                </template>
                <template #item="{item}">
                  <v-icon>{{ item.icon }}</v-icon> {{ item.text }}
                </template>
              </v-select>
              <v-btn
                v-if="!account.data.syncing"
                class="primary"
                small
                @click="onTriggerSync">
                <v-icon>mdi-sync</v-icon>
                {{ t('LabelSyncnow') }}
              </v-btn>
              <v-btn
                v-else
                small
                @click="onCancelSync">
                <v-icon>mdi-cancel</v-icon>
                {{ t('LabelCancelsync') }}
              </v-btn>
            </v-col>
          </v-row>
          <v-row :class="{'d-none': !showDetails, 'pa-2': true, 'mt-3': true, 'justify-space-between': true}">
            <v-btn
              small
              @click="onTriggerSyncUp">
              <v-icon>mdi-arrow-up-bold</v-icon>
              {{ t('LabelSyncUpOnce') }}
            </v-btn>
            <v-switch
              v-model="account.data.enabled"
              :aria-label="t('LabelAutosync')"
              :label="t('LabelAutosync')"
              dense
              class="mt-0 pt-0"
              @change="onToggleEnabled" />
          </v-row>
          <v-row :class="{'d-none': !showDetails, 'pa-2': true, 'mb-3': true, 'justify-space-between': true}">
            <v-btn
              small
              @click="onTriggerSyncDown">
              <v-icon>mdi-arrow-down-bold</v-icon>
              {{ t('LabelSyncDownOnce') }}
            </v-btn>

            <v-btn
              small
              :to="{ name: routes.ACCOUNT_OPTIONS, params: { accountId: account.id } }"
              target="_blank">
              <v-icon>mdi-settings</v-icon>
              {{ t('LabelOptions') }}
            </v-btn>
          </v-row>
          <v-row>
            <v-btn
              text
              block
              @click="showDetails = !showDetails">
              <v-icon v-if="!showDetails">
                mdi-chevron-down
              </v-icon>
              <v-icon v-else>
                mdi-chevron-up
              </v-icon>
            </v-btn>
          </v-row>
        </v-col>
      </v-row>
    </v-container>
  </v-card>
</template>

<script>
import PathHelper from '../../lib/PathHelper'
import humanizeDuration from 'humanize-duration'
import { actions } from '../store'
import { routes } from '../router'
import BrowserTree from '../../lib/browser/BrowserTree'

export default {
  name: 'AccountCard',
  props: {
    account: {
      type: Object,
      required: true
    }
  },
  data() {
    return {
      rootPath: '',
      statusColors: {
        disabled: 'rgb(125, 114, 128)',
        ok: '#3d8e39',
        error: '#8e3939',
        syncing: '#2196F3'
      },
      statusIcons: {
        disabled: 'mdi-sync-off',
        ok: 'mdi-check',
        error: 'mdi-sync-alert',
        syncing: 'mdi-sync'
      },
      statusLabels: {
        disabled: this.t('StatusDisabled'),
        ok: this.t('StatusAllgood'),
        error: this.t('StatusError'),
        syncing: this.t('StatusSyncing')
      },
      strategyIcons: {
        slave: 'mdi-arrow-down-bold',
        overwrite: 'mdi-arrow-up-bold',
        default: 'mdi-sync',
      },
      strategyLabels: {
        slave: this.t('LabelSyncDown'),
        overwrite: this.t('LabelSyncUp'),
        default: this.t('LabelSyncNormal'),
      },
      showDetails: false
    }
  },
  computed: {
    folderName() {
      const pathArray = PathHelper.pathToArray(
        this.rootPath || this.t('LabelRootfolder')
      )
      return pathArray[pathArray.length - 1] || this.t('LabelUntitledfolder')
    },
    localRoot() {
      return this.account.data.localRoot
    },
    uri() {
      return this.account.label
    },
    status() {
      if (this.account.data.syncing) {
        return 'syncing'
      }
      if (this.account.data.error) {
        return 'error'
      }
      if (!this.account.data.enabled) {
        return 'disabled'
      }
      return 'ok'
    },
    statusIcon() {
      return this.statusIcons[this.status]
    },
    statusColor() {
      return this.statusColors[this.status]
    },
    statusLabel() {
      return this.statusLabels[this.status]
    },
    statusType() {
      if (this.account.data.error) {
        return 'error'
      }
      return 'info'
    },
    statusDetail() {
      if (this.account.data.error) {
        return this.account.data.error
      }
      if (this.account.data.syncing) {
        return 'Synchronization in progress.'
      }
      if (this.account.data.lastSync) {
        return this.t(
          'StatusLastsynced',
          [humanizeDuration(Date.now() - this.account.data.lastSync, {
            largest: 1,
            round: true,
            language: navigator.language.split('-')[0],
            fallbacks: navigator.languages.map(lang => lang.split('-')[0]).concat(['en'])
          })]
        )
      }
      return this.t('StatusNeversynced')
    },
    legacyWarning() {
      if (this.account.data.type === 'nextcloud' ||
          this.account.data.type === 'nextcloud-legacy') {
        return this.t('LegacyAdapterDeprecation')
      }
      return null
    },
    routes() {
      return routes
    }
  },
  watch: {
    async localRoot(localRoot) {
      this.rootPath = await BrowserTree.getPathFromLocalId(localRoot)
    }
  },
  async created() {
    this.rootPath = await BrowserTree.getPathFromLocalId(this.localRoot)
  },
  methods: {
    onChangeStrategy() {
      this.$store.dispatch(actions.STORE_ACCOUNT, {id: this.account.id, data: this.account.data})
    },
    onTriggerSync() {
      this.$store.dispatch(actions.TRIGGER_SYNC, this.account.id)
    },
    onTriggerSyncUp() {
      this.$store.dispatch(actions.TRIGGER_SYNC_UP, this.account.id)
    },
    onTriggerSyncDown() {
      this.$store.dispatch(actions.TRIGGER_SYNC_DOWN, this.account.id)
    },
    onCancelSync() {
      this.$store.dispatch(actions.CANCEL_SYNC, this.account.id)
    },
    onToggleEnabled() {
      this.$store.dispatch(actions.STORE_ACCOUNT, {id: this.account.id, data: this.account.data})
    },
    onGetLogs() {
      this.$store.dispatch(actions.DOWNLOAD_LOGS)
    }
  }
}
</script>

<style scoped>
.spinning {
  animation: spin 2s infinite linear;
}

@keyframes spin {
  0% {
    transform: rotate(360deg);
  }
  99.9% {
    transform: rotate(0deg);
  }
}
</style>
