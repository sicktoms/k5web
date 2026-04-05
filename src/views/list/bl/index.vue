<template>
  <div class="container">
    <Breadcrumb :items="[$t('menu.list'), 'Multi-booting']" />

    <!-- ── EEPROM Visual Map ─────────────────────────────────────────────────── -->
    <a-card class="general-card" style="margin-bottom: 16px;">
      <template #title>
        EEPROM Layout Map
        <span style="font-weight: normal; font-size: 0.82rem; margin-left: 8px; color: var(--color-text-3);">
          512 KB total · hover for details · click a firmware slot to select it
        </span>
      </template>

      <!-- Row 1: Management zone -->
      <div class="zone-header">
        <span class="zone-badge mgmt-badge">Mgmt Zone</span>
        <span class="zone-addr">0x00000 – 0x43FFF &nbsp;(272 KB)</span>
      </div>
      <div class="map-bar">
        <div
          v-for="(r, i) in mgmtRegions"
          :key="'m' + i"
          class="map-seg"
          :class="{ clickable: r.clickable }"
          :style="{ flex: r.flex, background: r.color }"
          @mouseenter="hoveredRegion = r"
          @mouseleave="hoveredRegion = null"
        >
          <span v-if="r.flex >= 100" class="seg-label">{{ r.shortName }}</span>
        </div>
      </div>

      <!-- Row 2: Firmware zone -->
      <div class="zone-header" style="margin-top: 10px;">
        <span class="zone-badge fw-badge">FW Zone</span>
        <span class="zone-addr">0x44000 – 0x7FFFF &nbsp;(240 KB)</span>
      </div>
      <div class="map-bar">
        <div
          v-for="(r, i) in fwRegions"
          :key="'f' + i"
          class="map-seg"
          :class="{ clickable: r.clickable }"
          :style="{ flex: r.flex, background: r.bgColor }"
          @mouseenter="hoveredRegion = r"
          @mouseleave="hoveredRegion = null"
          @click="r.clickable && onRegionClick(r)"
        >
          <!-- filled portion = actual firmware bytes used -->
          <div
            v-if="r.type === 'firmware' && r.fillPct > 0"
            class="fw-fill"
            :style="{ width: r.fillPct + '%', background: r.color }"
          />
          <span class="seg-label">{{ r.shortName }}</span>
        </div>
      </div>

      <!-- Hover info box -->
      <div class="map-info" :style="hoveredRegion ? { borderLeftColor: hoveredRegion.color } : {}">
        <template v-if="hoveredRegion">
          <span class="info-swatch" :style="{ background: hoveredRegion.color }" />
          <strong>{{ hoveredRegion.name }}</strong>
          <code class="info-addr">0x{{ pad5(hoveredRegion.start) }} – 0x{{ pad5(hoveredRegion.end) }}</code>
          <span class="info-size">{{ (hoveredRegion.size / 1024).toFixed(1) }} KB</span>
          <span class="info-sep">·</span>
          <span class="info-desc">{{ hoveredRegion.desc }}</span>
        </template>
        <span v-else class="info-placeholder">Hover a region for details · Click a firmware slot to jump to it</span>
      </div>

      <!-- Legend -->
      <div class="map-legend">
        <div v-for="item in legendItems" :key="item.name" class="legend-item">
          <div class="legend-swatch" :style="{ background: item.color }" />
          <span>{{ item.name }}</span>
        </div>
      </div>
    </a-card>

    <!-- ── Tabs ──────────────────────────────────────────────────────────────── -->
    <a-card class="general-card" :body-style="{ paddingTop: 0 }" :loading="loading">
      <a-tabs v-model:active-key="activeTab">

        <!-- ── Tab 1: Flash All ──────────────────────────────────────────────── -->
        <a-tab-pane key="flash" title="Flash All">
          <div style="padding-bottom: 8px;">
            <a-alert type="warning" style="margin-bottom: 16px;">
              ⚠ Experimental. Connect in <strong>bootloader mode</strong> (hold MENU on power-on, version shows "L_BL003")
              for full EEPROM access. "Flash bootloader only" also works when a regular firmware is running.
            </a-alert>

            <a-row style="margin-bottom: 16px; align-items: center;" :gutter="16">
              <a-col :flex="'none'">
                <span style="margin-right: 8px;">Firmware slots:</span>
                <a-select v-model="state.numSlots" style="width: 80px;" @change="rebuildSlots">
                  <a-option v-for="n in [1,2,3,4,5,6]" :key="n" :value="n">{{ n }}</a-option>
                </a-select>
                <span style="margin-left: 16px; color: var(--color-text-3); font-size: 0.85rem;">
                  {{ slotSizeKb }} KB/slot &nbsp;·&nbsp; first slot @ 0x44000
                </span>
              </a-col>
              <a-col :flex="'auto'" style="text-align: right; display: flex; gap: 8px; justify-content: flex-end; flex-wrap: wrap;">
                <a-button :loading="state.writing" @click="writeBootloaderOnly">
                  <template #icon><icon-upload /></template>
                  Flash bootloader only
                </a-button>
                <a-button
                  type="primary" status="danger"
                  :loading="state.writing"
                  :disabled="!state.slots.some((s:any) => s.firmware && !s.oversize)"
                  @click="writeAll"
                >
                  <template #icon><icon-thunderbolt /></template>
                  Write all to device
                </a-button>
              </a-col>
            </a-row>

            <a-row :gutter="[16, 16]">
              <a-col v-for="(slot, idx) in state.slots" :key="idx" :span="12">
                <a-card
                  :id="'slot-card-' + idx"
                  :bordered="true"
                  class="slot-card"
                  :style="slotCardStyle(slot, idx)"
                >
                  <template #title>
                    <span :style="slot.firmware ? { color: slot.color } : {}">Slot {{ slot.index }}</span>
                    <span v-if="slot.firmware" style="font-weight: normal; font-size: 0.82rem; margin-left: 8px; color: var(--color-text-3);">
                      {{ (slot.firmware.length / 1024).toFixed(1) }} / {{ slotSizeKb }} KB
                      &nbsp;({{ ((Math.ceil(slot.firmware.length / 0x40) * 0x40 / slotSize) * 100).toFixed(0) }}%)
                    </span>
                  </template>
                  <template #extra>
                    <div v-if="slot.firmware" :style="{ width: '12px', height: '12px', borderRadius: '2px', background: slot.color }" />
                  </template>

                  <!-- Mini fill bar -->
                  <div v-if="slot.firmware" class="slot-fill-bar" style="margin-bottom: 10px;">
                    <div class="slot-fill-track">
                      <div
                        class="slot-fill-inner"
                        :style="{
                          width: Math.min((Math.ceil(slot.firmware.length / 0x40) * 0x40 / slotSize) * 100, 100) + '%',
                          background: slot.oversize ? '#f53f3f' : slot.color
                        }"
                      />
                    </div>
                  </div>

                  <a-form-item label="Name" style="margin-bottom: 10px;">
                    <a-input
                      v-model="slot.name"
                      :max-length="13"
                      show-word-limit
                      :placeholder="'Slot ' + slot.index"
                      @change="() => slot.name = slot.name.replace(/[^\x00-\x7f]/g, '')"
                    />
                  </a-form-item>
                  <a-space>
                    <a-button @click="selectFile(idx)">
                      <template #icon><icon-upload /></template>
                      {{ slot.firmware ? slot.filename : 'Select .bin file' }}
                    </a-button>
                    <a-tag v-if="slot.firmware && slot.oversize" color="red">TOO LARGE</a-tag>
                    <a-tag v-else-if="slot.firmware" color="green">OK</a-tag>
                    <a-button v-if="slot.firmware" type="text" status="danger" size="small" @click="clearSlot(idx)">
                      <template #icon><icon-delete /></template>
                    </a-button>
                  </a-space>
                </a-card>
              </a-col>
            </a-row>
          </div>
        </a-tab-pane>

        <!-- ── Tab 2: Update Single Slot ─────────────────────────────────────── -->
        <a-tab-pane key="update" title="Update Single Slot">
          <div style="padding-bottom: 8px;">
            <a-alert type="info" style="margin-bottom: 16px;">
              Power off → hold MENU → power on → connect here. Version shown should be "L_BL003".
              Do <strong>not</strong> use Quansheng mode (PTT + power on) — it doesn't support 32-bit EEPROM addressing.
            </a-alert>
            <a-alert
              v-if="appStore.connectState && appStore.configuration?.uart !== 'losehu'"
              type="error"
              style="margin-bottom: 16px;"
            >
              Connected as "{{ appStore.firmwareVersion || '?' }}" — 32-bit EEPROM access unavailable.
              Enter bootloader mode first (MENU + power on).
            </a-alert>

            <a-button type="primary" :loading="state.reading" style="margin-bottom: 16px;" @click="readSlots">
              <template #icon><icon-refresh /></template>
              Read slots from device
            </a-button>

            <a-table
              v-if="state.deviceSlots.length > 0"
              :data="state.deviceSlots"
              :pagination="false"
              :bordered="{ cell: true }"
              style="margin-bottom: 16px;"
            >
              <template #columns>
                <a-table-column title="#" data-index="index" :width="50" />
                <a-table-column title="Name" data-index="name" />
                <a-table-column title="Start" data-index="startHex" />
                <a-table-column title="End" data-index="endHex" />
                <a-table-column title="Capacity" data-index="sizeKb" />
                <a-table-column title="">
                  <template #cell="{ record }">
                    <a-button
                      size="small"
                      :type="state.selectedSlot?.index === record.index ? 'primary' : 'outline'"
                      @click="selectDeviceSlot(record)"
                    >
                      {{ state.selectedSlot?.index === record.index ? 'Selected ✓' : 'Select' }}
                    </a-button>
                  </template>
                </a-table-column>
              </template>
            </a-table>

            <template v-if="state.selectedSlot">
              <a-divider />
              <div style="display: flex; align-items: center; gap: 12px; flex-wrap: wrap; margin-bottom: 12px;">
                <a-button @click="selectUpdateFile">
                  <template #icon><icon-upload /></template>
                  {{ state.newFirmware ? state.newFirmwareName : 'Select .bin file' }}
                </a-button>
                <span v-if="state.newFirmware" style="color: var(--color-text-3); font-size: 0.85rem;">
                  {{ (state.newFirmware.length / 1024).toFixed(1) }} KB
                </span>
              </div>

              <a-alert v-if="state.newFirmware && state.sizeError" type="error" style="margin-bottom: 12px;">
                Firmware too large for this slot (max {{ (state.selectedSlot.capacity / 1024).toFixed(1) }} KB).
              </a-alert>

              <div v-if="state.newFirmware && !state.sizeError" style="display: flex; align-items: center; gap: 8px; margin-bottom: 16px;">
                <span>Slot name:</span>
                <a-input
                  v-model="state.newName"
                  :max-length="13"
                  show-word-limit
                  style="width: 220px;"
                  @change="() => state.newName = state.newName.replace(/[^\x00-\x7f]/g, '')"
                />
              </div>

              <a-button
                v-if="state.newFirmware && !state.sizeError"
                type="primary"
                status="danger"
                :loading="state.writing"
                @click="writeSlot"
              >
                <template #icon><icon-thunderbolt /></template>
                Flash slot {{ state.selectedSlot.index }} ("{{ state.selectedSlot.name }}")
              </a-button>
            </template>
          </div>
        </a-tab-pane>

      </a-tabs>
    </a-card>

    <!-- ── Shared status log ──────────────────────────────────────────────────── -->
    <div
      id="statusArea"
      style="height: 16em; background-color: var(--color-bg-3); color: var(--color-text-3);
             overflow: auto; padding: 16px; margin-top: 12px;
             font-family: monospace; font-size: 0.82rem; border-radius: 4px;"
      v-html="state.status"
    />
  </div>
</template>

<script lang="ts" setup>
import { reactive, computed, ref, onMounted, nextTick } from 'vue';
import useLoading from '@/hooks/loading';
import { useAppStore } from '@/store';
import {
  eeprom_write, eeprom_reboot, eeprom_init, eeprom_read,
  stringToUint8Array, check_eeprom, hexReverseStringToUint8Array, unpack,
} from '@/utils/serial.js';

const appStore = useAppStore();
const { loading, setLoading } = useLoading(false);

// ── EEPROM layout constants ───────────────────────────────────────────────────
const EEPROM_SIZE   = 0x80000;   // 512 KB
const FW_BASE       = 0x44000;   // first byte of firmware zone
const BL_ADDR       = 0x41000;   // Bootloader B
const META_BASE     = 0x40000;   // metadata block
const CFG_SLOT_BASE = 0x20000;   // first config snapshot slot
const CFG_SLOT_SIZE = 0x4000;    // 16 KB per config slot (matches CONFIG_SIZE in bootloader)

// ── Slot color palette ────────────────────────────────────────────────────────
const PALETTE = ['#4CAF50', '#7C4DFF', '#F44336', '#00BCD4', '#FF9800', '#E91E63'];

function slotColor(idx: number) { return PALETTE[idx % PALETTE.length]; }

function lighten(hex: string, t = 0.62): string {
  const r = parseInt(hex.slice(1, 3), 16);
  const g = parseInt(hex.slice(3, 5), 16);
  const b = parseInt(hex.slice(5, 7), 16);
  return '#' + [r, g, b].map(c => Math.round(c + (255 - c) * t).toString(16).padStart(2, '0')).join('');
}

// ── App state ─────────────────────────────────────────────────────────────────
const activeTab     = ref<string>('flash');
const hoveredRegion = ref<any>(null);

const state: any = reactive({
  // Flash All
  numSlots:  4,
  slots:     [] as any[],
  bl:        undefined as Uint8Array | undefined,
  blName:    '',
  writing:   false,
  highlightSlot: -1,

  // Update Single Slot
  reading:         false,
  deviceSlots:     [] as any[],
  selectedSlot:    null as any,
  newFirmware:     null as Uint8Array | null,
  newFirmwareName: '',
  newName:         '',
  sizeError:       false,

  // Shared
  status: '',
});

// ── Computed geometry ─────────────────────────────────────────────────────────
const slotSize = computed(() => {
  const raw = Math.floor((EEPROM_SIZE - FW_BASE) / state.numSlots);
  return Math.floor(raw / 0x40) * 0x40;
});
const slotSizeKb = computed(() => (slotSize.value / 1024).toFixed(1));

function slotStart(i: number) { return FW_BASE + i * slotSize.value; }
function slotEnd(i: number)   { return slotStart(i) + slotSize.value - 1; }

// ── EEPROM map ────────────────────────────────────────────────────────────────
function pad5(n: number) { return n.toString(16).toUpperCase().padStart(5, '0'); }

const LIVE_COLOR = '#1565C0';
const FREE_COLOR = '#9E9E9E';
const META_COLOR = '#E65100';
const BL_COLOR   = '#B71C1C';

function reg(name: string, shortName: string, start: number, size: number,
             color: string, desc: string, extra: Record<string, any> = {}): any {
  return {
    name, shortName, start, size,
    end: start + size - 1,
    flex: Math.max(Math.round(size / 64), 4),
    color, bgColor: color,
    desc, type: 'other', clickable: false,
    ...extra,
  };
}

// Management zone (0x00000–0x43FFF)
const mgmtRegions = computed<any[]>(() => {
  const n  = Math.min(state.numSlots, PALETTE.length);
  const rs: any[] = [];

  rs.push(reg(
    'Live Config', 'Config', 0x00000, 0x4000, LIVE_COLOR,
    'Active radio config — channels, settings, calibration, firmware-specific data (e.g. F4HWN settings at 0x2000+). ' +
    'The bootloader saves this entire 16 KB region (0x0000–0x3FFF) to the corresponding Config Slot ' +
    'before every firmware switch, and restores it when switching back.',
    { type: 'live' }
  ));

  rs.push(reg('Free', '', 0x04000, 0x1C000, FREE_COLOR,
    'Unused free zone — 112 KB available for user data or future bootloader features.'));

  for (let i = 0; i < n; i++) {
    rs.push(reg(
      `Config Slot ${i + 1}`, `Cfg ${i + 1}`,
      CFG_SLOT_BASE + i * CFG_SLOT_SIZE, CFG_SLOT_SIZE,
      lighten(slotColor(i)),
      `Config snapshot for firmware slot ${i + 1}. ` +
      `The bootloader copies 0x0000–0x3FFF here before switching away from firmware ${i + 1}, ` +
      `and restores it when switching back. ` +
      `Blanked (all 0xFF) when a new firmware is written to slot ${i + 1}, ` +
      `so the first boot gets a clean slate from the firmware's built-in defaults.`,
      { type: 'config', cfgFor: i + 1 }
    ));
  }

  const cfgEnd = CFG_SLOT_BASE + n * CFG_SLOT_SIZE;
  if (cfgEnd < META_BASE) {
    rs.push(reg('Free', '', cfgEnd, META_BASE - cfgEnd, FREE_COLOR,
      `Unused free zone — ${((META_BASE - cfgEnd) / 1024).toFixed(0)} KB.`));
  }

  rs.push(reg(
    'Metadata', 'Meta', META_BASE, 0x1000, META_COLOR,
    'Bootloader metadata block. ' +
    '0x40000: firmware count (1 B). ' +
    '0x40001: boot mode flag. ' +
    '0x40002: last-run firmware index (scratch byte — set to 0xFF after full reflash). ' +
    '0x40008: bootloader name (8 B). ' +
    '0x40020+: slot table — 32 B per slot: 16 B name | 4 B start addr LE | 4 B end addr LE | 8 B padding.',
    { type: 'meta' }
  ));

  rs.push(reg(
    'Bootloader B', 'BL-B', BL_ADDR, 0x3000, BL_COLOR,
    'Losehu Bootloader B (L_BL003.bin, ~11 KB). ' +
    'Stored in EEPROM and executed by Bootloader A (resident in MCU flash) ' +
    'when the radio is powered on with the MENU button held. ' +
    'Handles firmware selection, config save/restore, and UART flashing.',
    { type: 'bootloader' }
  ));

  return rs;
});

// Firmware zone (0x44000–0x7FFFF)
const fwRegions = computed<any[]>(() => {
  const rs: any[] = [];

  for (let i = 0; i < state.slots.length; i++) {
    const slot   = state.slots[i];
    const cap    = slotSize.value;
    const fwSz   = slot.firmware ? Math.ceil(slot.firmware.length / 0x40) * 0x40 : 0;
    const fillPct = fwSz > 0 ? Math.min((fwSz / cap) * 100, 100) : 0;
    const base   = slotColor(i);

    rs.push({
      name:      `FW Slot ${slot.index}${slot.name ? ' · ' + slot.name : ''}`,
      shortName: slot.name || `Slot ${slot.index}`,
      start:     slot.start,
      end:       slot.end,
      size:      cap,
      flex:      Math.max(Math.round(cap / 64), 8),
      color:     base,
      bgColor:   slot.firmware ? lighten(base, 0.68) : FREE_COLOR,
      desc: slot.firmware
        ? `Slot ${slot.index} "${slot.name || ''}" — ` +
          `${(fwSz / 1024).toFixed(1)} KB used (${fillPct.toFixed(0)}%) of ${(cap / 1024).toFixed(1)} KB capacity. ` +
          `Range: 0x${pad5(slot.start)}–0x${pad5(slot.end)}. ` +
          `Config snapshot at 0x${(CFG_SLOT_BASE + i * CFG_SLOT_SIZE).toString(16).toUpperCase()}. ` +
          `Click to highlight this slot.`
        : `Slot ${slot.index} — empty, ${(cap / 1024).toFixed(1)} KB capacity. ` +
          `Range: 0x${pad5(slot.start)}–0x${pad5(slot.end)}. Click to jump to this slot.`,
      type:        'firmware',
      hasFirmware: !!slot.firmware,
      fillPct,
      slotIdx:     i,
      clickable:   true,
    });
  }

  // Free tail after last slot
  if (state.slots.length > 0) {
    const tail = slotStart(state.slots.length);
    if (tail < EEPROM_SIZE) {
      rs.push(reg('Free', 'Free', tail, EEPROM_SIZE - tail, FREE_COLOR,
        `Unused EEPROM space — ${((EEPROM_SIZE - tail) / 1024).toFixed(1)} KB.`));
    }
  }

  return rs;
});

// Legend
const legendItems = computed(() => {
  const n = Math.min(state.numSlots, PALETTE.length);
  const items: { name: string; color: string }[] = [
    { name: 'Live Config',   color: LIVE_COLOR },
    { name: 'Free / Unused', color: FREE_COLOR },
  ];
  for (let i = 0; i < n; i++)
    items.push({ name: `Config Slot ${i + 1}`, color: lighten(slotColor(i)) });
  items.push({ name: 'Metadata', color: META_COLOR }, { name: 'Bootloader B', color: BL_COLOR });
  for (let i = 0; i < n; i++)
    items.push({ name: `FW Slot ${i + 1}${state.slots[i]?.name ? ' · ' + state.slots[i].name : ''}`, color: slotColor(i) });
  return items;
});

// ── Map interactivity ─────────────────────────────────────────────────────────
function onRegionClick(r: any) {
  if (r.type !== 'firmware') return;
  activeTab.value = 'flash';
  state.highlightSlot = r.slotIdx;
  nextTick(() => {
    document.getElementById('slot-card-' + r.slotIdx)?.scrollIntoView({ behavior: 'smooth', block: 'center' });
    setTimeout(() => { state.highlightSlot = -1; }, 2500);
  });
}

function slotCardStyle(slot: any, idx: number): Record<string, string> {
  if (state.highlightSlot === idx) return { borderColor: '#1890ff', borderWidth: '2px' };
  if (slot.firmware) return { borderColor: slot.color };
  return {};
}

// ── Slot management (Flash All) ───────────────────────────────────────────────
function makeSlot(idx: number): any {
  return {
    index: idx + 1, start: slotStart(idx), end: slotEnd(idx),
    name: '', filename: '', firmware: null as Uint8Array | null,
    color: slotColor(idx), oversize: false,
  };
}

function rebuildSlots() {
  const prev = state.slots.slice();
  state.slots = Array.from({ length: state.numSlots }, (_: any, i: number) => {
    const s = makeSlot(i);
    if (prev[i]) {
      s.name = prev[i].name; s.filename = prev[i].filename; s.firmware = prev[i].firmware;
      if (s.firmware) s.oversize = Math.ceil(s.firmware.length / 0x40) * 0x40 > slotSize.value;
    }
    return s;
  });
}

function clearSlot(idx: number) {
  Object.assign(state.slots[idx], { firmware: null, filename: '', oversize: false });
}

// ── Bootloader loading ────────────────────────────────────────────────────────
async function loadBL() {
  setLoading(true);
  state.blName = 'L_BL003.bin';
  try {
    const res = await fetch('/L_BL003.bin');
    if (res.body) {
      const reader = res.body.getReader();
      const chunks: number[] = [];
      while (true) { const { done, value } = await reader.read(); if (done) break; chunks.push(...value); }
      const bl = new Uint8Array(0x3000); bl.set(chunks, 0); state.bl = bl;
    }
  } finally { setLoading(false); }
}

onMounted(() => { rebuildSlots(); loadBL(); });

// ── File selection ────────────────────────────────────────────────────────────
function selectFile(idx: number) {
  const input = document.createElement('input');
  input.type = 'file'; input.accept = '.bin';
  input.onchange = async () => {
    if (!input.files?.length) return;
    const file = input.files[0];
    const fw   = unpack(new Uint8Array(await file.arrayBuffer()));
    const slot = state.slots[idx];
    slot.filename = file.name;
    slot.firmware = fw;
    slot.oversize = Math.ceil(fw.length / 0x40) * 0x40 > slotSize.value;
    if (!slot.name)
      slot.name = file.name.replace(/\.bin$/i, '').replace(/[^\x00-\x7f]/g, '').substring(0, 13);
    if (slot.oversize)
      log(`WARNING slot ${slot.index}: "${file.name}" is ${(Math.ceil(fw.length/0x40)*0x40/1024).toFixed(1)} KB, capacity is ${slotSizeKb.value} KB.`);
  };
  input.click();
}

// ── Logging ───────────────────────────────────────────────────────────────────
function log(msg: string) {
  state.status += msg + '<br/>';
  nextTick(() => { const el = document.getElementById('statusArea'); if (el) el.scrollTop = el.scrollHeight; });
}

function errMsg(e: any): string {
  if (e instanceof Error) return e.message;
  if (typeof e === 'string') return e;
  return JSON.stringify(e);
}

// ── Write helpers ─────────────────────────────────────────────────────────────
function isEqual(a: Uint8Array, b: Uint8Array) {
  return a.length === b.length && a.every((v, i) => v === b[i]);
}

async function writeRange(start: number, data: Uint8Array | number[], label: string) {
  const arr = data instanceof Uint8Array ? data : new Uint8Array(data);
  for (let i = start; i < start + arr.length; i += 0x40) {
    const chunk = arr.slice(i - start, i - start + 0x40);
    await eeprom_write(appStore.connectPort, i, chunk, chunk.length, appStore.configuration?.uart);
    const verify = await eeprom_read(appStore.connectPort, i, chunk.length, appStore.configuration?.uart);
    if (!isEqual(chunk, verify)) { log(`${label} write error @ 0x${i.toString(16)}, retrying…`); i -= 0x40; continue; }
    log(`${label} ${(((i - start) / arr.length) * 100).toFixed(1)}%`);
  }
}

// ── Config slot clear — BUG FIX ───────────────────────────────────────────────
// The bootloader checks config_slot_blank() before restoring saved config.
// Without this, writing a new firmware to a slot leaves stale config snapshot data,
// so the bootloader restores it on first boot, overwriting the new firmware's defaults.
// Fix: blank the 16 KB config slot after every firmware write.
async function clearConfigSlot(slotIndex: number) {
  const addr  = CFG_SLOT_BASE + (slotIndex - 1) * CFG_SLOT_SIZE;
  const blank = new Uint8Array(0x40).fill(0xFF);
  log(`Clearing config slot ${slotIndex} @ 0x${addr.toString(16).toUpperCase()}…`);
  for (let i = addr; i < addr + CFG_SLOT_SIZE; i += 0x40)
    await eeprom_write(appStore.connectPort, i, blank, 0x40, appStore.configuration?.uart);
}

// ── Write bootloader only ─────────────────────────────────────────────────────
async function writeBootloaderOnly() {
  if (!appStore.connectState) { alert(sessionStorage.getItem('noticeConnectK5')); return; }
  state.writing = true; state.status = '';
  try {
    const sz = await check_eeprom(appStore.connectPort, appStore.configuration?.uart);
    if (sz < 0x80000) { alert('Only 4Mbit (512 KB) EEPROM supported.'); state.writing = false; return; }
    await eeprom_init(appStore.connectPort);
    log('--- Writing bootloader B ---');
    await writeRange(BL_ADDR, state.bl, 'Bootloader');
    log('--- Done! Rebooting… ---');
    await eeprom_reboot(appStore.connectPort);
  } catch (e: any) { log('Error: ' + errMsg(e)); }
  state.writing = false;
}

// ── Write all ────────────────────────────────────────────────────────────────
async function writeAll() {
  if (!appStore.connectState) { alert(sessionStorage.getItem('noticeConnectK5')); return; }
  const filled = state.slots.filter((s: any) => s.firmware && !s.oversize);
  if (!filled.length) { log('No valid firmware assigned to any slot.'); return; }

  state.writing = true; state.status = '';
  try {
    const sz = await check_eeprom(appStore.connectPort, appStore.configuration?.uart);
    if (sz < 0x80000) { alert('Only 4Mbit (512 KB) EEPROM supported.'); state.writing = false; return; }
    await eeprom_init(appStore.connectPort);

    log('--- Writing bootloader B ---');
    await writeRange(BL_ADDR, state.bl, 'Bootloader');

    log('--- Writing metadata ---');
    await writeRange(META_BASE, new Uint8Array([filled.length]), 'FW count');
    const blNameArr = new Uint8Array(8);
    blNameArr.set(stringToUint8Array(state.blName.split('.')[0]));
    await writeRange(0x40008, blNameArr, 'BL name');

    // Reset last-run FW index so bootloader starts completely fresh on first switch
    // (no attempt to save "current" config which would just be whatever was left in EEPROM)
    await eeprom_write(appStore.connectPort, 0x40002, new Uint8Array([0xFF]), 1, appStore.configuration?.uart);
    log('Reset last-FW index (0x40002 ← 0xFF).');

    // Build and write slot table
    const metaBuf: number[] = [];
    for (const slot of filled) {
      const pad = new Uint8Array(Math.ceil(slot.firmware.length / 0x40) * 0x40).fill(0xFF);
      pad.set(slot.firmware, 0);
      slot._padded = pad;
      const nameArr = new Uint8Array(16);
      nameArr.set(stringToUint8Array((slot.name || `Slot${slot.index}`).substring(0, 13)));
      const startArr = new Uint8Array(4);
      startArr.set(hexReverseStringToUint8Array(slot.start.toString(16)));
      const endArr = new Uint8Array(4);
      endArr.set(hexReverseStringToUint8Array((slot.start + pad.length - 1).toString(16)));
      metaBuf.push(...nameArr, ...startArr, ...endArr, ...new Uint8Array(8));
    }
    await writeRange(0x40020, new Uint8Array(metaBuf), 'Metadata');

    // Write each firmware binary, then blank its config slot
    for (const slot of filled) {
      log(`--- Slot ${slot.index}: "${slot.name}" @ 0x${slot.start.toString(16).toUpperCase()} ---`);
      await writeRange(slot.start, slot._padded, slot.name || `Slot${slot.index}`);
      if (slot.index <= PALETTE.length) await clearConfigSlot(slot.index);
    }

    log('--- Done! Rebooting… ---');
    await eeprom_reboot(appStore.connectPort);
  } catch (e: any) { log('Error: ' + errMsg(e)); }
  state.writing = false;
}

// ── Read device slots (Update Single Slot tab) ────────────────────────────────
async function readBytes(addr: number, size: number): Promise<Uint8Array> {
  const out = new Uint8Array(size);
  for (let off = 0; off < size;) {
    const n    = Math.min(0x40, size - off);
    const data = await eeprom_read(appStore.connectPort, addr + off, n, appStore.configuration?.uart);
    out.set(data.slice(0, n), off); off += n;
  }
  return out;
}

function u32le(buf: Uint8Array, off: number) {
  return buf[off] | (buf[off+1] << 8) | (buf[off+2] << 16) | (buf[off+3] << 24);
}

async function readSlots() {
  if (!appStore.connectState) { alert(sessionStorage.getItem('noticeConnectK5')); return; }
  if (appStore.configuration?.uart !== 'losehu') {
    log('ERROR: requires losehu bootloader mode (MENU + power on). Reconnect in bootloader mode.'); return;
  }
  state.reading = true; state.deviceSlots = []; state.selectedSlot = null; state.status = '';
  try {
    await eeprom_init(appStore.connectPort);
    const countBuf = await readBytes(0x40000, 1);
    const count    = countBuf[0];
    if (!count || count > 64) { log('No valid multiboot metadata found. Use "Flash All" first.'); return; }
    log(`Found ${count} firmware slot(s).`);

    const table = await readBytes(0x40020, count * 32);
    const raw   = Array.from({ length: count }, (_, i) => {
      const b = i * 32;
      let name = '';
      for (let j = 0; j < 13 && table[b + j]; j++) name += String.fromCharCode(table[b + j]);
      return { name: name || `Slot ${i + 1}`, start: u32le(table, b + 16), end: u32le(table, b + 20) };
    });

    state.deviceSlots = raw.map((r, i) => {
      const nextStart = i + 1 < raw.length ? raw[i + 1].start : 0x80000;
      const capacity  = nextStart - r.start;
      return {
        index: i + 1, name: r.name, start: r.start, end: r.end,
        nextStart, capacity,
        startHex: '0x' + r.start.toString(16).toUpperCase().padStart(5, '0'),
        endHex:   '0x' + r.end.toString(16).toUpperCase().padStart(5, '0'),
        sizeKb:   (capacity / 1024).toFixed(1) + ' KB',
      };
    });
    state.deviceSlots.forEach((s: any) =>
      log(`Slot ${s.index}: "${s.name}"  ${s.startHex}–${s.endHex}  cap ${s.sizeKb}`));
  } catch (e: any) { log('Error reading slots: ' + errMsg(e)); }
  state.reading = false;
}

// ── Update Single Slot: select + write ───────────────────────────────────────
function selectDeviceSlot(slot: any) {
  Object.assign(state, { selectedSlot: slot, newFirmware: null, newFirmwareName: '', newName: slot.name, sizeError: false });
}

function selectUpdateFile() {
  const input = document.createElement('input');
  input.type = 'file'; input.accept = '.bin';
  input.onchange = async () => {
    if (!input.files?.length) return;
    const file   = input.files[0];
    const fw     = unpack(new Uint8Array(await file.arrayBuffer()));
    const needed = Math.ceil(fw.length / 0x40) * 0x40;
    state.newFirmwareName = file.name;
    state.newFirmware     = fw;
    state.newName = file.name.replace(/\.bin$/i, '').replace(/[^\x00-\x7f]/g, '').substring(0, 13);
    state.sizeError = needed > state.selectedSlot.capacity;
    log(state.sizeError
      ? `ERROR: ${(needed/1024).toFixed(1)} KB > slot capacity ${(state.selectedSlot.capacity/1024).toFixed(1)} KB.`
      : `Selected "${file.name}" (${(fw.length/1024).toFixed(1)} KB) → fits in slot ${state.selectedSlot.index}.`);
  };
  input.click();
}

async function writeSlot() {
  if (!state.selectedSlot || !state.newFirmware) return;
  if (!appStore.connectState) { alert(sessionStorage.getItem('noticeConnectK5')); return; }
  if (appStore.configuration?.uart !== 'losehu') { log('ERROR: bootloader mode required.'); return; }

  state.writing = true;
  const slot   = state.selectedSlot;
  const fw: Uint8Array = state.newFirmware;
  const padded = new Uint8Array(Math.ceil(fw.length / 0x40) * 0x40).fill(0xFF);
  padded.set(fw, 0);
  const newEnd = slot.start + padded.length - 1;

  if (padded.length > slot.capacity) {
    log(`ERROR: ${(padded.length/1024).toFixed(1)} KB > capacity ${(slot.capacity/1024).toFixed(1)} KB.`);
    state.writing = false; return;
  }

  log(`--- Writing slot ${slot.index} ("${slot.name}") @ 0x${slot.start.toString(16).toUpperCase()} ---`);
  try {
    await eeprom_init(appStore.connectPort);

    // Write firmware binary
    for (let i = slot.start; i < slot.start + padded.length; i += 0x40) {
      const chunk = padded.slice(i - slot.start, i - slot.start + 0x40);
      await eeprom_write(appStore.connectPort, i, chunk, chunk.length, appStore.configuration?.uart);
      const verify = await eeprom_read(appStore.connectPort, i, chunk.length, appStore.configuration?.uart);
      if (!isEqual(chunk, verify)) { log(`Write error @ 0x${i.toString(16)}, retrying…`); i -= 0x40; continue; }
      log(`Writing… ${(((i - slot.start) / padded.length) * 100).toFixed(1)}%`);
    }

    // Update slot metadata entry
    const metaBase = 0x40020 + 32 * (slot.index - 1);
    const meta     = new Uint8Array(32).fill(0x00);
    meta.set(stringToUint8Array(state.newName.substring(0, 13)), 0);
    meta.set(hexReverseStringToUint8Array(slot.start.toString(16)), 16);
    meta.set(hexReverseStringToUint8Array(newEnd.toString(16)), 20);
    await eeprom_write(appStore.connectPort, metaBase,        meta.slice(0, 0x10), 0x10, appStore.configuration?.uart);
    await eeprom_write(appStore.connectPort, metaBase + 0x10, meta.slice(0x10),    0x10, appStore.configuration?.uart);
    log(`Metadata: "${state.newName}" @ 0x${slot.start.toString(16).toUpperCase()}–0x${newEnd.toString(16).toUpperCase()}`);

    // BUG FIX: blank config slot so bootloader skips stale-config restore on first boot
    if (slot.index <= PALETTE.length) await clearConfigSlot(slot.index);

    log('Done! Rebooting…');
    await eeprom_reboot(appStore.connectPort);
    state.selectedSlot = null; state.newFirmware = null;
    await readSlots();
  } catch (e: any) { log('Error: ' + errMsg(e)); }
  state.writing = false;
}
</script>

<script lang="ts">
export default { name: 'BL' };
</script>

<style scoped lang="less">
.container {
  padding: 0 20px 20px 20px;
}

/* ── EEPROM map ─────────────────────────────────────────────────────────────── */
.map-bar {
  display: flex;
  height: 52px;
  border-radius: 4px;
  overflow: hidden;
  border: 1px solid rgba(0, 0, 0, 0.12);
}

.map-seg {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 6px;
  overflow: hidden;
  transition: filter 0.12s;

  &.clickable { cursor: pointer; }
  &:hover     { filter: brightness(0.82); }
}

.seg-label {
  position: relative;
  z-index: 1;
  font-size: 10px;
  font-weight: 700;
  color: rgba(255, 255, 255, 0.93);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  padding: 0 4px;
  text-shadow: 0 1px 3px rgba(0, 0, 0, 0.55);
  pointer-events: none;
  letter-spacing: 0.02em;
}

.fw-fill {
  position: absolute;
  inset: 0 auto 0 0;
  opacity: 0.72;
  pointer-events: none;
}

.zone-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 5px;
}

.zone-badge {
  font-size: 10px;
  font-weight: 800;
  letter-spacing: 0.05em;
  padding: 2px 7px;
  border-radius: 3px;
  text-transform: uppercase;

  &.mgmt-badge { background: #FFF3E0; color: #BF360C; }
  &.fw-badge   { background: #E8F5E9; color: #1B5E20; }
}

.zone-addr {
  font-size: 11px;
  font-family: monospace;
  color: var(--color-text-3);
}

.map-info {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 6px;
  margin-top: 10px;
  padding: 8px 12px;
  background: var(--color-bg-2);
  border-left: 3px solid var(--color-border-2);
  border-radius: 2px;
  font-size: 0.83rem;
  min-height: 38px;
  transition: border-color 0.15s;

  .info-swatch {
    display: inline-block;
    width: 12px;
    height: 12px;
    border-radius: 2px;
    flex-shrink: 0;
  }
  .info-addr  { font-family: monospace; font-size: 0.8rem; color: var(--color-text-2); }
  .info-size  { color: var(--color-text-3); font-size: 0.82rem; }
  .info-sep   { color: var(--color-text-4); }
  .info-desc  { color: var(--color-text-2); flex: 1; min-width: 200px; }
  .info-placeholder { color: var(--color-text-3); font-size: 0.82rem; }
}

.map-legend {
  display: flex;
  flex-wrap: wrap;
  gap: 5px 14px;
  margin-top: 10px;

  .legend-item {
    display: flex;
    align-items: center;
    gap: 5px;
    font-size: 0.8rem;
    color: var(--color-text-2);
  }

  .legend-swatch {
    width: 12px;
    height: 12px;
    border-radius: 2px;
    flex-shrink: 0;
  }
}

/* ── Slot cards ──────────────────────────────────────────────────────────────── */
.slot-card {
  transition: border-color 0.3s, border-width 0.2s;
}

.slot-fill-track {
  height: 5px;
  background: var(--color-fill-2);
  border-radius: 3px;
  overflow: hidden;
}

.slot-fill-inner {
  height: 100%;
  border-radius: 3px;
  transition: width 0.3s;
}
</style>
