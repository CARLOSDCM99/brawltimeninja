<template>
  <card v-bind="$attrs">
    <template v-slot:content>
      <div class="flex md:hidden">
        <b-button
          :selected="showFilters"
          primary
          sm
          class="mr-3 h-8"
          @click="showFilters = !showFilters"
        >
          <font-awesome-icon
            :icon="faFilter"
          ></font-awesome-icon>
        </b-button>

        <span
          :class="['my-auto leading-tight ', {
            'whitespace-nowrap overflow-hidden overflow-ellipsis': !showFilters,
          }]"
          @click="showFilters = true"
        >
          {{ filtersDescription }}
        </span>
      </div>

      <div
        :class="['md:flex flex-wrap', {
          'hidden': !showFilters,
          'mt-3': showFilters,
        }]"
      >
        <div
          v-if="cubes != undefined"
          class="mr-2 my-1"
        >
          <b-select
            v-model="selectedCube"
            dark
            sm
          >
            <option
              v-for="c in cubes"
              :key="c"
              :value="c"
            >
              {{ cubeLabel[c] }}
            </option>
          </b-select>
        </div>

        <div
          v-if="['synergy'].includes(cube)"
          class="mr-2 my-1"
        >
          <b-select
            v-model="ally"
            dark
            sm
          >
            <option
              v-for="b in brawlers"
              :key="b"
              :value="b"
            >with {{ capitalize(b.toLowerCase()) }}</option>
          </b-select>
        </div>

        <div class="mr-2 my-1">
          <b-select
            :value="measurement"
            dark
            sm
            @input="m => $emit('measurement', m)"
          >
            <option
              v-for="m in measurements"
              :key="m"
              :value="m"
            >
              {{ metaStatMaps.labels[m] }}
            </option>
          </b-select>
        </div>

        <div class="mr-2 my-1">
          <b-select
            v-model="timeRange"
            dark
            sm
          >
            <option
              v-for="(label, t) in timeRangeLabel"
              :key="t"
              :value="t"
            >
              Current {{ label }}
            </option>
          </b-select>
        </div>

        <div
          class="mr-2 my-1"
          v-if="['battle'].includes(cube)"
        >
          <input
            type="text"
            v-model.lazy="nameFilter"
            class="rounded font-semibold text-sm py-1 pl-2 border-2 form-input bg-gray-700 hover:bg-gray-500 border-gray-500 hover:border-yellow-400 text-gray-200"
          >
        </div>

        <div
          class="mr-2 my-1"
          v-if="['map', 'battle'].includes(cube)"
        >
          <b-select
            v-model="powerPlayActive"
            dark
            sm
          >
            <option value="false">Regular Battles</option>
            <option value="true">Power Play</option>
          </b-select>
        </div>

        <trophy-slider-select
          v-if="['map', 'starpower', 'gadget', 'battle'].includes(cube)"
          v-model="trophyRange"
          :name="powerPlayActive == 'true' ? 'Points' : undefined"
          class="mr-2 my-1"
        ></trophy-slider-select>

        <!-- TODO add icons and previews to selects -->
        <div
          v-if="['map', 'synergy', 'team', 'battle'].includes(cube)"
          class="mr-2 my-1"
        >
          <b-select
            v-model="mode"
            dark
            sm
          >
            <option value="">All Modes</option>
            <option
              v-for="mode in modes"
              :key="mode"
              :value="mode"
            >{{ formatMode(mode) }}</option>
          </b-select>
        </div>

        <div
          v-if="['map', 'synergy', 'team', 'battle'].includes(cube)"
          v-show="mode != ''"
          class="mr-2 my-1"
        >
          <b-select
            v-model="map"
            dark
            sm
          >
            <option value="">All Maps</option>
            <option
              v-for="map in maps"
              :key="map"
              :value="map"
            >{{ map }}</option>
          </b-select>
        </div>

        <client-only>
          <b-button
            v-if="supportsShareApi"
            secondary
            sm
            class="ml-auto my-auto h-8"
            @click="share()"
          >
            <font-awesome-icon
              :icon="faShare"
            ></font-awesome-icon>
          </b-button>
        </client-only>
      </div>
    </template>
  </card>
</template>

<script lang="ts">
import Vue, { PropType } from 'vue'
import { capitalize, formatMode, metaStatMaps } from '~/lib/util'
import { faFilter, faShare } from '@fortawesome/free-solid-svg-icons'

// TODO add big brawler

interface Slices {
  trophy_season_end: string[]
  brawler_trophyrange: string[]
  battle_event_powerplay?: string[]
  battle_event_mode?: string[]
  battle_event_map?: string[]
  ally_brawler_name?: string[]
  player_name_ilike?: string[]
}

function withN1Slice(slices: Slices, name: keyof Slices, value: string) {
  const newSlices = { ...slices }
  if (value == '') {
    delete newSlices[name]
  } else {
    newSlices[name] = [value]
  }
  return newSlices
}

export default Vue.extend({
  inheritAttrs: false,
  props: {
    value: {
      type: Object as PropType<Slices>,
      required: true
    },
    measurement: {
      type: String,
      required: true
    },
    measurements: {
      type: Array as PropType<string[]>,
      required: true
    },
    cube: {
      type: String,
      required: true
    },
    cubes: {
      type: Array as PropType<string[]>,
    },
  },
  data() {
    return {
      allModes: [] as string[],
      allMaps: [] as string[],
      allBrawlers: [] as string[],
      timeRangeLabel: {
        'current': 'Season',
        'balance': 'Update',
        'month': 'Month',
      },
      cubeLabel: {
        map: 'Brawlers',
        gadget: 'Gadgets',
        starpower: 'Star Powers',
        synergy: 'Synergies',
        team: 'Teams',
        battle: 'Players',
      },
      showFilters: false,
    }
  },
  watch: {
    cube: '$fetch',
    mode: '$fetch',
  },
  fetchDelay: 0,
  fetchOnServer: false, // FIXME: causes render error
  async fetch() {
    if (['map', 'synergy', 'team', 'battle'].includes(this.cube)) {
      this.allModes = await this.$clicker.queryAllModes()
      const maps = await this.$clicker.queryAllMaps(this.mode == '' ? undefined : this.mode)
      this.allMaps = maps.sort((m1, m2) => m1.localeCompare(m2))
    }
    if (this.cube == 'synergy') {
      const brawlers = await this.$clicker.queryAllBrawlers()
      this.allBrawlers = brawlers.sort((b1, b2) => b1.localeCompare(b2))
    }
  },
  computed: {
    modes(): string[] {
      if (this.allModes.length == 0 && !['', undefined].includes(this.mode)) {
        return [this.mode]
      }
      return this.allModes
    },
    maps(): string[] {
      if (this.allMaps.length == 0 && !['', undefined].includes(this.map)) {
        return [this.map]
      }
      return this.allMaps
    },
    brawlers(): string[] {
      if (this.allBrawlers.length == 0 && !['', undefined].includes(this.ally)) {
        return [this.ally]
      }
      return this.allBrawlers
    },
    selectedCube: {
      get(): string {
        return this.cube
      },
      set(c: string) {
        this.$emit('cube', c)
      }
    },
    trophyRange: {
      get(): number[] {
        return (this.value.brawler_trophyrange || ['0', '10']).map(n => parseInt(n))
      },
      set(v: number[]) {
        this.$emit('input', {
          ...this.value,
          brawler_trophyrange: v.map(n => n.toString()),
        })
      }
    },
    timeRange: {
      get(): string {
        return this.value.trophy_season_end[0] || ''
      },
      set(v: string) {
        this.$emit('input', withN1Slice(this.value, 'trophy_season_end', v))
      }
    },
    powerPlayActive: {
      get(): string {
        return (this.value.battle_event_powerplay || [])[0] || 'false'
      },
      set(v: string) {
        this.$emit('input', withN1Slice(this.value, 'battle_event_powerplay', v))
      }
    },
    mode: {
      get(): string {
        return (this.value.battle_event_mode || [])[0] || ''
      },
      set(v: string) {
        // reset map
        this.$emit('input', withN1Slice(
          withN1Slice(this.value, 'battle_event_mode', v),
          'battle_event_map', ''
        ))
      }
    },
    map: {
      get(): string {
        return (this.value.battle_event_map || [])[0] || ''
      },
      set(v: string) {
        this.$emit('input', withN1Slice(this.value, 'battle_event_map', v))
      }
    },
    ally: {
      get(): string {
        return (this.value.ally_brawler_name || [])[0] || 'SHELLY'
      },
      set(v: string) {
        this.$emit('input', withN1Slice(this.value, 'ally_brawler_name', v))
      }
    },
    nameFilter: {
      get(): string {
        return (this.value.player_name_ilike || [])[0] || '%'
      },
      set(v: string) {
        this.$emit('input', withN1Slice(this.value, 'player_name_ilike', v))
      }
    },
    filtersDescription(): string {
      const formatTrophies = (n: number) => n == 10 ? '1000+' : n * 100
      const pieces = [
        this.cubeLabel[this.cube],
        ...(['synergy'].includes(this.cube) && this.ally != '' ? ['with ' + capitalize(this.ally.toLowerCase())] : []),
        metaStatMaps.labels[this.measurement],
        this.timeRange in this.$clicker.timePresets ? 'Current ' + this.$clicker.timePresets[this.timeRange] : 'Since ' + this.timeRange,
        this.powerPlayActive == 'false' ? 'Regular Battles' : 'Power Play',
        formatTrophies(this.trophyRange[0]) + '-' + formatTrophies(this.trophyRange[1]) + (this.powerPlayActive == 'false' ? ' Trophies' : ' Points'),
        this.mode == '' ? 'All Modes' : formatMode(this.mode),
        this.map == '' ? 'All Maps' : this.map,
      ]
      return pieces.join(', ')
    },
    supportsShareApi() {
      return global.window && 'share' in window.navigator
    },
    faFilter() {
      return faFilter
    },
    faShare() {
      return faShare
    },
    metaStatMaps() {
      return metaStatMaps
    },
    formatMode() {
      return formatMode
    },
    capitalize() {
      return capitalize
    },
  },
  methods: {
    async share() {
      // https://developer.mozilla.org/en-US/docs/Web/API/Navigator/share
      try {
        const { query } = this.$clicker.slicesToLocation(this.value as any, this.$clicker.defaultSlices(this.cube))
        const params = new URLSearchParams()
        params.append('cube', this.cube)
        Object.entries(query as Record<string, string[]>)
          .forEach(([name, values]) => values.forEach(value => params.append(name, value)))

        await navigator.share({
          url: 'https://brawltime.ninja/dashboard?' + params.toString(),
        })
        this.$gtag.event('click', {
          'event_category': 'dashboard',
          'event_label': 'share_filters',
        })
      } catch (err) {
        console.error(err);
        this.$gtag.event('click', {
          'event_category': 'dashboard',
          'event_label': 'share_filters_error',
        })
      }
    },
  },
})
</script>
