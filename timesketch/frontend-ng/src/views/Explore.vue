<!--
Copyright 2021 Google Inc. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->
<template>
  <v-container fluid>
    <!-- Right side menu -->
    <!-- Placeholder at the moment. Keeping it here for quick developement later. -->
    <v-navigation-drawer v-if="showRightSidePanel" fixed right width="600" style="box-shadow: 0 10px 15px -3px #888">
      <template v-slot:prepend>
        <v-toolbar flat>
          <v-toolbar-title>Right Side Panel</v-toolbar-title>
          <v-spacer></v-spacer>
          <v-btn icon @click="showRightSidePanel = false">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </v-toolbar>
      </template>
      <v-container> TODO: Add content here </v-container>
    </v-navigation-drawer>

    <!-- Search and Filters -->
    <v-card flat class="pa-3 pt-0 mt-n3" color="transparent">
      <v-card class="d-flex align-start mb-1" outlined>
        <v-sheet class="mt-2">
          <ts-search-history-buttons @toggleSearchHistory="toggleSearchHistory()"></ts-search-history-buttons>
        </v-sheet>

        <v-menu v-model="showSearchDropdown" offset-y attach :close-on-content-click="false" :close-on-click="true">
          <template v-slot:activator="{ on, attrs }">
            <v-text-field
              v-model="currentQueryString"
              hide-details
              label="Search"
              placeholder="Search"
              single-line
              dense
              filled
              flat
              solo
              class="pa-1"
              append-icon="mdi-magnify"
              @click:append="search()"
              id="tsSearchInput"
              @keyup.enter="search()"
              @click="showSearchDropdown = true"
              ref="searchInput"
              v-bind="attrs"
              v-on="on"
            >
            </v-text-field>
          </template>

          <ts-search-dropdown
            v-click-outside="onClickOutside"
            :selected-labels="selectedLabels"
            :query-string="currentQueryString"
            @setActiveView="searchView"
            @addChip="addChip"
            @updateLabelChips="updateLabelChips()"
            @close-on-click="showSearchDropdown = false"
            @node-click="jumpInHistory"
            @setQueryAndFilter="setQueryAndFilter"
          >
          </ts-search-dropdown>
        </v-menu>
      </v-card>

      <!-- Search History -->
      <div class="mt-4">
        <v-card v-show="showSearchHistory" outlined>
          <v-toolbar dense flat>
            <v-toolbar-title>Search history</v-toolbar-title>
            <v-spacer></v-spacer>
            <v-slider
              v-model="zoomLevel"
              thumb-label
              ticks
              append-icon="mdi-magnify-plus-outline"
              prepend-icon="mdi-magnify-minus-outline"
              min="0.1"
              max="1"
              step="0.1"
              class="mt-6"
            >
              <template v-slot:thumb-label="{ value }"> {{ value * 100 }}% </template>
            </v-slider>

            <v-btn icon @click="showSearchHistory = false" class="ml-4">
              <v-icon>mdi-close</v-icon>
            </v-btn>
          </v-toolbar>

          <v-divider></v-divider>

          <div
            v-dragscroll
            class="pa-md-4 no-scrollbars"
            style="overflow: scroll; white-space: nowrap; max-height: 500px; min-height: 100px"
          >
            <ts-search-history-tree
              @node-click="jumpInHistory"
              :show-history="showSearchHistory"
              v-bind:style="{ transform: 'scale(' + zoomLevel + ')' }"
              style="transform-origin: top left"
            ></ts-search-history-tree>
          </div>
        </v-card>
      </div>

      <!-- Timeline picker -->
      <v-sheet class="mb-4 mt-4" color="transparent">
        <ts-timeline-picker
          @updateSelectedTimelines="updateSelectedTimelines($event)"
          :current-query-filter="currentQueryFilter"
          :count-per-index="countPerIndex"
          :count-per-timeline="countPerTimeline"
        ></ts-timeline-picker>

        <span style="position: relative; top: -5px">
          <ts-upload-timeline-form btn-type="small"></ts-upload-timeline-form>
        </span>

        <span style="position: relative; top: -5px">
          <v-dialog v-model="addManualEvent" width="600">
            <template v-slot:activator="{ on, attrs }">
              <v-btn small text rounded color="primary" v-bind="attrs" v-on="on">
                <v-icon left small> mdi-plus </v-icon>
                Add manual event
              </v-btn>
            </template>
            <ts-add-manual-event
              app
              @cancel="addManualEvent = false"
              :datetimeProp="datetimeManualEvent"
            ></ts-add-manual-event>
          </v-dialog>
        </span>
      </v-sheet>

      <!-- Time filter chips -->
      <div class="mt-n3">
        <span v-for="(chip, index) in timeFilterChips" :key="index + chip.value">
          <v-menu offset-y content-class="menu-with-gap">
            <template v-slot:activator="{ on }">
              <v-chip outlined v-on="on">
                <v-icon left small> mdi-clock-outline </v-icon>
                <span v-bind:style="[!chip.active ? { 'text-decoration': 'line-through', opacity: '50%' } : '']">
                  <span>{{ chip.value.split(',')[0] }}</span>
                  <span v-if="chip.type === 'datetime_range' && chip.value.split(',')[0] !== chip.value.split(',')[1]">
                    &rarr; {{ chip.value.split(',')[1] }}</span
                  >
                </span>
              </v-chip>
            </template>
            <v-card>
              <v-list>
                <!-- Edit timefilter menu -->
                <v-menu
                  offset-y
                  :close-on-content-click="false"
                  :close-on-click="true"
                  nudge-top="70"
                  content-class="menu-with-gap"
                  allow-overflow
                  style="overflow: visible"
                >
                  <template v-slot:activator="{ on, attrs }">
                    <v-list-item v-bind="attrs" v-on="on">
                      <v-list-item-action>
                        <v-icon>mdi-square-edit-outline</v-icon>
                      </v-list-item-action>
                      <v-list-item-subtitle>Edit filter</v-list-item-subtitle>
                    </v-list-item>
                  </template>
                  <ts-filter-menu app :selected-chip="chip" @updateChip="updateChip($event, chip)"></ts-filter-menu>
                </v-menu>

                <v-list-item @click="toggleChip(chip)">
                  <v-list-item-action>
                    <v-icon v-if="chip.active">mdi-eye-off</v-icon>
                    <v-icon v-else>mdi-eye</v-icon>
                  </v-list-item-action>
                  <v-list-item-subtitle
                    ><span v-if="chip.active">Temporarily disable</span
                    ><span v-else>Re-enable</span></v-list-item-subtitle
                  >
                </v-list-item>
                <v-list-item @click="removeChip(chip)">
                  <v-list-item-action>
                    <v-icon>mdi-delete</v-icon>
                  </v-list-item-action>
                  <v-list-item-subtitle>Remove filter</v-list-item-subtitle>
                </v-list-item>
              </v-list>
            </v-card>
          </v-menu>
          <v-btn v-if="index + 1 < timeFilterChips.length" icon small style="margin-top: 2px" class="mr-2">OR</v-btn>
        </span>
        <span>
          <v-menu
            v-model="timeFilterMenu"
            offset-y
            :close-on-content-click="false"
            :close-on-click="true"
            content-class="menu-with-gap"
            allow-overflow
            style="overflow: visible"
          >
            <template v-slot:activator="{ on, attrs }">
              <v-btn small text rounded color="primary" v-bind="attrs" v-on="on">
                <v-icon left small> mdi-plus </v-icon>
                Add timefilter
              </v-btn>
            </template>

            <ts-filter-menu app @cancel="timeFilterMenu = false" @addChip="addChip"></ts-filter-menu>
          </v-menu>
        </span>
      </div>

      <!-- Term filters -->
      <div v-if="filterChips.length">
        <v-chip-group>
          <span v-for="(chip, index) in filterChips" :key="index + chip.value">
            <v-chip small outlined close @click:close="removeChip(chip)">
              <v-icon v-if="chip.value === '__ts_star'" left small color="amber">mdi-star</v-icon>
              <v-icon v-if="chip.value === '__ts_comment'" left small>mdi-comment-multiple-outline</v-icon>
              {{ chip.value | formatLabelText }}
            </v-chip>
            <v-btn v-if="index + 1 < timeFilterChips.length" icon small style="margin-top: 2px" class="mr-2">AND</v-btn>
          </span>
        </v-chip-group>
      </div>
    </v-card>

    <!-- DFIQ context -->
    <div class="mt-3 mx-3">
      <div :class="[$vuetify.theme.dark ? 'dark-info-card' : 'light-info-card']" v-if="activeContext.question">
        <v-toolbar dense flat color="transparent">
          <h4>{{ activeContext.question.display_name }}</h4>
          <v-spacer></v-spacer>
          <v-btn small icon @click="$store.dispatch('clearActiveContext')" class="mr-1">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </v-toolbar>
        <p class="mt-1 pb-4 px-4" style="font-size: 0.9em">
          {{ activeContext.question.description }}
        </p>
      </div>
    </div>

    <!-- Eventlist -->
    <v-card flat class="mt-5 mx-3">
      <ts-event-list
        :query-request="activeQueryRequest"
        @countPerIndex="updateCountPerIndex($event)"
        @countPerTimeline="updateCountPerTimeline($event)"
      ></ts-event-list>
    </v-card>
  </v-container>
</template>

<script>
import ApiClient from '../utils/RestApiClient'
import EventBus from '../main'

import { dragscroll } from 'vue-dragscroll'

import TsSearchHistoryTree from '../components/Explore/SearchHistoryTree'
import TsSearchHistoryButtons from '../components/Explore/SearchHistoryButtons'
import TsSearchDropdown from '../components/Explore/SearchDropdown'
import TsTimelinePicker from '../components/Explore/TimelinePicker'
import TsFilterMenu from '../components/Explore/FilterMenu'
import TsUploadTimelineForm from '../components/UploadForm'
import TsAddManualEvent from '../components/Explore/AddManualEvent'
import TsEventList from '../components/Explore/EventList'

const defaultQueryFilter = () => {
  return {
    from: 0,
    terminate_after: 40,
    size: 40,
    indices: [],
    order: 'asc',
    chips: [],
  }
}

export default {
  directives: {
    dragscroll,
  },
  components: {
    TsSearchHistoryTree,
    TsSearchHistoryButtons,
    TsSearchDropdown,
    TsTimelinePicker,
    TsFilterMenu,
    TsUploadTimelineForm,
    TsAddManualEvent,
    TsEventList,
  },
  props: ['sketchId'],
  data() {
    return {
      countPerIndex: {},
      countPerTimeline: {},
      currentItemsPerPage: 40,
      timeFilterMenu: false,
      selectedFields: [{ field: 'message', type: 'text' }],
      showRightSidePanel: false,
      addManualEvent: false,
      datetimeManualEvent: '',
      params: {},
      contextEvent: false,
      originalContext: false,
      showSearchDropdown: false,
      activeQueryRequest: {},
      currentQueryString: '',
      currentQueryFilter: defaultQueryFilter(),
      selectedLabels: [],
      showSearchHistory: false,
      zoomLevel: 0.7,
      zoomOrigin: {
        x: 0,
        y: 0,
      },
    }
  },
  computed: {
    sketch() {
      return this.$store.state.sketch
    },
    meta() {
      return this.$store.state.meta
    },
    filterChips: function () {
      return this.currentQueryFilter.chips.filter((chip) => chip.type === 'label' || chip.type === 'term')
    },
    timeFilterChips: function () {
      return this.currentQueryFilter.chips.filter((chip) => chip.type.startsWith('datetime'))
    },
    filteredLabels() {
      return this.$store.state.meta.filter_labels.filter((label) => !label.label.startsWith('__'))
    },
    currentSearchNode() {
      return this.$store.state.currentSearchNode
    },
    activeContext() {
      return this.$store.state.activeContext
    },
  },
  methods: {
    updateCountPerIndex: function (count) {
      this.countPerIndex = count
    },
    updateCountPerTimeline: function (count) {
      this.countPerTimeline = count
    },
    toggleSearchHistory: function () {
      this.showSearchHistory = !this.showSearchHistory
      if (this.showSearchHistory) {
        this.triggerScrollTo()
      }
    },
    setQueryAndFilter: function (searchEvent) {
      if (this.$route.name !== 'Explore') {
        this.$router.push({ name: 'Explore', params: { sketchId: this.sketch.id } })
      }
      this.currentQueryString = searchEvent.queryString
      this.currentQueryFilter = searchEvent.queryFilter
      // Preserve user defined item count instead of resetting.
      this.currentQueryFilter.size = this.currentItemsPerPage
      this.currentQueryFilter.terminate_after = this.currentItemsPerPage
      if (searchEvent.doSearch) {
        this.search()
      }
    },
    search: function (resetPagination = true, incognito = false, parent = false) {
      let queryRequest = {}
      queryRequest['queryString'] = this.currentQueryString
      queryRequest['queryFilter'] = this.currentQueryFilter
      queryRequest['resetPagination'] = resetPagination
      queryRequest['incognito'] = incognito
      queryRequest['parent'] = parent
      this.activeQueryRequest = queryRequest
      this.showSearchDropdown = false
    },
    searchView: function (viewId) {
      this.showSearchDropdown = false

      if (this.$route.name !== 'Explore') {
        this.$router.push({ name: 'Explore', params: { sketchId: this.sketch.id } })
      }

      if (viewId !== parseInt(viewId, 10) && typeof viewId !== 'string') {
        viewId = viewId.id
      }

      ApiClient.getView(this.sketchId, viewId)
        .then((response) => {
          let view = response.data.objects[0]
          this.currentQueryString = view.query_string
          this.currentQueryFilter = JSON.parse(view.query_filter)
          if (!this.currentQueryFilter.fields || !this.currentQueryFilter.fields.length) {
            this.currentQueryFilter.fields = [{ field: 'message', type: 'text' }]
          }
          this.selectedFields = this.currentQueryFilter.fields
          if (this.currentQueryFilter.indices[0] === '_all' || this.currentQueryFilter.indices === '_all') {
            let allIndices = []
            this.sketch.active_timelines.forEach((timeline) => {
              let isLegacy = this.meta.indices_metadata[timeline.searchindex.index_name].is_legacy
              if (isLegacy) {
                allIndices.push(timeline.searchindex.index_name)
              } else {
                allIndices.push(timeline.id)
              }
            })
            this.currentQueryFilter.indices = allIndices
          }
          let chips = this.currentQueryFilter.chips
          if (chips) {
            for (let i = 0; i < chips.length; i++) {
              if (chips[i].type === 'label') {
                this.selectedLabels.push(chips[i].value)
              }
            }
          }
          this.contextEvent = false
          this.search()
        })
        .catch((e) => {})
    },
    searchContext: function (event) {
      // TODO: Make this selectable in the UI
      const contextTime = 300
      const numContextEvents = 500

      this.contextEvent = event
      if (!this.originalContext) {
        let currentQueryStringCopy = JSON.parse(JSON.stringify(this.currentQueryString))
        let currentQueryFilterCopy = JSON.parse(JSON.stringify(this.currentQueryFilter))
        this.originalContext = { queryString: currentQueryStringCopy, queryFilter: currentQueryFilterCopy }
      }

      const dateTimeTemplate = 'YYYY-MM-DDTHH:mm:ss'
      let startDateTimeMoment = this.$moment.utc(this.contextEvent._source.datetime)
      let newStartDate = startDateTimeMoment.clone().subtract(contextTime, 's').format(dateTimeTemplate)
      let newEndDate = startDateTimeMoment.clone().add(contextTime, 's').format(dateTimeTemplate)
      let startChip = {
        field: '',
        value: newStartDate + ',' + startDateTimeMoment.format(dateTimeTemplate),
        type: 'datetime_range',
        operator: 'must',
        active: true,
      }
      let endChip = {
        field: '',
        value: startDateTimeMoment.format(dateTimeTemplate) + ',' + newEndDate,
        type: 'datetime_range',
        operator: 'must',
        active: true,
      }
      // TODO: Use chips instead
      this.currentQueryString = '* OR ' + '_id:' + this.contextEvent._id

      this.currentQueryFilter.chips = [startChip, endChip]

      let isLegacy = this.meta.indices_metadata[this.contextEvent._index].is_legacy
      if (isLegacy) {
        this.currentQueryFilter.indices = [this.contextEvent._index]
      } else {
        this.currentQueryFilter.indices = [this.contextEvent._source.__ts_timeline_id]
      }
      this.currentQueryFilter.size = numContextEvents
      this.search()
    },
    removeContext: function () {
      this.contextEvent = false
      this.currentQueryString = JSON.parse(JSON.stringify(this.originalContext.queryString))
      this.currentQueryFilter = JSON.parse(JSON.stringify(this.originalContext.queryFilter))
      this.search()
    },
    updateSelectedTimelines: function (timelines) {
      let selected = []
      timelines.forEach((timeline) => {
        let isLegacy = this.meta.indices_metadata[timeline.searchindex.index_name].is_legacy
        if (isLegacy) {
          selected.push(timeline.searchindex.index_name)
        } else {
          selected.push(timeline.id)
        }
      })
      this.currentQueryFilter.indices = selected
      this.search()
    },
    toggleChip: function (chip) {
      // Treat undefined as active to support old chip formats.
      if (chip.active === undefined) {
        chip.active = true
      }
      chip.active = !chip.active
      this.search()
    },
    removeChip: function (chip, search = true) {
      let chipIndex = this.currentQueryFilter.chips.findIndex((c) => c.value === chip.value)
      this.currentQueryFilter.chips.splice(chipIndex, 1)
      if (chip.type === 'label') {
        this.selectedLabels = this.selectedLabels.filter((label) => label !== chip.value)
      }
      if (search) {
        this.search()
      }
    },
    updateChip: function (newChip, oldChip) {
      // Replace the chip at the given index
      let chipIndex = this.currentQueryFilter.chips.findIndex(
        (c) => c.value === oldChip.value && c.type === oldChip.type
      )
      this.currentQueryFilter.chips.splice(chipIndex, 1, newChip)
      this.search()
    },
    addChip: function (chip) {
      // Legacy views don't support chips so we need to add an array in order
      // to upgrade the view to the new filter system.
      if (!this.currentQueryFilter.chips) {
        this.currentQueryFilter.chips = []
      }
      this.currentQueryFilter.chips.push(chip)
      this.search()
    },
    addChipFromHistogram: function (chip) {
      if (!this.currentQueryFilter.chips) {
        this.currentQueryFilter.chips = []
      }
      this.currentQueryFilter.chips.forEach((chip) => {
        if (chip.type === 'datetime_range') {
          this.removeChip(chip, false)
        }
      })
      this.addChip(chip)
    },
    toggleLabelChip: function (labelName) {
      let chip = {
        field: '',
        value: labelName,
        type: 'label',
        operator: 'must',
        active: true,
      }
      let chips = this.currentQueryFilter.chips
      if (chips) {
        for (let i = 0; i < chips.length; i++) {
          if (chips[i].value === labelName) {
            this.removeChip(i)
            return
          }
        }
      }
      this.addChip(chip)
    },
    updateLabelChips: function () {
      // Remove all current label chips
      this.currentQueryFilter.chips = this.currentQueryFilter.chips.filter((chip) => chip.type !== 'label')
      this.selectedLabels.forEach((label) => {
        let chip = {
          field: '',
          value: label,
          type: 'label',
          operator: 'must',
          active: true,
        }
        this.addChip(chip)
        this.showSearchDropdown = false
      })
    },
    updateLabelList: function (label) {
      if (this.meta.filter_labels.indexOf(label) === -1) {
        this.meta.filter_labels.push(label)
      }
    },
    jumpInHistory: function (node) {
      this.currentQueryString = node.query_string
      this.currentQueryFilter = JSON.parse(node.query_filter)
      if (!this.currentQueryFilter.fields || !this.currentQueryFilter.fields.length) {
        this.currentQueryFilter.fields = [{ field: 'message', type: 'text' }]
      }
      this.selectedFields = this.currentQueryFilter.fields
      if (this.currentQueryFilter.indices[0] === '_all' || this.currentQueryFilter.indices === '_all') {
        let allIndices = []
        this.sketch.active_timelines.forEach((timeline) => {
          let isLegacy = this.meta.indices_metadata[timeline.searchindex.index_name].is_legacy
          if (isLegacy) {
            allIndices.push(timeline.searchindex.index_name)
          } else {
            allIndices.push(timeline.id)
          }
        })
        this.currentQueryFilter.indices = allIndices
      }
      let chips = this.currentQueryFilter.chips
      if (chips) {
        for (let i = 0; i < chips.length; i++) {
          if (chips[i].type === 'label') {
            this.selectedLabels.push(chips[i].value)
          }
        }
      }
      this.contextEvent = false
      this.search(true, true, node.id)
    },
    triggerScrollTo: function () {
      EventBus.$emit('triggerScrollTo')
    },
    zoomWithMouse: function (event) {
      // Add @wheel="zoomWithMouse" on element to activate.
      this.zoomOrigin.x = event.pageX
      this.zoomOrigin.y = event.pageY
      if (event.deltaY < 0) {
        this.zoomLevel += 0.07
      } else if (event.deltaY > 0) {
        this.zoomLevel -= 0.07
      }
    },
    closeSearchDropdown: function (targetElement) {
      // Prevent dropdown to close when the search input field is clicked.
      if (targetElement !== this.$refs.searchInput && targetElement.getAttribute('data-explore-element') === null) {
        this.showSearchDropdown = false
      }
    },
    onClickOutside: function (e) {
      if (e.target.id !== 'tsSearchInput') {
        this.showSearchDropdown = false
      }
    },
  },
  mounted() {
    this.$refs.searchInput.focus()
    EventBus.$on('setQueryAndFilter', this.setQueryAndFilter)
    EventBus.$on('setActiveView', this.searchView)
  },
  beforeDestroy() {
    EventBus.$off('setQueryAndFilter')
    EventBus.$off('setActiveView')
  },
  created: function () {
    let doSearch = false

    this.params = {
      viewId: this.$route.query.view,
      indexName: this.$route.query.timeline,
      resultLimit: this.$route.query.limit,
      queryString: this.$route.query.q,
    }

    if (this.params.viewId) {
      this.searchView(this.params.viewId)
      return
    }

    if (this.params.queryString) {
      this.currentQueryString = this.params.queryString
      doSearch = true
    }

    if (this.params.indexName) {
      if (!this.params.queryString) {
        this.currentQueryString = '*'
      }

      let timeline = this.sketch.active_timelines.find((timeline) => {
        return timeline.id === parseInt(this.params.indexName, 10)
      })

      let isLegacy = this.meta.indices_metadata[timeline.searchindex.index_name].is_legacy
      if (isLegacy) {
        this.currentQueryFilter.indices = [timeline.searchindex.index_name]
      } else {
        this.currentQueryFilter.indices = [timeline.id]
      }
      doSearch = true
    }

    if (!this.currentQueryString) {
      this.currentQueryFilter.indices = ['_all']
    }

    if (doSearch) {
      if (!this.currentQueryFilter.indices.length) {
        this.currentQueryFilter.indices = ['_all']
      }
      this.search()
    }
  },
}
</script>

<style lang="scss">
.chip-disabled {
  text-decoration: line-through;
  opacity: 0.5;
}

.chip-operator-label {
  margin-right: 7px;
  font-size: 0.7em;
  cursor: default;
}

.no-scrollbars::-webkit-scrollbar {
  display: none;
}

.no-scrollbars {
  -ms-overflow-style: none;
  scrollbar-width: none;
}
</style>
