<template>
  <div class="subcontent">
    <navigation-bar @today="onToday" @prev="onPrev" @next="onNext" />

    <div style="display: flex; justify-content: center; align-items: center; flex-wrap: nowrap;">
      <div style="font-size: 2em;">{{ formattedMonth }}</div>
    </div>

    <div class="row justify-center">
      <div style="display: flex; max-width: 800px; width: 100%;">

        <q-calendar-month ref="calendar" v-model="selectedDate" animated :day-min-height="100" bordered focusable
          :focus-type="['day']" :weekdays="[1, 2, 3, 4, 5]" hoverable use-navigation :locale="'pt-BR'"
          @change="onChange" @moved="onMoved" @click-date="onClickDate" @click-day="onClickDay"
          @click-workweek="onClickWorkweek" @click-head-workweek="onClickHeadWorkweek" @click-head-day="onClickHeadDay">

          <template #day="{ scope: { timestamp } }">
            <template v-for="event in eventsMap[timestamp.date]" :key="event.id">
              <div :class="badgeClasses(event, 'day')" :style="badgeStyles(event, 'day')" class="my-event">
                <div class="title q-calendar__ellipsis">
                  {{ event.title + (event.time ? ' - ' + event.time : '') }}
                  <q-tooltip>{{ event.details + ': ' + event.title + (event.time ? ' - ' + event.time : '')
                    }}</q-tooltip>
                </div>
              </div>
            </template>
          </template>

        </q-calendar-month>
      </div>
    </div>
  </div>
</template>

<script>
import {
  QCalendarMonth,
  PARSE_DATE, // regex for parsing out date
  addToDate,
  parseTimestamp,
  today,
  isBetweenDates
  // parseDate,
} from '@quasar/quasar-ui-qcalendar/src/index.js'
import '@quasar/quasar-ui-qcalendar/src/QCalendarVariables.sass'
import '@quasar/quasar-ui-qcalendar/src/QCalendarTransitions.sass'
import '@quasar/quasar-ui-qcalendar/src/QCalendarMonth.sass'

import {
  defineComponent,
  ref,
  reactive,

  computed
} from 'vue'
import NavigationBar from '../components/NavigationBar.vue'
import Holidays from 'date-holidays'

// const CURRENT_DAY = new Date()

// function getCurrentDay (day) {
//   const newDay = new Date(CURRENT_DAY)
//   newDay.setDate(day)
//   const tm = parseDate(newDay)
//   return tm.date
// }
export default defineComponent({
  name: 'MonthSlotDayHolidays',
  components: {
    NavigationBar,
    QCalendarMonth
  },
  setup (props, {
    slots,
    emit
  }) {
    const countries = new Holidays().getCountries()
    const selectedDate = ref(today()),
      selectedMonth = reactive([]),
      year = ref(new Date().getFullYear()),
      calendar = ref(null),
      country = ref('pt-BR')
    // var events = [{
    //   id: 1,
    //   title: '1st of the Month',
    //   details: 'Everything is funny as long as it is happening to someone else',
    //   date: getCurrentDay(1),
    //   bgcolor: 'orange'
    // },
    // {
    //   id: 2,
    //   title: 'Sisters Birthday',
    //   details: 'Buy a nice present',
    //   date: getCurrentDay(4),
    //   bgcolor: 'green',
    //   icon: 'fas fa-birthday-cake'
    // }]
    const formattedMonth = computed(() => {
      const date = new Date(selectedDate.value)
      return monthFormatter().format(date) + ' ' + date.getFullYear()
    })

    function monthFormatter () {
      try {
        return new Intl.DateTimeFormat('pt-BR' || undefined, {
          month: 'long',
          timeZone: 'UTC'
        })
      } catch (e) {
        //
      }
    }

    const holidaysMap = computed(() => {
      // keep previous, current and next year so in dec/jan
      // you can see holidays from different years
      return [
        ...(new Holidays(country.value).getHolidays(year.value - 1)),
        ...(new Holidays(country.value).getHolidays(year.value)),
        ...(new Holidays(country.value).getHolidays(year.value + 1))
      ]
    })

    function getColor (item) {
      switch (item.type) {
        case 'public':
          return 'blue'
        case 'observance':
          return 'green'
        case 'optional':
          return 'red'
        default: // bank|school
          return 'orange'
      }
    }

    /// where the magic happens...
    const eventsMap = computed(() => {
      const map = {}
      if (selectedMonth.length > 0) {
        const start = selectedMonth[0]
        const end = selectedMonth[selectedMonth.length - 1]
        holidaysMap.value
          .filter(item => {
            const timestamp = parseTimestamp((PARSE_DATE.exec(item.date))[0])
            return isBetweenDates(timestamp, start, end)
          })
          .map((item, index) => {
            return {
              id: index,
              title: item.name,
              details: item.type,
              date: (PARSE_DATE.exec(item.date))[0],
              bgcolor: getColor(item)
            }
          })
          .forEach(event => {
            (map[event.date] = (map[event.date] || [])).push(event)
            if (event.days !== undefined) {
              let timestamp = parseTimestamp(event.date)
              let days = event.days
              // add a new event for each day
              // skip 1st one which would have been done above
              do {
                timestamp = addToDate(timestamp, {
                  day: 1
                })
                if (!map[timestamp.date]) {
                  map[timestamp.date] = []
                }
                map[timestamp.date].push(event)
                // already accounted for 1st day
              } while (--days > 1)
            }
          })
      }

      return map
    })

    function badgeClasses (event, type) {
      return {
        [`text-white bg-${event.bgcolor}`]: true,
        'rounded-border': true
      }
    }

    function badgeStyles (day, event) {
      const s = {}
      // s.left = day.weekday === 0 ? 0 : (day.weekday * parsedCellWidth) + '%'
      // s.top = 0
      // s.bottom = 0
      // s.width = (event.days * parsedCellWidth) + '%'
      return s
    }

    function onToday () {
      calendar.value.moveToToday()
    }

    function onPrev () {
      calendar.value.prev()
    }

    function onNext () {
      calendar.value.next()
    }

    function onMoved (data) {
      console.log('onMoved', data)
    }

    function onChange (data) {
      console.log('onChange', data)
      selectedMonth.splice(0, selectedMonth.length, ...data.days)
      // get year of 1st of the month
      for (let index = 0; index < selectedMonth.length; ++index) {
        if (selectedMonth[index].day === 1) {
          year.value = selectedMonth[index].year
          break
        }
      }
    }

    function onClickDate (data) {
      console.log('onClickDate', data)
    }

    function onClickDay (data) {
      console.log('onClickDay', data)
    }

    function onClickWorkweek (data) {
      console.log('onClickWorkweek', data)
    }

    function onClickHeadDay (data) {
      console.log('onClickHeadDay', data)
    }

    function onClickHeadWorkweek (data) {
      console.log('onClickHeadWorkweek', data)
    }

    return {
      selectedDate,
      calendar,
      countries,
      country,
      eventsMap,
      formattedMonth,
      badgeClasses,
      badgeStyles,
      onToday,
      onPrev,
      onNext,
      onMoved,
      onChange,
      onClickDate,
      onClickDay,
      onClickWorkweek,
      onClickHeadDay,
      onClickHeadWorkweek
    }
  }
})
</script>

<style lang="sass" scoped>
.my-event
  position: relative
  font-size: 12px
  width: 100%
  margin: 1px 0 0 0
  justify-content: center
  text-overflow: ellipsis
  overflow: hidden
  cursor: pointer

.title
  position: relative
  display: flex
  justify-content: center
  align-items: center
  height: 100%

.text-white
  color: white

.bg-blue
  background: blue

.bg-green
  background: green

.bg-orange
  background: orange

.bg-red
  background: red

.bg-teal
  background: teal

.bg-grey
  background: grey

.bg-purple
  background: purple

.rounded-border
  border-radius: 2px
</style>