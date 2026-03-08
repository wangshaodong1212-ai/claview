<script setup lang="ts">
import { computed } from 'vue';

interface Props {
  data: {
    date: string;
    count: number;
  }[];
}

const props = defineProps<Props>();

// 周几标签
const weekDays = ['Mon', 'Wed', 'Fri'];

// 月份标签
const months = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'];

// 计算颜色等级
const getColorLevel = (count: number) => {
  if (count === 0) return 0;
  if (count <= 2) return 1;
  if (count <= 4) return 2;
  if (count <= 6) return 3;
  return 4;
};

// 格式化数据为 53 周 x 7 天的网格
const weeks = computed(() => {
  const result: (number | null)[][] = [];

  // 找到最早的日期
  if (props.data.length === 0) return result;

  // 初始化 53 周，每周 7 天
  for (let w = 0; w < 53; w++) {
    result[w] = Array(7).fill(null);
  }

  // 填充数据
  const dataMap = new Map(props.data.map(d => [d.date, d.count]));

  // 获取最早和最晚日期
  const dates = props.data.map(d => new Date(d.date));
  const minDate = new Date(Math.min(...dates.map(d => d.getTime())));
  const maxDate = new Date(Math.max(...dates.map(d => d.getTime())));

  // 从一年前的第一个周日开始
  const startDate = new Date(minDate);
  startDate.setDate(startDate.getDate() - startDate.getDay());

  // 填充网格
  let currentDate = new Date(startDate);
  const endDate = new Date(maxDate);
  endDate.setDate(endDate.getDate() + 6 - endDate.getDay());

  let weekIndex = 0;
  let dayIndex = 0;

  while (currentDate <= endDate && weekIndex < 53) {
    const dateKey = currentDate.toISOString().split('T')[0];
    const count = dataMap.get(dateKey ?? '') ?? 0;

    result[weekIndex]![dayIndex] = count;

    dayIndex++;
    if (dayIndex >= 7) {
      dayIndex = 0;
      weekIndex++;
    }

    currentDate.setDate(currentDate.getDate() + 1);
  }

  return result;
});

// 计算总计
const totalContributions = computed(() => {
  return props.data.reduce((sum, d) => sum + d.count, 0);
});

// 计算最高贡献
const maxContributions = computed(() => {
  return Math.max(...props.data.map(d => d.count), 0);
});
</script>

<template>
  <div class="contribution-graph">
    <div class="graph-header">
      <div class="graph-title">
        <span class="title-text">{{ totalContributions }}</span>
        <span class="title-label">contributions</span>
      </div>
      <div class="graph-legend">
        <span class="legend-label">Less</span>
        <div class="legend-levels">
          <span v-for="i in 5" :key="i" class="legend-level" :class="`level-${i - 1}`"></span>
        </div>
        <span class="legend-label">More</span>
      </div>
    </div>

    <div class="graph-container">
      <div class="week-labels">
        <span v-for="(day, i) in weekDays" :key="i" class="week-label" :style="{ top: `${i * 2 + 1}rem` }">
          {{ day }}
        </span>
      </div>

      <div class="graph-scroll">
        <div class="weeks">
          <div v-for="(week, w) in weeks" :key="w" class="week">
            <div v-for="(day, d) in week" :key="`${w}-${d}`"
              class="day-cell"
              :class="'level-' + (day === null ? 0 : getColorLevel(day))"
              :title="day === null ? 'No data' : `${day} contributions`"
            ></div>
          </div>
        </div>

        <div class="month-labels">
          <span v-for="m in 12" :key="m" class="month-label" :style="{ left: `${m * 3.5}rem` }">
            {{ months[m - 1] }}
          </span>
        </div>
      </div>
    </div>

    <div class="graph-footer">
      <span v-if="totalContributions === 0" class="empty-text">暂无贡献数据</span>
      <span v-else class="stats-text">
        最高单日: {{ maxContributions }} 条对话
      </span>
    </div>
  </div>
</template>

<style scoped>
.contribution-graph {
  padding: 20px;
  background: var(--bg-secondary);
  border-radius: 12px;
  border: 1px solid var(--border-color);
}

.graph-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 16px;
}

.graph-title {
  display: flex;
  align-items: baseline;
  gap: 8px;
}

.title-text {
  font-size: 28px;
  font-weight: 700;
  color: var(--text-primary);
}

.title-label {
  font-size: 14px;
  color: var(--text-muted);
}

.graph-legend {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 11px;
  color: var(--text-muted);
}

.legend-levels {
  display: flex;
  gap: 3px;
}

.legend-level {
  width: 12px;
  height: 12px;
  border-radius: 2px;
  background: var(--border-color);
}

.legend-level.level-1 {
  background: rgba(88, 166, 255, 0.2);
}

.legend-level.level-2 {
  background: rgba(88, 166, 255, 0.4);
}

.legend-level.level-3 {
  background: rgba(88, 166, 255, 0.6);
}

.legend-level.level-4 {
  background: rgba(88, 166, 255, 0.8);
}

.graph-container {
  position: relative;
  display: flex;
  gap: 8px;
}

.week-labels {
  position: absolute;
  left: 0;
  top: 20px;
  bottom: 24px;
  width: 30px;
}

.week-label {
  position: absolute;
  font-size: 10px;
  color: var(--text-muted);
  transform: translateX(-100%);
}

.graph-scroll {
  flex: 1;
  overflow-x: auto;
}

.weeks {
  display: flex;
  gap: 3px;
}

.week {
  display: flex;
  flex-direction: column;
  gap: 3px;
}

.day-cell {
  width: 12px;
  height: 12px;
  border-radius: 2px;
  background: var(--border-color);
  transition: all 0.15s ease;
}

.day-cell:hover {
  transform: scale(1.3);
  outline: 2px solid var(--accent-color);
  outline-offset: 1px;
}

.day-cell.level-1 {
  background: rgba(88, 166, 255, 0.2);
}

.day-cell.level-2 {
  background: rgba(88, 166, 255, 0.4);
}

.day-cell.level-3 {
  background: rgba(88, 166, 255, 0.6);
}

.day-cell.level-4 {
  background: rgba(88, 166, 255, 0.8);
}

.month-labels {
  position: relative;
  height: 24px;
  margin-top: 8px;
}

.month-label {
  position: absolute;
  font-size: 10px;
  color: var(--text-muted);
}

.graph-footer {
  margin-top: 16px;
  text-align: center;
}

.empty-text {
  color: var(--text-muted);
  font-size: 13px;
}

.stats-text {
  color: var(--text-secondary);
  font-size: 12px;
}
</style>
