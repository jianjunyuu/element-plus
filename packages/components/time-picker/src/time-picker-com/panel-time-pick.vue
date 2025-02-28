<template>
  <transition :name="transitionName">
    <div v-if="actualVisible || visible" class="el-time-panel">
      <div
        class="el-time-panel__content"
        :class="{ 'has-seconds': showSeconds }"
      >
        <time-spinner
          ref="spinner"
          :role="datetimeRole || 'start'"
          :arrow-control="arrowControl"
          :show-seconds="showSeconds"
          :am-pm-mode="amPmMode"
          :spinner-date="parsedValue"
          :disabled-hours="disabledHours"
          :disabled-minutes="disabledMinutes"
          :disabled-seconds="disabledSeconds"
          @change="handleChange"
          @set-option="onSetOption"
          @select-range="setSelectionRange"
        />
      </div>
      <div class="el-time-panel__footer">
        <button
          type="button"
          class="el-time-panel__btn cancel"
          @click="handleCancel"
        >
          {{ t('el.datepicker.cancel') }}
        </button>
        <button
          type="button"
          class="el-time-panel__btn confirm"
          @click="handleConfirm()"
        >
          {{ t('el.datepicker.confirm') }}
        </button>
      </div>
    </div>
  </transition>
</template>

<script lang="ts">
import { defineComponent, ref, computed, inject } from 'vue'
import dayjs from 'dayjs'
import { EVENT_CODE } from '@element-plus/utils/aria'
import { useLocale } from '@element-plus/hooks'
import TimeSpinner from './basic-time-spinner.vue'
import { getAvailableArrs, useOldValue } from './useTimePicker'

import type { PropType } from 'vue'
import type { Dayjs } from 'dayjs'

export default defineComponent({
  components: {
    TimeSpinner,
  },

  props: {
    visible: Boolean,
    actualVisible: {
      type: Boolean,
      default: undefined,
    },
    datetimeRole: {
      type: String,
    },
    parsedValue: {
      type: [Object, String] as PropType<string | Dayjs>,
    },
    format: {
      type: String,
      default: '',
    },
  },

  emits: ['pick', 'select-range', 'set-picker-option'],

  setup(props, ctx) {
    const { t, lang } = useLocale()
    // data
    const selectionRange = ref([0, 2])
    const oldValue = useOldValue(props)
    // computed
    const transitionName = computed(() => {
      return props.actualVisible === undefined ? 'el-zoom-in-top' : ''
    })
    const showSeconds = computed(() => {
      return props.format.includes('ss')
    })
    const amPmMode = computed(() => {
      if (props.format.includes('A')) return 'A'
      if (props.format.includes('a')) return 'a'
      return ''
    })
    // method
    const isValidValue = (_date: Dayjs) => {
      const parsedDate = dayjs(_date).locale(lang.value)
      const result = getRangeAvailableTime(parsedDate)
      return parsedDate.isSame(result)
    }
    const handleCancel = () => {
      ctx.emit('pick', oldValue.value, false)
    }
    const handleConfirm = (visible = false, first = false) => {
      if (first) return
      ctx.emit('pick', props.parsedValue, visible)
    }
    const handleChange = (_date: Dayjs) => {
      // visible avoids edge cases, when use scrolls during panel closing animation
      if (!props.visible) {
        return
      }
      const result = getRangeAvailableTime(_date).millisecond(0)
      ctx.emit('pick', result, true)
    }

    const setSelectionRange = (start, end) => {
      ctx.emit('select-range', start, end)
      selectionRange.value = [start, end]
    }

    const changeSelectionRange = (step: number) => {
      const list = [0, 3].concat(showSeconds.value ? [6] : [])
      const mapping = ['hours', 'minutes'].concat(
        showSeconds.value ? ['seconds'] : []
      )
      const index = list.indexOf(selectionRange.value[0])
      const next = (index + step + list.length) % list.length
      timePickerOptions['start_emitSelectRange'](mapping[next])
    }

    const handleKeydown = (event: KeyboardEvent) => {
      const code = event.code

      if (code === EVENT_CODE.left || code === EVENT_CODE.right) {
        const step = code === EVENT_CODE.left ? -1 : 1
        changeSelectionRange(step)
        event.preventDefault()
        return
      }

      if (code === EVENT_CODE.up || code === EVENT_CODE.down) {
        const step = code === EVENT_CODE.up ? -1 : 1
        timePickerOptions['start_scrollDown'](step)
        event.preventDefault()
        return
      }
    }

    const getRangeAvailableTime = (date: Dayjs) => {
      const availableMap = {
        hour: getAvailableHours,
        minute: getAvailableMinutes,
        second: getAvailableSeconds,
      }
      let result = date
      ;['hour', 'minute', 'second'].forEach((_) => {
        if (availableMap[_]) {
          let availableArr
          const method = availableMap[_]
          if (_ === 'minute') {
            availableArr = method(result.hour(), props.datetimeRole)
          } else if (_ === 'second') {
            availableArr = method(
              result.hour(),
              result.minute(),
              props.datetimeRole
            )
          } else {
            availableArr = method(props.datetimeRole)
          }
          if (
            availableArr &&
            availableArr.length &&
            !availableArr.includes(result[_]())
          ) {
            result = result[_](availableArr[0])
          }
        }
      })
      return result
    }

    const parseUserInput = (value: Dayjs) => {
      if (!value) return null
      const unformattedDayjsValue = dayjs(value, props.format).locale(
        lang.value
      )

      function formatValue(value: string, format: string): string {
        if (value) {
          const strArr = value.split('')
          const formats = format.split(/[^a-z]+/i)
          let replacedValue = format
          formats.forEach((s) => {
            const temp = new Array(s.length).fill(0)
            const patch = strArr.splice(0, s.length)
            const patchLen = patch.length
            if (patchLen > 0) {
              temp.splice(temp.length - patchLen, patchLen, ...patch)
            }

            replacedValue = replacedValue.replace(RegExp(s), temp.join(''))
          })
          return replacedValue
        }
        return ''
      }

      // 没有效果，并且是纯数字的字符串则进行尝试解析
      return !unformattedDayjsValue.isValid() && !/[^\d]/g.test(value)
        ? dayjs(formatValue(value, props.format), props.format).locale(
            lang.value
          )
        : unformattedDayjsValue
    }

    const formatToString = (value: Dayjs) => {
      if (!value) return null
      return value.format(props.format)
    }

    const getDefaultValue = () => {
      return dayjs(defaultValue).locale(lang.value)
    }

    ctx.emit('set-picker-option', ['isValidValue', isValidValue])
    ctx.emit('set-picker-option', ['formatToString', formatToString])
    ctx.emit('set-picker-option', ['parseUserInput', parseUserInput])
    ctx.emit('set-picker-option', ['handleKeydown', handleKeydown])
    ctx.emit('set-picker-option', [
      'getRangeAvailableTime',
      getRangeAvailableTime,
    ])
    ctx.emit('set-picker-option', ['getDefaultValue', getDefaultValue])
    const timePickerOptions = {} as any
    const onSetOption = (e) => {
      timePickerOptions[e[0]] = e[1]
    }
    const pickerBase = inject('EP_PICKER_BASE') as any
    const {
      arrowControl,
      disabledHours,
      disabledMinutes,
      disabledSeconds,
      defaultValue,
    } = pickerBase.props
    const { getAvailableHours, getAvailableMinutes, getAvailableSeconds } =
      getAvailableArrs(disabledHours, disabledMinutes, disabledSeconds)

    return {
      transitionName,
      arrowControl,
      onSetOption,
      t,
      handleConfirm,
      handleChange,
      setSelectionRange,
      amPmMode,
      showSeconds,
      handleCancel,
      disabledHours,
      disabledMinutes,
      disabledSeconds,
    }
  },
})
</script>
