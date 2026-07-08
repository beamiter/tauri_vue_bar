<template>
  <!-- 加载态 -->
  <div v-if="!monitorSnapshot" class="button-row">Loading...</div>

  <!-- 主界面 -->
  <div v-else class="button-row">
    <div class="buttons-container">
      <!-- Tag Buttons -->
      <button
        v-for="(icon, i) in TAG_ICONS"
        :key="i"
        :class="buttonClass(i)"
        @mousedown="pressedButton = i"
        @mouseup="onTagRelease(i)"
        @mouseleave="pressedButton = null"
        :title="`Tag ${i + 1}`"
      >
        <span class="nf-icon">{{ icon }}</span>
      </button>

      <!-- 布局切换 -->
      <div class="layout-controls">
        <div
          :class="['pill', 'layout-toggle', layoutOpen ? 'open' : 'closed']"
          @click="layoutOpen = !layoutOpen"
          title="切换布局"
        >
          {{ currentSymbol }}
        </div>
        <div v-if="layoutOpen" class="layout-selector">
          <div
            :class="['pill', 'layout-option', currentSymbol === '[]=' ? 'current' : '']"
            @click="onLayoutSelect(0)"
          >
            []=
          </div>
          <div
            :class="['pill', 'layout-option', currentSymbol === '><>' ? 'current' : '']"
            @click="onLayoutSelect(1)"
          >
            &gt;&lt;&gt;
          </div>
          <div
            :class="['pill', 'layout-option', currentSymbol === '[M]' ? 'current' : '']"
            @click="onLayoutSelect(2)"
          >
            [M]
          </div>
        </div>
      </div>
    </div>

    <div class="spacer"></div>

    <div class="right-info-container">
      <!-- 系统信息 -->
      <template v-if="systemSnapshot">
        <div class="system-info-container">
          <div class="pill usage-pill" :class="cpuClass" title="CPU 平均使用率">
            <span class="nf-icon">{{ ICON_CPU }}</span>
            {{ Math.round(systemSnapshot.cpu_average) }}%
          </div>
          <div
            class="pill usage-pill"
            :class="memClass"
            :title="`内存使用: ${formatBytes(systemSnapshot.memory_used)} / ${formatBytes(systemSnapshot.memory_total)}`"
          >
            <span class="nf-icon">{{ ICON_MEM }}</span>
            {{ Math.round(systemSnapshot.memory_usage_percent) }}%
          </div>
          <div
            class="pill usage-pill"
            :class="battClass"
            :title="systemSnapshot.is_charging
              ? `电池充电中: ${systemSnapshot.battery_percent.toFixed(1)}%`
              : `电池电量: ${systemSnapshot.battery_percent.toFixed(1)}%`"
          >
            <span class="nf-icon">{{ systemSnapshot.is_charging ? ICON_BAT_CHG : ICON_BAT_FULL }}</span>
            {{ Math.round(systemSnapshot.battery_percent) }}%
          </div>
        </div>
      </template>
      <template v-else>
        <div class="system-info-container">
          <div class="pill usage-pill usage-warn">
            <span class="nf-icon">{{ ICON_CPU }}</span> --%
          </div>
          <div class="pill usage-pill usage-warn">
            <span class="nf-icon">{{ ICON_MEM }}</span> --%
          </div>
          <div class="pill usage-pill usage-warn">
            <span class="nf-icon">{{ ICON_BAT_FULL }}</span> --%
          </div>
        </div>
      </template>

      <!-- 亮度 -->
      <div
        class="pill brightness-pill"
        @click="onBrightnessClick"
        @wheel.prevent="onBrightnessWheel"
        @contextmenu.prevent="onBrightnessRight"
        title="左键加亮 / 右键减暗 / 滚轮调节"
      >
        <span class="nf-icon">{{ ICON_BRIGHT }}</span> {{ brightnessLabel }}
      </div>

      <!-- 音量 -->
      <div
        :class="['pill', 'volume-pill', volumeMuted ? 'muted' : '']"
        @click="onToggleMute"
        @wheel.prevent="onVolumeWheel"
        title="左键静音 / 滚轮调节"
      >
        <span class="nf-icon">{{ volumeIconChar }}</span> {{ volumeLabel }}
      </div>

      <!-- 截图按钮 -->
      <div
        class="pill screenshot-pill"
        :class="{ taking: isTaking }"
        @click="onScreenshot"
        title="截图 (Flameshot)"
      >
        <span class="nf-icon">{{ ICON_SHOT }}</span>
      </div>

      <!-- 时间 -->
      <div
        class="pill time-pill"
        @click="showSeconds = !showSeconds"
        title="点击切换秒显示"
      >
        <span class="nf-icon">{{ ICON_TIME }}</span> {{ formattedTime }}
      </div>

      <!-- 显示器/缩放 -->
      <div class="pill monitor-pill" title="显示器">
        <span class="nf-icon">{{ ICON_MON }}</span> {{ monitorIcon(monitorNum) }}
      </div>
      <div class="pill scale-pill" title="Scale Factor">
        s: {{ scaleText }}
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount, watch } from 'vue';
import { listen, type UnlistenFn } from '@tauri-apps/api/event';
import { invoke } from '@tauri-apps/api/core';

// --- 类型定义，与后端 Rust 结构体对应 ---
interface TagStatus {
  is_selected: boolean;
  is_urg: boolean;
  is_filled: boolean;
  is_occ: boolean;
}

interface MonitorInfoSnapshot {
  monitor_num: number;
  monitor_width: number;
  monitor_height: number;
  monitor_x: number;
  monitor_y: number;
  tag_status_vec: TagStatus[];
  client_name: string;
  ltsymbol: string;
}

interface SystemSnapshot {
  cpu_average: number;
  memory_used: number;
  memory_total: number;
  memory_usage_percent: number;
  battery_percent: number;
  is_charging: boolean;
}

interface AudioSnapshot {
  volume: number;
  is_muted: boolean;
  device_name: string;
  has_device: boolean;
}

interface BrightnessSnapshot {
  percent: number | null;
}

// --- Nerd Font 图标 ---
const TAG_ICONS = [
  '\u{F0A1E}', // terminal
  '\u{F0239}', // firefox
  '\u{F0A1B}', // code
  '\u{F0B79}', // chat
  '\u{F024B}', // folder
  '\u{F0388}', // music
  '\u{F0567}', // video
  '\u{F01F0}', // mail
  '\u{F0297}', // gamepad
];

const ICON_CPU = '\u{F4BC}';
const ICON_MEM = '\u{F035B}';
const ICON_BAT_FULL = '\u{F0079}';
const ICON_BAT_CHG = '\u{F0084}';
const ICON_VOL_HIGH = '\u{F057E}';
const ICON_VOL_MID = '\u{F0580}';
const ICON_VOL_LOW = '\u{F057F}';
const ICON_VOL_MUTE = '\u{F075F}';
const ICON_BRIGHT = '\u{F00DE}';
const ICON_SHOT = '\u{F0104}';
const ICON_TIME = '\u{F0954}';
const ICON_MON = '\u{F0379}';

// --- 帮助函数 ---
const getButtonClass = (tagStatus: TagStatus): string => {
  if (tagStatus.is_filled) return 'emoji-button state-filtered';
  if (tagStatus.is_selected) return 'emoji-button state-selected';
  if (tagStatus.is_urg) return 'emoji-button state-urgent';
  if (tagStatus.is_occ) return 'emoji-button state-occupied';
  return 'emoji-button state-default';
};

const formatBytes = (bytes: number): string => {
  if (bytes === 0) return '0B';
  const UNITS = ['B', 'KB', 'MB', 'GB', 'TB'];
  const i = Math.floor(Math.log(bytes) / Math.log(1024));
  const size = parseFloat((bytes / Math.pow(1024, i)).toFixed(i === 0 ? 0 : 1));
  return `${size}${UNITS[i]}`;
};

function parseLtSymbol(lts?: string) {
  if (!lts) return { symbol: '[]=', scale: undefined as number | undefined };
  const symbolMatch = lts.match(/^(\S+)/);
  const scaleMatch = lts.match(/s:\s*([0-9.]+)/i);
  const symbol = symbolMatch ? symbolMatch[1] : '[]=';
  const scale = scaleMatch ? parseFloat(scaleMatch[1]) : undefined;
  return { symbol, scale };
}

function monitorIcon(num: number) {
  if (num === 0) return '\u{F02DA}';
  if (num === 1) return '\u{F02DB}';
  return `M${num}`;
}

// --- 响应式状态 ---
const monitorSnapshot = ref<MonitorInfoSnapshot | null>(null);
const systemSnapshot = ref<SystemSnapshot | null>(null);
const audioSnapshot = ref<AudioSnapshot | null>(null);
const brightnessSnapshot = ref<BrightnessSnapshot | null>(null);

const pressedButton = ref<number | null>(null);
const layoutOpen = ref(false);
const isTaking = ref(false);

const showSeconds = ref(true);
const now = ref(new Date());
let timer: number | undefined;

// --- 事件监听（Tauri） ---
onMounted(() => {
  console.log('Tauri Vue frontend has loaded.');
  let unlistenMon: UnlistenFn | null = null;
  let unlistenSys: UnlistenFn | null = null;
  let unlistenAud: UnlistenFn | null = null;
  let unlistenBri: UnlistenFn | null = null;

  (async () => {
    try {
      unlistenMon = await listen<MonitorInfoSnapshot>('monitor-update', (event) => {
        monitorSnapshot.value = event.payload;
      });
      unlistenSys = await listen<SystemSnapshot>('system-update', (event) => {
        systemSnapshot.value = event.payload;
      });
      unlistenAud = await listen<AudioSnapshot>('audio-update', (event) => {
        audioSnapshot.value = event.payload;
      });
      unlistenBri = await listen<BrightnessSnapshot>('brightness-update', (event) => {
        brightnessSnapshot.value = event.payload;
      });
    } catch (e) {
      console.error('Failed to register Tauri event listeners:', e);
    }
  })();

  startTimer();

  onBeforeUnmount(() => {
    if (unlistenMon) unlistenMon();
    if (unlistenSys) unlistenSys();
    if (unlistenAud) unlistenAud();
    if (unlistenBri) unlistenBri();
    if (timer) clearInterval(timer);
  });
});

watch(showSeconds, () => startTimer());

function startTimer() {
  if (timer) clearInterval(timer);
  timer = window.setInterval(() => {
    now.value = new Date();
  }, showSeconds.value ? 1000 : 60000);
}

// --- 计算属性 ---
const monitorNum = computed(() => monitorSnapshot.value?.monitor_num ?? 0);

const currentSymbol = computed(() => {
  const lts = monitorSnapshot.value?.ltsymbol;
  return parseLtSymbol(lts).symbol;
});

const scaleText = computed(() => {
  const lts = monitorSnapshot.value?.ltsymbol;
  const { scale } = parseLtSymbol(lts);
  return scale !== undefined ? scale.toFixed(2) : '--';
});

const cpuClass = computed(() => {
  if (!systemSnapshot.value) return 'usage-warn';
  const p = systemSnapshot.value.cpu_average;
  return p <= 30 ? 'usage-good' : p <= 60 ? 'usage-warn' : p <= 80 ? 'usage-caution' : 'usage-danger';
});

const memClass = computed(() => {
  if (!systemSnapshot.value) return 'usage-warn';
  const p = systemSnapshot.value.memory_usage_percent;
  return p <= 30 ? 'usage-good' : p <= 60 ? 'usage-warn' : p <= 80 ? 'usage-caution' : 'usage-danger';
});

const battClass = computed(() => {
  if (!systemSnapshot.value) return 'usage-warn';
  const p = systemSnapshot.value.battery_percent;
  return p > 50 ? 'usage-good' : p > 20 ? 'usage-warn' : 'usage-danger';
});

const volumeMuted = computed(() => {
  const s = audioSnapshot.value;
  return !s || s.is_muted || !s.has_device;
});

const volumeIconChar = computed(() => {
  const s = audioSnapshot.value;
  if (!s || !s.has_device) return ICON_VOL_MUTE;
  if (s.is_muted) return ICON_VOL_MUTE;
  if (s.volume <= 0) return ICON_VOL_MUTE;
  if (s.volume < 34) return ICON_VOL_LOW;
  if (s.volume < 67) return ICON_VOL_MID;
  return ICON_VOL_HIGH;
});

const volumeLabel = computed(() => {
  const s = audioSnapshot.value;
  if (!s || !s.has_device) return '--';
  return `${s.volume}%`;
});

const brightnessLabel = computed(() => {
  const p = brightnessSnapshot.value?.percent;
  return typeof p === 'number' ? `${p}%` : '--';
});

function pad(n: number) {
  return n.toString().padStart(2, '0');
}

const formattedTime = computed(() => {
  const d = now.value;
  const ts = `${pad(d.getHours())}:${pad(d.getMinutes())}${showSeconds.value ? `:${pad(d.getSeconds())}` : ''}`;
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())} ${ts}`;
});

// --- 事件处理 ---
function buttonClass(i: number) {
  const tagStatus =
    monitorSnapshot.value?.tag_status_vec?.[i] ??
    ({ is_selected: false, is_urg: false, is_filled: false, is_occ: false } as TagStatus);
  const baseClass = getButtonClass(tagStatus);
  const isPressed = pressedButton.value === i;
  return isPressed ? `${baseClass} pressed` : baseClass;
}

async function onTagRelease(index: number) {
  pressedButton.value = null;
  try {
    await invoke('send_tag_command', {
      tagIndex: index,
      isView: true,
      monitorId: monitorNum.value,
    });
  } catch (e) {
    console.error('send_tag_command error:', e);
  }
}

async function onLayoutSelect(idx: number) {
  layoutOpen.value = false;
  try {
    await invoke('send_layout_command', {
      layoutIndex: idx,
      monitorId: monitorNum.value,
    });
  } catch (e) {
    console.error('send_layout_command error:', e);
  }
}

async function onScreenshot() {
  if (isTaking.value) return;
  isTaking.value = true;
  try {
    await invoke('take_screenshot');
  } catch (e) {
    console.error('take_screenshot error:', e);
  } finally {
    setTimeout(() => (isTaking.value = false), 500);
  }
}

async function onToggleMute() {
  try {
    await invoke('toggle_mute');
  } catch (e) {
    console.error('toggle_mute error:', e);
  }
}

async function onVolumeWheel(e: WheelEvent) {
  const delta = e.deltaY < 0 ? 5 : -5;
  try {
    await invoke('adjust_volume', { delta });
  } catch (err) {
    console.error('adjust_volume error:', err);
  }
}

async function onBrightnessClick() {
  try {
    await invoke('adjust_brightness', { delta: 5 });
  } catch (e) {
    console.error('adjust_brightness error:', e);
  }
}

async function onBrightnessRight() {
  try {
    await invoke('adjust_brightness', { delta: -5 });
  } catch (e) {
    console.error('adjust_brightness error:', e);
  }
}

async function onBrightnessWheel(e: WheelEvent) {
  const delta = e.deltaY < 0 ? 5 : -5;
  try {
    await invoke('adjust_brightness', { delta });
  } catch (err) {
    console.error('adjust_brightness error:', err);
  }
}
</script>

<style>
/* 重置所有默认样式 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html,
body {
  margin: 0;
  padding: 0;
  height: 40px !important;
  overflow: hidden;
  font-family:
    "Symbols Nerd Font",
    "JetBrainsMono Nerd Font",
    "FiraCode Nerd Font",
    "Hack Nerd Font",
    system-ui,
    -apple-system,
    BlinkMacSystemFont,
    "Segoe UI",
    Roboto,
    sans-serif;
  background: transparent;
}

/* Nerd Font 图标统一字体回退 */
.nf-icon {
  font-family:
    "Symbols Nerd Font",
    "JetBrainsMono Nerd Font",
    "FiraCode Nerd Font",
    "Hack Nerd Font",
    "Symbols Nerd Font Mono",
    monospace;
  font-size: 15px;
  line-height: 1;
  display: inline-block;
  vertical-align: middle;
  width: 1.2em;
  text-align: center;
}

.pill .nf-icon {
  margin-right: 6px;
}

.emoji-button .nf-icon {
  margin: 0;
  font-size: 18px;
  width: auto;
}

#main,
#app {
  margin: 0;
  padding: 0;
  height: 40px !important;
  overflow: hidden;
}

.button-row {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: space-between;
  margin: 0;
  padding: 1px 6px;
  gap: 8px;
  width: 100vw;
  height: 40px;
  min-height: 40px;
  max-height: 40px;
  background: rgba(255, 255, 255, 0.95);
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  position: relative;
  overflow: visible;
  box-sizing: border-box;
}

.buttons-container {
  display: flex;
  flex-direction: row;
  align-items: center;
  gap: 8px;
  flex-shrink: 1;
  flex-grow: 0;
  min-width: 0;
  overflow: visible;
  padding: 2px 0;
}

.right-info-container {
  display: flex;
  align-items: center;
  gap: 10px;
  flex-shrink: 0;
  flex-grow: 0;
  margin-left: auto;
}

.system-info-container {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-shrink: 0;
}

.system-metric {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 3px 6px;
  background: rgba(248, 249, 250, 0.8);
  border-radius: 6px;
  border: 1px solid rgba(222, 226, 230, 0.8);
  transition: all 0.2s ease;
  cursor: default;
  user-select: none;
}

.system-metric:hover {
  background: rgba(233, 236, 239, 0.9);
  border-color: rgba(173, 181, 189, 0.8);
  transform: scale(1.02);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.metric-icon {
  font-size: 14px;
  line-height: 1;
}

.metric-value {
  font-family:
    "JetBrains Mono", "Fira Code", "Cascadia Code", "SF Mono", Consolas,
    monospace;
  font-size: 13px;
  font-weight: 600;
  min-width: 40px;
  text-align: right;
}

.layout-symbol {
  color: #000000;
  font-size: 14px;
  padding: 4px 8px;
  background-color: rgba(255, 255, 255, 0.1);
  border-radius: 4px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  min-width: 20px;
  text-align: center;
  margin-left: 8px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

/* ==================== 按钮基础样式 ==================== */

.emoji-button {
  width: 38px;
  height: 32px;
  min-width: 38px;
  min-height: 32px;
  max-width: 38px;
  max-height: 32px;
  font-size: 18px;
  border: 1px solid transparent;
  border-radius: 6px;
  background: transparent;
  cursor: pointer;
  transition: all 0.2s ease;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  user-select: none;
  flex-shrink: 0;
  overflow: hidden;
}

.emoji-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  filter: grayscale(50%);
}

.emoji-button > * {
  position: relative;
  z-index: 2;
}

.emoji-button.state-default {
  background: #ffffff;
  border-color: #dee2e6;
}

.emoji-button.state-default:hover:not(:disabled):not(.pressed):not(:active) {
  background: #f8f9fa;
  border-color: #adb5bd;
  transform: scale(1.02);
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.15);
}

/* 各索引位置的颜色状态 */
.emoji-button:nth-child(1).state-occupied { background: rgba(255, 107, 107, 0.3) !important; border-color: rgba(255, 107, 107, 0.6) !important; color: #333 !important; }
.emoji-button:nth-child(1).state-selected { background: rgba(255, 107, 107, 0.7) !important; border-color: rgba(255, 107, 107, 0.9) !important; color: white !important; }
.emoji-button:nth-child(1).state-filtered { background: rgba(255, 107, 107, 1) !important; border-color: rgba(255, 107, 107, 1) !important; color: white !important; box-shadow: 0 2px 8px rgba(255, 107, 107, 0.4); }

.emoji-button:nth-child(2).state-occupied { background: rgba(78, 205, 196, 0.3) !important; border-color: rgba(78, 205, 196, 0.6) !important; color: #333 !important; }
.emoji-button:nth-child(2).state-selected { background: rgba(78, 205, 196, 0.7) !important; border-color: rgba(78, 205, 196, 0.9) !important; color: white !important; }
.emoji-button:nth-child(2).state-filtered { background: rgba(78, 205, 196, 1) !important; border-color: rgba(78, 205, 196, 1) !important; color: white !important; box-shadow: 0 2px 8px rgba(78, 205, 196, 0.4); }

.emoji-button:nth-child(3).state-occupied { background: rgba(69, 183, 209, 0.3) !important; border-color: rgba(69, 183, 209, 0.6) !important; color: #333 !important; }
.emoji-button:nth-child(3).state-selected { background: rgba(69, 183, 209, 0.7) !important; border-color: rgba(69, 183, 209, 0.9) !important; color: white !important; }
.emoji-button:nth-child(3).state-filtered { background: rgba(69, 183, 209, 1) !important; border-color: rgba(69, 183, 209, 1) !important; color: white !important; box-shadow: 0 2px 8px rgba(69, 183, 209, 0.4); }

.emoji-button:nth-child(4).state-occupied { background: rgba(150, 206, 180, 0.3) !important; border-color: rgba(150, 206, 180, 0.6) !important; color: #333 !important; }
.emoji-button:nth-child(4).state-selected { background: rgba(150, 206, 180, 0.7) !important; border-color: rgba(150, 206, 180, 0.9) !important; color: white !important; }
.emoji-button:nth-child(4).state-filtered { background: rgba(150, 206, 180, 1) !important; border-color: rgba(150, 206, 180, 1) !important; color: white !important; box-shadow: 0 2px 8px rgba(150, 206, 180, 0.4); }

.emoji-button:nth-child(5).state-occupied { background: rgba(254, 202, 87, 0.3) !important; border-color: rgba(254, 202, 87, 0.6) !important; color: #333 !important; }
.emoji-button:nth-child(5).state-selected { background: rgba(254, 202, 87, 0.7) !important; border-color: rgba(254, 202, 87, 0.9) !important; color: #333 !important; }
.emoji-button:nth-child(5).state-filtered { background: rgba(254, 202, 87, 1) !important; border-color: rgba(254, 202, 87, 1) !important; color: #333 !important; box-shadow: 0 2px 8px rgba(254, 202, 87, 0.4); }

.emoji-button:nth-child(6).state-occupied { background: rgba(255, 159, 243, 0.3) !important; border-color: rgba(255, 159, 243, 0.6) !important; color: #333 !important; }
.emoji-button:nth-child(6).state-selected { background: rgba(255, 159, 243, 0.7) !important; border-color: rgba(255, 159, 243, 0.9) !important; color: white !important; }
.emoji-button:nth-child(6).state-filtered { background: rgba(255, 159, 243, 1) !important; border-color: rgba(255, 159, 243, 1) !important; color: white !important; box-shadow: 0 2px 8px rgba(255, 159, 243, 0.4); }

.emoji-button:nth-child(7).state-occupied { background: rgba(84, 160, 255, 0.3) !important; border-color: rgba(84, 160, 255, 0.6) !important; color: #333 !important; }
.emoji-button:nth-child(7).state-selected { background: rgba(84, 160, 255, 0.7) !important; border-color: rgba(84, 160, 255, 0.9) !important; color: white !important; }
.emoji-button:nth-child(7).state-filtered { background: rgba(84, 160, 255, 1) !important; border-color: rgba(84, 160, 255, 1) !important; color: white !important; box-shadow: 0 2px 8px rgba(84, 160, 255, 0.4); }

.emoji-button:nth-child(8).state-occupied { background: rgba(95, 39, 205, 0.3) !important; border-color: rgba(95, 39, 205, 0.6) !important; color: #333 !important; }
.emoji-button:nth-child(8).state-selected { background: rgba(95, 39, 205, 0.7) !important; border-color: rgba(95, 39, 205, 0.9) !important; color: white !important; }
.emoji-button:nth-child(8).state-filtered { background: rgba(95, 39, 205, 1) !important; border-color: rgba(95, 39, 205, 1) !important; color: white !important; box-shadow: 0 2px 8px rgba(95, 39, 205, 0.4); }

.emoji-button:nth-child(9).state-occupied { background: rgba(0, 210, 211, 0.3) !important; border-color: rgba(0, 210, 211, 0.6) !important; color: #333 !important; }
.emoji-button:nth-child(9).state-selected { background: rgba(0, 210, 211, 0.7) !important; border-color: rgba(0, 210, 211, 0.9) !important; color: white !important; }
.emoji-button:nth-child(9).state-filtered { background: rgba(0, 210, 211, 1) !important; border-color: rgba(0, 210, 211, 1) !important; color: white !important; box-shadow: 0 2px 8px rgba(0, 210, 211, 0.4); }

.emoji-button.state-urgent {
  background: linear-gradient(135deg, #dc3545, #c82333) !important;
  border-color: #bd2130 !important;
  color: white !important;
}

.emoji-button.state-urgent::after {
  content: "U";
  position: absolute;
  top: -3px;
  right: -3px;
  background: #ffc107;
  border-radius: 50%;
  width: 12px;
  height: 12px;
  border: 1px solid white;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.3);
  font-size: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #000;
  font-weight: bold;
}

.emoji-button.state-filtered::after { content: "●"; position: absolute; top: 2px; right: 2px; color: rgba(255, 255, 255, 0.9); font-size: 10px; text-shadow: 0 1px 2px rgba(0, 0, 0, 0.5); font-weight: bold; }
.emoji-button.state-selected::after { content: "◆"; position: absolute; top: 2px; right: 2px; color: rgba(255, 255, 255, 0.9); font-size: 8px; text-shadow: 0 1px 2px rgba(0, 0, 0, 0.5); font-weight: bold; }
.emoji-button:nth-child(5).state-selected::after,
.emoji-button:nth-child(5).state-filtered::after { color: rgba(51, 51, 51, 0.8); text-shadow: 0 1px 1px rgba(255, 255, 255, 0.3); }
.emoji-button.state-occupied::after { content: "○"; position: absolute; top: 2px; right: 2px; color: rgba(51, 51, 51, 0.7); font-size: 8px; text-shadow: 0 1px 1px rgba(255, 255, 255, 0.3); font-weight: bold; }

/* 按下效果 */
.emoji-button::before {
  content: "";
  position: absolute;
  top: 50%;
  left: 50%;
  width: 0;
  height: 0;
  border-radius: 50%;
  background: radial-gradient(circle, rgba(255, 255, 255, 0.6) 0%, rgba(255, 255, 255, 0) 70%);
  transform: translate(-50%, -50%);
  opacity: 0;
  pointer-events: none;
  z-index: 1;
  transition: all 0.3s ease;
}

.emoji-button.pressed,
.emoji-button:active {
  transform: scale(0.92) !important;
  box-shadow:
    inset 0 2px 6px rgba(0, 0, 0, 0.3),
    0 1px 2px rgba(0, 0, 0, 0.2) !important;
  transition: all 0.1s ease !important;
}

.emoji-button.state-default.pressed,
.emoji-button.state-default:active {
  background: #dee2e6 !important;
  border-color: #6c757d !important;
}

.emoji-button.state-occupied.pressed,
.emoji-button.state-selected.pressed,
.emoji-button.state-filtered.pressed {
  opacity: 0.8;
  transform: scale(0.92) !important;
  box-shadow: inset 0 2px 6px rgba(0, 0, 0, 0.3) !important;
}

.emoji-button:hover:not(.pressed):not(:active) {
  transform: scale(1.05);
  transition: all 0.2s ease;
}

.emoji-button:disabled.pressed,
.emoji-button:disabled:active {
  transform: none !important;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1) !important;
  background: #f8f9fa !important;
}

.emoji-button:disabled::before {
  display: none;
}

@media (hover: none) {
  .emoji-button:hover { transform: none; }
  .emoji-button.pressed,
  .emoji-button:active { transform: scale(0.95) !important; }
}

/* 通用 pill 样式 */
.pill {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: 12px;
  padding: 4px 10px;
  font-size: 14px;
  line-height: 1;
  border: 1px solid transparent;
  transition: all 120ms ease-in-out;
  white-space: nowrap;
}

.system-info-container {
  display: inline-flex;
  align-items: center;
  gap: 6px;
}

.usage-pill {
  color: #fff;
  border-width: 1px;
}

.usage-good { background: rgba(31, 191, 81, 0.90); border-color: #1fbf51; }
.usage-warn { background: rgba(244, 194, 13, 0.90); border-color: #f4c20d; color: #000; }
.usage-caution { background: rgba(255, 140, 26, 0.90); border-color: #ff8c1a; }
.usage-danger { background: rgba(229, 57, 53, 0.90); border-color: #e53935; }

.layout-controls {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  margin-left: 6px;
}

.layout-toggle { cursor: pointer; color: #fff; }
.layout-toggle.open { background: rgba(60, 179, 113, 0.85); border-color: #3cb371; }
.layout-toggle.closed { background: rgba(211, 84, 0, 0.85); border-color: #d35400; }
.layout-toggle:hover { filter: brightness(1.05); border-width: 2px; }

.layout-selector { display: inline-flex; align-items: center; gap: 6px; }
.layout-option { cursor: pointer; color: #fff; background: rgba(65, 105, 225, 0.85); border-color: #4169e1; }
.layout-option.current { background: rgba(60, 179, 113, 0.9); border-color: #3cb371; border-width: 2px; }
.layout-option:hover { filter: brightness(1.05); border-width: 2px; }

.screenshot-pill { cursor: pointer; color: #fff; background: rgba(0, 204, 204, 0.9); border-color: #00cccc; }
.screenshot-pill:hover { background: rgba(255, 136, 0, 0.95); border-color: #ff8800; }

.time-pill { color: #fff; background: rgba(77, 163, 255, 0.90); border-color: #4da3ff; cursor: pointer; }
.monitor-pill { color: #fff; background: rgba(155, 89, 182, 0.90); border-color: #9b59b6; }
.scale-pill { color: #fff; background: rgba(120, 120, 120, 0.88); border-color: #777; }

/* 音量 pill */
.volume-pill {
  cursor: pointer;
  color: #fff;
  background: rgba(20, 184, 166, 0.90);
  border-color: #14b8a6;
  user-select: none;
}
.volume-pill:hover { filter: brightness(1.08); border-width: 2px; }
.volume-pill.muted { background: rgba(120, 120, 120, 0.85); border-color: #888; color: #eee; }

/* 亮度 pill */
.brightness-pill {
  cursor: pointer;
  color: #1f2937;
  background: rgba(253, 224, 71, 0.92);
  border-color: #facc15;
  user-select: none;
}
.brightness-pill:hover { filter: brightness(1.05); border-width: 2px; }

.spacer {
  flex: 1 1 auto;
}
</style>
