<template>
  <b-spinner
    v-if="processing"
    variant="primary"
    small
  />

  <div v-else>
    <span
      v-for="(v, index) of formattedValue"
      :key="index"
      class="text-nowrap"
      :class="{ 'd-block': field.options.multiDelimiter === '\n' }"
      @click.stop
    >
      <a
        v-if="['modal', 'newTab'].includes(extraOptions.recordSelectorDisplayOption)"
        href="#"
        :class="{ 'text-decoration-none default-cursor': !v.to}"
        @click="(e) => onRecordSelectorClick(e, v.to)"
      >
        {{ v.value }}{{ index !== formattedValue.length - 1 ? field.options.multiDelimiter : '' }}
      </a>

      <router-link
        v-else
        :to="v.to"
        :class="{ 'text-decoration-none default-cursor': !v.to}"
      >
        {{ v.value }}{{ index !== formattedValue.length - 1 ? field.options.multiDelimiter : '' }}
      </router-link>
    </span>
  </div>
</template>

<script>
import base from './base'
import { compose, NoID } from '@cortezaproject/corteza-js'
import { mapActions, mapGetters } from 'vuex'

export default {
  extends: base,

  data () {
    return {
      processing: false,

      recordValues: {},

      relRecords: [],
    }
  },

  computed: {
    ...mapGetters({
      pages: 'page/set',
      findUserByID: 'user/findByID',
      findRecordsByIDs: 'record/findByIDs',
    }),

    formattedValue () {
      const value = Array.isArray(this.value) ? this.value : [this.value].filter(v => v) || []
      return value.map(recordID => {
        return {
          to: this.linkToRecord(recordID),
          value: this.recordValues[recordID] || recordID,
        }
      })
    },

    recordPage () {
      return this.pages.find(p => p.moduleID === this.field.options.moduleID)
    },
  },

  watch: {
    value: {
      immediate: true,
      handler (value) {
        this.formatRecordValues(value)
      },
    },
  },

  beforeDestroy () {
    this.setDefaultValues()
  },

  methods: {
    ...mapActions({
      findModuleByID: 'module/findByID',
      resolveUsers: 'user/resolveUsers',
      resolveRecords: 'record/resolveRecords',
    }),

    linkToRecord (recordID) {
      if (!this.recordPage || !recordID) {
        return ''
      }

      return {
        name: 'page.record',
        params: {
          pageID: this.recordPage.pageID,
          recordID: recordID,
        },
      }
    },

    async formatRecordValues (recordIDs) {
      recordIDs = Array.isArray(recordIDs) ? recordIDs : [recordIDs].filter(v => v) || []
      const { namespaceID = NoID } = this.namespace
      const { moduleID = NoID, labelField, recordLabelField } = this.field.options

      if (!recordIDs.length || [moduleID, namespaceID].includes(NoID) || !labelField) {
        return
      }

      return this.findModuleByID({ namespace: this.namespace, moduleID }).then(async module => {
        const relatedField = module.fields.find(({ name }) => name === labelField)
        const records = this.findRecordsByIDs(recordIDs).map(r => new compose.Record(module, r))
        const mappedIDs = {}

        if (relatedField.kind === 'Record' && recordLabelField) {
          this.processing = true

          const relatedModule = await this.findModuleByID({ namespaceID, moduleID: relatedField.options.moduleID })
          const relatedRecordIDs = new Set()

          records.forEach(r => {
            const recordValue = relatedField.isMulti ? r.values[relatedField.name] : [r.values[relatedField.name]]
            recordValue.forEach(rID => relatedRecordIDs.add(rID))
          })
          await this.resolveRecords({ namespaceID, moduleID: relatedModule.moduleID, recordIDs: [...relatedRecordIDs] })

          const relatedLabelField = relatedModule.fields.find(({ name }) => name === recordLabelField)

          for (let r of await this.findRecordsByIDs([...relatedRecordIDs])) {
            r = new compose.Record(relatedModule, r)
            let relatedRecordValue = relatedLabelField.isMulti ? r.values[relatedLabelField.name] : [r.values[relatedLabelField.name]]

            if (relatedLabelField.kind === 'User') {
              await this.resolveUsers(relatedRecordValue)
              relatedRecordValue = relatedRecordValue.map(v => relatedLabelField.formatter(this.findUserByID(v)))
            }

            mappedIDs[r.recordID] = relatedRecordValue.join(relatedLabelField.options.multiDelimiter)
            relatedRecordIDs.clear()
          }
        } else if (relatedField.kind === 'User') {
          this.processing = true

          const relatedUserIDs = new Set()
          records.forEach(r => {
            const recordValue = relatedField.isMulti ? r.values[relatedField.name] : [r.values[relatedField.name]]
            recordValue.forEach(uID => relatedUserIDs.add(uID))
          })

          await this.resolveUsers([...relatedUserIDs])
        }

        records.forEach(record => {
          let recordValue = relatedField.isMulti ? record.values[relatedField.name] : [record.values[relatedField.name]]

          if (relatedField.kind === 'User') {
            recordValue = recordValue.map(v => relatedField.formatter(this.findUserByID(v)))
          } else if (relatedField.kind === 'Record' && recordLabelField) {
            recordValue = recordValue.map(v => mappedIDs[v])
          }

          this.$set(this.recordValues, record.recordID, recordValue.join(relatedField.options.multiDelimiter))
        })
      }).finally(() => {
        setTimeout(() => {
          this.processing = false
        }, 300)
      })
    },

    onRecordSelectorClick (e, route) {
      e.preventDefault()

      if (this.extraOptions.recordSelectorDisplayOption === 'modal') {
        this.$root.$emit('show-record-modal', {
          recordID: route.params.recordID,
          recordPageID: route.params.pageID,
        })
      } else if (this.extraOptions.recordSelectorDisplayOption === 'newTab') {
        window.open(this.$router.resolve(route).href, '_blank')
      }
    },

    setDefaultValues () {
      this.processing = false
      this.recordValues = {}
      this.relRecords = []
    },
  },
}
</script>

<style lang="scss" scoped>
.default-cursor {
  cursor: default;
}
</style>
