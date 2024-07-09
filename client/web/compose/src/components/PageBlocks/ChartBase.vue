<template>
  <wrap
    v-bind="$props"
    v-on="$listeners"
    @refreshBlock="refresh"
  >
    <!-- <div class="d-flex justify-content-end"> -->
    <template>
      <b-button
        variant="outline-light"
        size="lg"
        rounded="full"
        class="livefilter-btn text-primary px-2 mt-2 mr-2"
        @click="liveFilterModal.show = true"
      >
        <font-awesome-icon
          :icon="['fas', 'filter']"
        />
      </b-button>

      <b-modal
        v-model="liveFilterModal.show"
        :title="liveFilterModal.title"
        :ok-title="$t('general:label.saveAndClose')"
        centered
        size="md"
        cancel-variant="light"
        no-fade
        @ok="liveFilterUpdate()"
        @cancel="liveFilterModal.show = undefined"
        @hide="liveFilterModal.show = undefined"
      >
        <div
          class="px-3"
        >
          <b-row>
            <b-col
              cols="12"
              lg="12"
            >
              <b-form-group
                :label="$t('chart:edit.filter.preset')"
                label-class="text-primary"
              >
                <c-input-select
                  v-model="selectedFilter"
                  :options="predefinedFilters"
                  label="text"
                  :reduce="filter => filter.value"
                  :placeholder="$t('chart:edit.filter.noFilter')"
                />
              </b-form-group>
            </b-col>

            <b-col
              v-if="selectedFilter === 'custom'"
              cols="12"
              lg="12"
            >
              <b-row>
                <b-col
                  cols="12"
                  md="6"
                >
                  <b-form-group
                    :label="'Start'"
                    label-class="text-primary"
                  >
                    <c-input-date-time
                      v-model="customDate.start"
                      :no-date="false"
                      :no-time="true"
                      :only-future="false"
                      :only-past="false"
                      :labels="{
                        clear: $t('general:label.clear'),
                        none: $t('general:label.none'),
                        now: $t('general:label.now'),
                        today: $t('general:label.today'),
                      }"
                    />
                  </b-form-group>
                </b-col>
                <b-col
                  cols="12"
                  md="6"
                >
                  <b-form-group
                    :label="'End'"
                    label-class="text-primary"
                  >
                    <c-input-date-time
                      v-model="customDate.end"
                      :no-date="false"
                      :no-time="true"
                      :only-future="false"
                      :only-past="false"
                      :labels="{
                        clear: $t('general:label.clear'),
                        none: $t('general:label.none'),
                        now: $t('general:label.now'),
                        today: $t('general:label.today'),
                      }"
                    />
                  </b-form-group>
                </b-col>
              </b-row>
            </b-col>

            <!-- Configure report filters -->
            <b-col
              cols="12"
              class="mt-1"
            >
              <b-form-group
                :label="$t('chart:edit.filter.label')"
                label-class="text-primary"
              >
                <b-form-textarea
                  v-model="liveFilterValue"
                  :placeholder="$t('chart:edit.filter.placeholder')"
                />

                <i18next
                  path="chart:edit.filter.footnote"
                  tag="small"
                  class="text-muted"
                >
                  <code>${record.values.fieldName}</code>
                  <code>${recordID}</code>
                  <code>${ownerID}</code>
                  <span><code>${userID}</code>, <code>${user.name}</code></span>
                </i18next>
              </b-form-group>
            </b-col>
          </b-row>
        </div>
      </b-modal>
    </template>

    <chart-component
      v-if="chart"
      :key="key"
      :chart="chart"
      :record="record"
      :reporter="reporter"
      @drill-down="drillDown"
    />
  </wrap>
</template>

<script>
import moment from 'moment'
import { mapActions } from 'vuex'
import base from './base'
import ChartComponent from '../Chart'
import { NoID, compose } from '@cortezaproject/corteza-js'
import { evaluatePrefilter, isFieldInFilter } from 'corteza-webapp-compose/src/lib/record-filter'
import { components } from '@cortezaproject/corteza-vue'

const { CInputDateTime } = components

export default {
  i18nOptions: {
    namespaces: 'notification',
  },

  components: {
    ChartComponent,
    CInputDateTime,
  },

  extends: base,

  data () {
    return {
      chart: null,

      filter: undefined,

      drillDownFilter: undefined,

      liveFilterModal: {
        title: 'Live chart filter',
        show: false,
      },

      predefinedFilters: [
        ...compose.chartUtil.predefinedFilters.map(pf => ({ ...pf, text: this.$t(`chart:edit.filter.${pf.text}`) })),
        { text: 'Custom', value: 'custom' },
      ],

      selectedFilter: undefined,

      customDate: {
        start: undefined,
        end: undefined,
      },

      liveFilterValue: undefined,
    }
  },

  watch: {
    options: {
      deep: true,
      handler () {
        this.refresh()
      },
    },
    customDate: {
      deep: true,
      handler (date) {
        const startDateTime = moment(new Date(date.start), 'YYYY-MM-DDTHH:mm:ssZ', true)
        const endDateTime = moment(new Date(date.end), 'YYYY-MM-DDTHH:mm:ssZ', true)
        const name = 'createdAt'

        const dataFmtEntry = (date) => `TIMESTAMP(DATE_FORMAT('${date.format()}', '%Y-%m-%dT%H:%i:00.%f+00:00'))`

        if (startDateTime.isValid() && endDateTime.isValid()) {
          this.liveFilterValue = `TIMESTAMP(DATE_FORMAT(${name}, '%Y-%m-%dT%H:%i:00.%f+00:00'))` + 'BETWEEN' + `${dataFmtEntry(startDateTime)} ${dataFmtEntry(endDateTime)}`
        }
      },
    },
    selectedFilter: {
      deep: true,
      handler (value) {
        if (value !== 'custom') {
          this.liveFilterValue = value
        }
      },

    },
  },

  mounted () {
    this.fetchChart()
    this.refreshBlock(this.refresh)
    this.createEvents()
  },

  beforeDestroy () {
    this.destroyEvents()
    this.setDefaultValues()
  },

  methods: {
    ...mapActions({
      findChartByID: 'chart/findByID',
    }),

    createEvents () {
      this.$root.$on('drill-down-chart', this.drillDown)
      this.$root.$on('module-records-updated', this.refreshOnRelatedRecordsUpdate)
      this.$root.$on('record-field-change', this.refetchOnPrefilterValueChange)
    },

    refetchOnPrefilterValueChange ({ fieldName }) {
      const { filter } = this.filter

      if (isFieldInFilter(fieldName, filter)) {
        this.refresh()
      }
    },

    refreshOnRelatedRecordsUpdate ({ moduleID, notPageID }) {
      if (this.filter.moduleID === moduleID && this.page.pageID !== notPageID) {
        this.refresh()
      }
    },

    async fetchChart (params = {}) {
      const { chartID } = this.options

      if (!chartID) {
        return
      }

      const { namespaceID } = this.namespace

      return this.findChartByID({ chartID, namespaceID, ...params }).then((chart) => {
        this.chart = chart
      }).catch(this.toastErrorHandler(this.$t('chart.loadFailed')))
    },

    reporter (r) {
      console.log('r', r)
      this.filter = r

      let filter = r.filter

      if (this.liveFilterValue) {
        filter = this.liveFilterValue
      }

      if (filter) {
        // If we use ${record} or ${ownerID} and there is no record, resolve empty
        /* eslint-disable no-template-curly-in-string */
        if (!this.record && (filter.includes('${record') || filter.includes('${ownerID}'))) {
          return new Promise((resolve) => resolve([]))
        }

        filter = evaluatePrefilter(filter, {
          record: this.record,
          user: this.$auth.user || {},
          recordID: (this.record || {}).recordID || NoID,
          ownerID: (this.record || {}).ownedBy || NoID,
          userID: (this.$auth.user || {}).userID || NoID,
        })

        this.filter.filter = filter
      }

      const { namespaceID } = this.namespace

      return this.$ComposeAPI.recordReport({ namespaceID, ...r, filter })
    },

    liveFilterUpdate () {
      // call reproter() with the live filter value
      // change the reporter
      this.refresh()
    },

    refresh () {
      this.fetchChart({ force: true }).then(() => {
        this.chart.config.noAnimation = true
        this.key++
      })
    },

    /**
     *
     * Based on drill down configuration, either changes the linked block on the page
     * or opens it in a modal wit the filter and dimensions from the chart and the clicked value
     */
    drillDown ({ trueName, value }) {
      const { chartID, drillDown } = this.options

      if (!drillDown.enabled) {
        return
      }

      const report = this.chart.config.reports[0] || {}
      const { yAxis = {} } = report

      // If trueName exists we use it as value, otherwise we need to look at the actual value based on if it is horizontal or vertical
      let drillDownValue = trueName
      if (!trueName) {
        drillDownValue = yAxis.horizontal ? value[1] : value[0]
      }

      // Get recordListID that is linked
      let { moduleID, dimensions, filter } = this.filter

      // Construct filter
      const dimensionFilter = dimensions ? `(${dimensions} = '${drillDownValue}')` : ''
      filter = filter ? `(${filter})` : ''
      const prefilter = [dimensionFilter, filter].filter(f => f).join(' AND ')

      if (drillDown.blockID) {
        // Use linked record list to display drill down data
        const { pageID = NoID } = this.page
        const { recordID = NoID } = this.record || {}
        // Construct its uniqueID to identify it
        const recordListUniqueID = [pageID, recordID, drillDown.blockID, false].map(v => v || NoID).join('-')

        this.$root.$emit(`drill-down-recordList:${recordListUniqueID}`, prefilter)
      } else {
        const { title } = this.block
        const { fields = [] } = this.options.drillDown.recordListOptions || {}

        // Open in modal
        const block = new compose.PageBlockRecordList({
          title: title ? `${title} - "${drillDownValue}"` : drillDownValue,
          blockID: `drillDown-${chartID}`,
          options: {
            moduleID,
            fields,
            prefilter,
            presort: 'createdAt DESC',
            hideRecordReminderButton: true,
            hideRecordViewButton: false,
            hideConfigureFieldsButton: false,
            hideImportButton: true,
            enableRecordPageNavigation: true,
            selectable: true,
            allowExport: true,
            perPage: 14,
            showTotalCount: true,
            recordDisplayOption: 'modal',
          },
        })

        this.$root.$emit('magnify-page-block', { block })
      }
    },

    setDefaultValues () {
      this.chart = null
      this.filter = undefined
      this.drillDownFilter = undefined
    },

    destroyEvents () {
      this.$root.$off('drill-down-chart', this.drillDown)
      this.$root.$off('module-records-updated', this.refreshOnRelatedRecordsUpdate)
      this.$root.$off('record-field-change', this.refetchOnPrefilterValueChange)
    },
  },
}
</script>

<style lang="scss" scoped>
.livefilter-btn {
  /* text-primary px-2 mt-2 mr-2 position-absolute right-0 z-1 */
  display: flex;
  align-items: center;
  margin-left: auto;
  border: 0;
  position: absolute;
  right: 0;
  z-index: 1;
}
</style>
