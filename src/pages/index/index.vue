<template>
  <view class="index">
    <!-- 自绘日历卡片，日期单元格下方显示农历 -->
    <view class="calendar-card">
      <view class="calendar-header">
        <text class="calendar-arrow" @tap="goPrevMonth">‹</text>
        <text class="calendar-month" @tap="toggleYearPicker">
          {{ currentYear }}年{{ currentMonth }}月
        </text>
        <text class="calendar-arrow" @tap="goNextMonth">›</text>
      </view>
      <view class="calendar-week-row">
        <text v-for="w in weekLabels" :key="w" class="week-label">{{ w }}</text>
      </view>
      <view class="calendar-grid">
        <view
          v-for="day in calendarDays"
          :key="day.key"
          class="calendar-cell"
          :class="{
            'is-placeholder': day.isPlaceholder,
            'is-today': day.isToday,
            'is-selected': day.dateStr === currentDate,
            'is-other-month': !day.isCurrentMonth
          }"
          @tap="handleSelectDay(day)"
        >
          <template v-if="!day.isPlaceholder">
            <text class="cell-solar">{{ day.day }}</text>
            <text class="cell-lunar">{{ day.lunarText }}</text>
          </template>
        </view>
      </view>

      <!-- 年份选择面板 -->
      <view v-if="showYearPicker" class="year-picker-mask" @tap="toggleYearPicker">
        <view class="year-picker" @tap.stop>
          <view class="year-picker-title">选择年份</view>
          <view class="year-grid">
            <text
              v-for="y in yearOptions"
              :key="y"
              class="year-item"
              :class="{ 'year-item-active': y === currentYear }"
              @tap="selectYear(y)"
            >
              {{ y }}年
            </text>
          </view>
        </view>
      </view>
    </view>

    <!-- 选中日期卡片 + 万年历吉凶分析 -->

    <view class="fortune-card">
      <view class="fortune-header">
        <text>{{ currentDate }} · 农历{{ lunarInfo.lunarMonthName }} · {{ lunarInfo.lunarText }}</text>
      </view>
      <view class="fortune-section">
        <text class="fortune-title">宜：</text>
        <text class="fortune-body">
          {{ lunarInfo.goodList.length ? lunarInfo.goodList.join('、') : '暂无适宜事项' }}
        </text>
      </view>
      <view class="fortune-section">
        <text class="fortune-title">忌：</text>
        <text class="fortune-body">
          {{ lunarInfo.badList.length ? lunarInfo.badList.join('、') : '暂无禁忌事项' }}
        </text>
      </view>
    </view>
  </view>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { SolarDay } from 'tyme4ts'

const today = new Date()

const formatDate = (d: Date) => {
  const y = d.getFullYear()
  const m = `${d.getMonth() + 1}`.padStart(2, '0')
  const day = `${d.getDate()}`.padStart(2, '0')
  return `${y}-${m}-${day}`
}

// 当前选中日期（YYYY-MM-DD）
const currentDate = ref(formatDate(today))

// 当前日历所在的年、月
const currentYear = ref(today.getFullYear())
const currentMonth = ref(today.getMonth() + 1)

// 年份选择
const showYearPicker = ref(false)
const yearRange = 10
const yearOptions = computed(() => {
  const center = currentYear.value
  const start = center - yearRange
  const end = center + yearRange
  const list: number[] = []
  for (let y = start; y <= end; y++) {
    list.push(y)
  }
  return list
})

const toggleYearPicker = () => {
  showYearPicker.value = !showYearPicker.value
}

const selectYear = (year: number) => {
  currentYear.value = year
  // 选中日期如果不在当前月，就重置到当月1号
  const newDate = new Date(year, currentMonth.value - 1, 1)
  currentDate.value = formatDate(newDate)
  showYearPicker.value = false
}

const goPrevMonth = () => {
  let year = currentYear.value
  let month = currentMonth.value - 1
  if (month < 1) {
    month = 12
    year -= 1
  }
  currentYear.value = year
  currentMonth.value = month
  currentDate.value = formatDate(new Date(year, month - 1, 1))
}

const goNextMonth = () => {
  let year = currentYear.value
  let month = currentMonth.value + 1
  if (month > 12) {
    month = 1
    year += 1
  }
  currentYear.value = year
  currentMonth.value = month
  currentDate.value = formatDate(new Date(year, month - 1, 1))
}

const weekLabels = ['一', '二', '三', '四', '五', '六', '日']

// 当月日历网格（含前后补位）
const calendarDays = computed(() => {
  const year = currentYear.value
  const month = currentMonth.value
  const firstDay = new Date(year, month - 1, 1)
  const firstWeekday = firstDay.getDay() || 7 // 周日=0 转成 7
  const daysInMonth = new Date(year, month, 0).getDate()

  const days: {
    key: string
    day: number
    dateStr: string
    isToday: boolean
    isCurrentMonth: boolean
    isPlaceholder: boolean
    lunarText: string
  }[] = []

  // 前置占位
  for (let i = 1; i < firstWeekday; i++) {
    days.push({
      key: `p-${i}`,
      day: 0,
      dateStr: '',
      isToday: false,
      isCurrentMonth: false,
      isPlaceholder: true,
      lunarText: ''
    })
  }

  // 当月日期
  for (let d = 1; d <= daysInMonth; d++) {
    const dateObj = new Date(year, month - 1, d)
    const dateStr = formatDate(dateObj)
    const info = buildLunarInfo(dateStr)

    days.push({
      key: `d-${dateStr}`,
      day: d,
      dateStr,
      isToday: dateStr === formatDate(today),
      isCurrentMonth: true,
      isPlaceholder: false,
      lunarText: info.lunarText
    })
  }

  // 末尾补齐到整周
  const remain = days.length % 7
  if (remain !== 0) {
    const need = 7 - remain
    for (let i = 0; i < need; i++) {
      days.push({
        key: `s-${i}`,
        day: 0,
        dateStr: '',
        isToday: false,
        isCurrentMonth: false,
        isPlaceholder: true,
        lunarText: ''
      })
    }
  }

  return days
})

const buildLunarInfo = (dateStr: string) => {
  const [y, m, d] = dateStr.split('-').map((v) => Number(v))
  const solarDay = SolarDay.fromYmd(y, m, d)
  const lunar = solarDay.getLunarDay()
  const lunarMonth = lunar.getLunarMonth()
  const lunarYear = lunarMonth.getLunarYear()
  const ganzhi = lunarYear.getSixtyCycle()

  const goodList = lunar.getRecommends().map((t) => t.getName())
  const badList = lunar.getAvoids().map((t) => t.getName())

  return {
    lunarText: lunar.getName(), // 例如「初一」
    lunarMonthName: lunarMonth.getName(), // 例如「正月」
    ganzhiYear: ganzhi.getName(), // 例如「甲子」
    zodiac: ganzhi.getEarthBranch().getZodiac().getName(), // 例如「鼠」
    term: '', // 节气后面再单独补
    goodList,
    badList
  }
}

const lunarInfo = computed(() => buildLunarInfo(currentDate.value))

// 选中某一天
const handleSelectDay = (day: { dateStr: string; isPlaceholder: boolean }) => {
  if (day.isPlaceholder || !day.dateStr) return
  currentDate.value = day.dateStr
}
</script>

<style lang="less">
.index {
  padding: 16px;
  background: #f5f6fa;
  min-height: 100vh;
  box-sizing: border-box;
}

.calendar-card {
  padding: 20px 16px;
  border-radius: 16px;
  background: #ffffff;
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.05);
}

.calendar-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.calendar-month {
  font-size: 32px;
  font-weight: 600;
  color: #333;
}

.calendar-arrow {
  font-size: 48px;
  color: #999;
  padding: 0 8px;
}

.calendar-week-row {
  display: flex;
  margin-bottom: 4px;
  margin-top: 16px;
}

.week-label {
  flex: 1;
  text-align: center;
  font-size: 24px;
  color: #999;
}

.year-picker-mask {
  position: fixed;
  left: 0;
  top: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.35);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.year-picker {
  width: 80%;
  max-height: 70%;
  background: #fff;
  border-radius: 16px;
  padding: 16px;
  box-sizing: border-box;
}

.year-picker-title {
  text-align: center;
  font-size: 20px;
  font-weight: 600;
  margin-bottom: 12px;
}

.year-grid {
  display: flex;
  flex-wrap: wrap;
}

.year-item {
  width: 33.3333%;
  text-align: center;
  padding: 12px 0;
  font-size: 24px;
  color: #666;
}

.year-item-active {
  color: #d48806;
  font-weight: 600;
}

.calendar-grid {
  display: flex;
  flex-wrap: wrap;
}

.calendar-cell {
  width: 14.2857%;
  padding: 14px 0;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  border-radius: 12px;
}

.calendar-cell.is-placeholder {
  opacity: 0;
}

.calendar-cell.is-other-month {
  opacity: 0.4;
}

.calendar-cell.is-today {
  border: 1px solid #ff9f0a;
}

.calendar-cell.is-selected {
  background: #ffecb3;
}

.cell-solar {
  font-size: 24px;
  font-weight: 500;
  color: #333;
}

.cell-lunar {
  margin-top: 2px;
  font-size: 18px;
  color: #999;
}

.lunar-info {
  margin-top: 16px;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  padding: 8px 4px;
}

.lunar-main text {
  font-size: 20px;
  font-weight: 600;
  color: #333;
}

.lunar-sub text {
  margin-top: 4px;
  font-size: 12px;
  color: #888;
}

.fortune-card {
  margin-top: 24px;
  padding: 16px;
  border-radius: 16px;
  background: linear-gradient(135deg, #fff7e6, #ffe7ba);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.08);
}

.fortune-header text {
  font-size: 32px;
  font-weight: 600;
  color: #8c6c3f;
}

.fortune-section {
  margin-top: 8px;
  display: flex;
  align-items: flex-start;
}

.fortune-title {
  font-size: 24px;
  font-weight: 600;
  color: #d48806;
  margin-right: 4px;
}

.fortune-body {
  flex: 1;
  font-size: 22px;
  line-height: 1.5;
  color: #614700;
}
</style>
