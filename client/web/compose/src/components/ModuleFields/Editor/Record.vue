<template>
  <b-form-group
    :label-cols-md="horizontal && '5'"
    :label-cols-xl="horizontal && '4'"
    :content-cols-md="horizontal && '7'"
    :content-cols-xl="horizontal && '8'"
    :class="formGroupStyleClasses"
  >
    <template
      #label
    >
      <div
        v-if="!valueOnly"
        class="d-flex align-items-center text-primary p-0"
      >
        <span
          class="d-inline-block mw-100 py-1"
          :title="label"
        >
          {{ label }}
        </span>

        <hint
          :text="hint"
        />

        <slot name="tools" />
      </div>
      <div
        class="small text-muted"
        :class="{ 'mb-1': description }"
      >
        {{ description }}
      </div>
    </template>

    <multi
      v-if="field.isMulti"
      :value.sync="value"
      :errors="errors"
      :single-input="field.options.selectType !== 'each'"
      :removable="field.options.selectType !== 'multiple'"
    >
      <template v-slot:single>
        <vue-select
          v-if="field.options.selectType === 'multiple'"
          v-model="multipleSelected"
          :options="options"
          :get-option-key="getOptionKey"
          :disabled="!module"
          :loading="processing"
          option-value="recordID"
          option-text="label"
          :append-to-body="appendToBody"
          :calculate-position="calculateDropdownPosition"
          :clearable="false"
          :filterable="false"
          :searchable="searchable"
          :selectable="option => option.selectable"
          class="bg-white w-100"
          :placeholder="placeholder"
          multiple
          @search="search"
        >
          <pagination
            v-if="showPagination"
            slot="list-footer"
            :has-prev-page="hasPrevPage"
            :has-next-page="hasNextPage"
            @prev="goToPage(false)"
            @next="goToPage(true)"
          />
        </vue-select>
        <vue-select
          v-else
          ref="singleSelect"
          :options="options"
          :get-option-key="getOptionKey"
          :disabled="!module"
          :loading="processing"
          option-value="recordID"
          option-text="label"
          :append-to-body="appendToBody"
          :calculate-position="calculateDropdownPosition"
          :clearable="false"
          :filterable="false"
          :searchable="searchable"
          :selectable="option => option.selectable"
          class="bg-white w-100"
          :placeholder="placeholder"
          @input="selectChange($event)"
          @search="search"
        >
          <pagination
            v-if="showPagination"
            slot="list-footer"
            :has-prev-page="hasPrevPage"
            :has-next-page="hasNextPage"
            @prev="goToPage(false)"
            @next="goToPage(true)"
          />
        </vue-select>
      </template>
      <template v-slot:default="ctx">
        <vue-select
          v-if="field.options.selectType === 'each'"
          :options="options"
          :get-option-key="getOptionKey"
          :disabled="!module"
          :loading="processing"
          option-value="recordID"
          option-text="label"
          :append-to-body="appendToBody"
          :calculate-position="calculateDropdownPosition"
          :clearable="false"
          :filterable="false"
          :searchable="searchable"
          :selectable="option => option.selectable"
          class="bg-white w-100"
          :placeholder="placeholder"
          :value="getRecord(ctx.index)"
          @input="setRecord($event, ctx.index)"
          @search="search"
        >
          <pagination
            v-if="showPagination"
            slot="list-footer"
            :has-prev-page="hasPrevPage"
            :has-next-page="hasNextPage"
            @prev="goToPage(false)"
            @next="goToPage(true)"
          />
        </vue-select>
        <b-spinner
          v-else-if="processing"
          variant="primary"
          small
        />
        <span v-else>
          {{ (multipleSelected[ctx.index] || {}).label }}
        </span>
      </template>
    </multi>
    <template
      v-else
    >
      <vue-select
        v-model="selected"
        :options="options"
        :get-option-key="getOptionKey"
        :disabled="!module"
        :loading="processing"
        option-value="recordID"
        option-text="label"
        :append-to-body="appendToBody"
        :calculate-position="calculateDropdownPosition"
        :placeholder="placeholder"
        :filterable="false"
        :searchable="searchable"
        :selectable="option => option.selectable"
        class="bg-white w-100"
        @search="search"
      >
        <pagination
          v-if="showPagination"
          slot="list-footer"
          :has-prev-page="hasPrevPage"
          :has-next-page="hasNextPage"
          @prev="goToPage(false)"
          @next="goToPage(true)"
        />
      </vue-select>
      <errors :errors="errors" />
    </template>
  </b-form-group>
</template>
<script>
import base from './base'
import { debounce } from 'lodash'
import { compose, NoID } from '@cortezaproject/corteza-js'
import { mapActions, mapGetters } from 'vuex'
import { VueSelect } from 'vue-select'
import { evaluatePrefilter } from 'corteza-webapp-compose/src/lib/record-filter'
import Pagination from '../Common/Pagination.vue'

export default {
  i18nOptions: {
    namespaces: 'field',
  },

  components: {
    VueSelect,
    Pagination,
  },

  extends: base,

  data () {
    return {
      processing: false,

      query: '',

      records: [],

      recordValues: {},

      filter: {
        query: '',
        sort: '',
        limit: 10,
        pageCursor: '',
        prevPage: '',
        nextPage: '',
      },
    }
  },

  computed: {
    ...mapGetters({
      getModuleByID: 'module/getByID',
      findUserByID: 'user/findByID',
      findRecordsByIDs: 'record/findByIDs',
    }),

    options () {
      return this.records.map(this.convert).filter(({ value, label }) => value && label)
    },

    module () {
      if (this.field.options.moduleID !== NoID) {
        return this.getModuleByID(this.field.options.moduleID)
      } else {
        return undefined
      }
    },

    searchable () {
      return !this.field.options.recordLabelField
    },

    placeholder () {
      return this.searchable ? this.$t('kind.record.suggestionPlaceholder') : this.$t('kind.select.placeholder')
    },

    multipleSelected: {
      get () {
        return this.value.map(v => this.convert({ recordID: v })).filter(({ value, label }) => value && label)
      },

      set (value) {
        if (value.length !== this.value.length) {
          this.value = value.map(({ value }) => value)
        }
      },
    },

    selected: {
      get () {
        return this.getRecord()
      },

      set (value) {
        this.setRecord(value)
      },
    },

    showPagination () {
      return this.hasPrevPage || this.hasNextPage
    },

    hasPrevPage () {
      return !!this.filter.prevPage
    },

    hasNextPage () {
      return !!this.filter.nextPage
    },
  },

  watch: {
    'filter.pageCursor': {
      handler (pageCursor) {
        if (pageCursor) {
          this.fetchPrefiltered(this.filter)
        }
      },
    },
  },

  created () {
    this.loadLatest()
    if (this.value) {
      this.formatRecordValues(this.value)
    }
  },

  beforeDestroy () {
    this.setDefaultValues()
  },

  methods: {
    ...mapActions({
      findModuleByID: 'module/findByID',
      resolveUsers: 'user/resolveUsers',
      resolveRecords: 'record/resolveRecords',
      updateRecords: 'record/updateRecords',
    }),

    getRecord (index = undefined) {
      const value = index !== undefined ? this.value[index] : this.value
      if (value) {
        return this.convert({ recordID: value })
      }
    },

    setRecord (event, index = undefined) {
      const crtValue = index !== undefined ? this.value[index] : this.value
      const { value } = event || {}
      if (value !== crtValue) {
        if (value) {
          if (index !== undefined) {
            this.value.splice(index, 1, value)
          } else {
            this.value = value
          }
        } else {
          if (index !== undefined) {
            this.value.splice(index, 1)
          } else {
            this.value = undefined
          }
        }
      }
    },

    convert (r) {
      if (!r || !this.field.options.labelField) {
        return {}
      }

      return {
        value: r.recordID,
        label: this.processing ? '' : this.recordValues[r.recordID] || r.recordID,
        selectable: this.field.isMulti && !this.field.options.isUniqueMultiValue
          ? this.value !== r.recordID
          : !(this.value || []).includes(r.recordID),
      }
    },

    search: debounce(function (query = '') {
      if (query !== this.query) {
        this.query = query
        this.filter.pageCursor = undefined
      }

      const { limit, pageCursor } = this.filter
      const namespaceID = this.namespace.namespaceID
      const moduleID = this.field.options.moduleID

      if (moduleID && moduleID !== NoID) {
        // Determine what fields to use for searching
        // Default to label field
        let qf = this.field.options.queryFields
        if (!qf || qf.length === 0) {
          qf = [this.field.options.labelField]
        }

        if (query.length > 0) {
          // Construct query
          query = qf.map(qf => {
            return `${qf} LIKE '%${query}%'`
          }).join(' OR ')
        }

        this.fetchPrefiltered({ namespaceID, moduleID, query, sort: this.sortString(), limit, pageCursor })
      }
    }, 300),

    loadLatest () {
      const namespaceID = this.namespace.namespaceID
      const moduleID = this.field.options.moduleID
      const { limit } = this.filter
      if (moduleID && moduleID !== NoID) {
        this.fetchPrefiltered({ namespaceID, moduleID, limit })
      }
    },

    fetchPrefiltered (q) {
      this.processing = true

      // Support prefilters
      let { query = '' } = q
      if (this.field.options.prefilter) {
        const pf = evaluatePrefilter(this.field.options.prefilter, {
          record: this.record,
          recordID: (this.record || {}).recordID || NoID,
          ownerID: (this.record || {}).ownedBy || NoID,
          userID: (this.$auth.user || {}).userID || NoID,
        })
        if (query) {
          query = `(${pf}) AND (${query})`
        } else {
          query = pf
        }
      }

      if (q.pageCursor) {
        q.sort = ''
      }

      this.$ComposeAPI.recordList({ ...q, query })
        .then(({ filter, set }) => {
          this.filter = { ...this.filter, ...filter }
          this.filter.nextPage = filter.nextPage
          this.filter.prevPage = filter.prevPage

          this.updateRecords(set)

          return this.formatRecordValues(set.map(({ recordID }) => recordID)).then(() => {
            this.records = set.map(r => new compose.Record(this.module, r))
          })
        })
        .finally(() => {
          this.processing = false
        })
    },

    sortString () {
      return [this.field.options.labelField].filter(f => !!f).join(', ')
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

    selectChange (event) {
      const { value } = event || {}
      if (value) {
        this.value.push(value)

        // reset singleSelect value for better value presentation
        if (this.$refs.singleSelect) {
          this.$refs.singleSelect._data._value = undefined
        }
      }
    },

    goToPage (next = true) {
      this.filter.pageCursor = next ? this.filter.nextPage : this.filter.prevPage
    },

    getOptionKey ({ value }) {
      return value
    },

    setDefaultValues () {
      this.processing = false
      this.query = ''
      this.records = []
      this.recordValues = {}
      this.filter = {}
    },
  },
}
</script>
