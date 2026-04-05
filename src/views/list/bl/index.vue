<template>
  <div class="container">
    <Breadcrumb :items="[$t('menu.list'), 'Multi-booting']" />

    <!-- ── EEPROM Map (only after read) ──────────────────────────────────────── -->
    <a-card v-if="phase === 'ready'" class="general-card" style="margin-bottom: 16px;">
      <template #title>
        EEPROM Map
        <span class="title-hint">512 KB · hover for details · click a slot to jump to it</span>
      </template>

      <div class="zone-header">
        <span class="zone-badge mgmt-badge">Management</span>
        <span class="zone-addr">0x00000 – 0x43FFF</span>
      </div>
      <div class="map-bar">
        <div
          v-for="(r, i) in mapMgmt"
          :key="'m'+i"
          class="map-seg"
          :style="{ flex: r.flex, background: r.color }"
          @mouseenter="hovered = r"
          @mouseleave="hovered = null"
        >
          <span v-if="r.flex >= 90" class="seg-label">{{ r.short }}</span>
        </div>
      </div>

      <div class="zone-header" style="margin-top:10px;">
        <span class="zone-badge fw-badge">Firmware Slots</span>
        <span class="zone-addr">0x44000 – 0x7FFFF</span>
      </div>
      <div class="map-bar">
        <div
          v-for="(r, i) in mapFw"
          :key="'f'+i"
          class="map-seg clickable"
          :style="{ flex: r.flex, background: r.bgColor }"
          @mouseenter="hovered = r"
          @mouseleave="hovered = null"
          @click="r.slotIdx != null && scrollToSlot(r.slotIdx)"
        >
          <div
            v-if="r.fillPct > 0"
            class="fw-fill"
            :style="{ width: r.fillPct + '%', background: r.color }"
          />
          <span class="seg-label">{{ r.short }}</span>
        </div>
      </div>

      <!-- Info box -->
      <div class="map-info" :style="hovered ? { borderLeftColor: hovered.color } : {}">
        <template v-if="hovered">
          <span class="info-swatch" :style="{ background: hovered.color }" />
          <strong>{{ hovered.name }}</strong>
          <code class="info-addr">0x{{ p5(hovered.start) }}–0x{{ p5(hovered.end) }}</code>
          <span class="info-size">{{ (hovered.size / 1024).toFixed(1) }} KB</span>
          <span class="info-sep">·</span>
          <span class="info-desc">{{ hovered.desc }}</span>
        </template>
        <span v-else class="info-placeholder">Hover a region for details · click a slot to jump to its card</span>
      </div>

      <!-- Legend -->
      <div class="map-legend">
        <div v-for="item in mapLegend" :key="item.label" class="legend-item">
          <div class="legend-swatch" :style="{ background: item.color }" />
          <span>{{ item.label }}</span>
        </div>
      </div>
    </a-card>

    <!-- ── Main card ─────────────────────────────────────────────────────────── -->
    <a-card class="general-card" :loading="loading">

      <!-- ── IDLE: read the radio ────────────────────────────────────────────── -->
      <template v-if="phase === 'idle'">
        <a-alert type="info" style="margin-bottom:16px;">
          Power off the radio → hold <strong>MENU</strong> → power on → connect here.
          The firmware version shown should be <strong>L_BL003</strong> (or similar).
          Do <em>not</em> use Quansheng programming mode (PTT + power on) —
          it doesn't support the 32-bit EEPROM addressing needed to read the slot table.
        </a-alert>
        <a-alert
          v-if="appStore.connectState && appStore.configuration?.uart !== 'losehu'"
          type="error"
          style="margin-bottom:16px;"
        >
          Connected as "{{ appStore.firmwareVersion || '?' }}" — this is a regular firmware, not the losehu bootloader.
          Please reconnect in bootloader mode (MENU + power on).
        </a-alert>
        <a-button type="primary" size="large" :loading="reading" @click="readDevice">
          <template #icon><icon-refresh /></template>
          Read device
        </a-button>
      </template>

      <!-- ── READY ───────────────────────────────────────────────────────────── -->
      <template v-if="phase === 'ready'">

        <!-- Top action bar -->
        <a-row style="margin-bottom:20px; align-items:center;" :gutter="12">
          <a-col :flex="'auto'">
            <span style="font-size:0.9rem; color:var(--color-text-2);">
              <strong>{{ device.slotCount }}</strong> firmware slot{{ device.slotCount !== 1 ? 's' : '' }} detected
              <span v-if="device.lastFw < 0xFF" style="margin-left:8px;">·
                last booted: slot {{ device.lastFw }}
              </span>
            </span>
          </a-col>
          <a-col :flex="'none'" style="display:flex; gap:8px; flex-wrap:wrap; justify-content:flex-end;">
            <a-button :loading="writing" @click="readDevice" :disabled="writing">
              <template #icon><icon-refresh /></template>
              Re-read
            </a-button>
            <a-button :loading="writing" @click="writeBootloaderOnly">
              <template #icon><icon-upload /></template>
              Flash bootloader only
            </a-button>
            <a-button
              :type="fullReconfig ? 'primary' : 'outline'"
              :status="fullReconfig ? 'warning' : 'normal'"
              @click="toggleReconfig"
              :disabled="writing"
            >
              <template #icon><icon-settings /></template>
              {{ fullReconfig ? 'Cancel reconfigure' : 'Full reconfigure' }}
            </a-button>
          </a-col>
        </a-row>

        <!-- ── Normal mode: replace individual slots ──────────────────────────── -->
        <div v-if="!fullReconfig">
          <a-row :gutter="[16, 16]">
            <a-col v-for="(slot, idx) in device.slots" :key="idx" :span="12">
              <a-card
                :id="'slot-card-'+idx"
                class="slot-card"
                :style="slotCardStyle(idx)"
              >
                <template #title>
                  <span :style="{ color: PALETTE[idx % PALETTE.length] }">
                    Slot {{ slot.index }}
                  </span>
                  <span class="slot-subtitle">{{ slot.name }}</span>
                </template>
                <template #extra>
                  <a-tag v-if="device.lastFw === slot.index" color="blue" size="small">last booted</a-tag>
                </template>

                <!-- Current firmware info -->
                <div class="slot-info">
                  <span class="info-row">
                    <code>0x{{ p5(slot.start) }}–0x{{ p5(slot.end) }}</code>
                    <span style="margin-left:8px; color:var(--color-text-3);">
                      {{ ((slot.end - slot.start + 1) / 1024).toFixed(1) }} KB used · capacity {{ (slot.capacity / 1024).toFixed(1) }} KB
                    </span>
                  </span>
                  <!-- Current fill bar -->
                  <div class="fill-track" style="margin-top:6px;">
                    <div class="fill-inner"
                      :style="{
                        width: Math.min((slot.end - slot.start + 1) / slot.capacity * 100, 100) + '%',
                        background: PALETTE[idx % PALETTE.length]
                      }"
                    />
                  </div>
                </div>

                <a-divider style="margin: 12px 0;" />

                <!-- Replacement firmware -->
                <div v-if="!replacements[idx]">
                  <a-button size="small" @click="pickReplacement(idx)">
                    <template #icon><icon-upload /></template>
                    Load replacement firmware
                  </a-button>
                </div>
                <div v-else>
                  <div style="display:flex; align-items:center; gap:8px; margin-bottom:8px; flex-wrap:wrap;">
                    <span style="font-size:0.85rem;">
                      <strong>→</strong> {{ replacements[idx].filename }}
                      ({{ (replacements[idx].firmware.length / 1024).toFixed(1) }} KB)
                    </span>
                    <a-tag :color="replacements[idx].oversize ? 'red' : 'green'" size="small">
                      {{ replacements[idx].oversize ? 'TOO LARGE' : 'fits' }}
                    </a-tag>
                    <a-button type="text" status="danger" size="mini" @click="clearReplacement(idx)">
                      <template #icon><icon-delete /></template>
                    </a-button>
                  </div>
                  <!-- New fill bar -->
                  <div class="fill-track" style="margin-bottom:10px;">
                    <div class="fill-inner"
                      :style="{
                        width: Math.min(Math.ceil(replacements[idx].firmware.length/0x40)*0x40 / slot.capacity * 100, 100) + '%',
                        background: replacements[idx].oversize ? '#f53f3f' : PALETTE[idx % PALETTE.length]
                      }"
                    />
                  </div>
                  <!-- Name override -->
                  <div style="display:flex; align-items:center; gap:8px; margin-bottom:10px;">
                    <span style="font-size:0.85rem; white-space:nowrap;">Slot name:</span>
                    <a-input
                      v-model="replacements[idx].name"
                      :max-length="13"
                      show-word-limit
                      size="small"
                      style="width:180px;"
                      @change="() => replacements[idx].name = replacements[idx].name.replace(/[^\x00-\x7f]/g,'')"
                    />
                  </div>
                  <a-button
                    v-if="!replacements[idx].oversize"
                    type="primary"
                    status="danger"
                    size="small"
                    :loading="writing"
                    @click="flashSingleSlot(idx)"
                  >
                    <template #icon><icon-thunderbolt /></template>
                    Flash slot {{ slot.index }}
                  </a-button>
                </div>
              </a-card>
            </a-col>
          </a-row>

          <!-- Flash all updated slots button -->
          <div v-if="anyReplacements" style="margin-top:16px; display:flex; justify-content:flex-end;">
            <a-button
              type="primary"
              status="danger"
              size="large"
              :loading="writing"
              @click="flashAllUpdated"
            >
              <template #icon><icon-thunderbolt /></template>
              Flash all updated slots ({{ updatedCount }})
            </a-button>
          </div>
        </div>

        <!-- ── Full reconfigure mode ─────────────────────────────────────────── -->
        <div v-else>
          <a-alert type="warning" style="margin-bottom:16px;">
            This will <strong>erase and rewrite</strong> all firmware slots, the bootloader, and all metadata.
            Config snapshots for replaced slots will be cleared so each firmware starts from its own defaults.
          </a-alert>

          <a-row style="margin-bottom:16px; align-items:center;">
            <a-col>
              <span style="margin-right:8px;">Number of slots:</span>
              <a-select v-model="rc.numSlots" style="width:80px;" @change="rebuildRcSlots">
                <a-option v-for="n in [1,2,3,4,5,6]" :key="n" :value="n">{{ n }}</a-option>
              </a-select>
              <span style="margin-left:12px; color:var(--color-text-3); font-size:0.85rem;">
                {{ rcSlotSizeKb }} KB/slot · first slot @ 0x44000
              </span>
            </a-col>
          </a-row>

          <a-row :gutter="[16,16]">
            <a-col v-for="(slot, idx) in rc.slots" :key="idx" :span="12">
              <a-card class="slot-card" :style="slot.firmware ? { borderColor: PALETTE[idx % PALETTE.length] } : {}">
                <template #title>
                  <span :style="{ color: PALETTE[idx % PALETTE.length] }">Slot {{ slot.index }}</span>
                  <span v-if="slot.firmware" class="slot-subtitle">
                    {{ (slot.firmware.length / 1024).toFixed(1) }} / {{ rcSlotSizeKb }} KB
                  </span>
                </template>
                <template #extra>
                  <div v-if="slot.firmware" :style="{ width:'12px', height:'12px', borderRadius:'2px', background: PALETTE[idx%PALETTE.length] }" />
                </template>

                <div v-if="slot.firmware" class="fill-track" style="margin-bottom:10px;">
                  <div class="fill-inner"
                    :style="{
                      width: Math.min(Math.ceil(slot.firmware.length/0x40)*0x40 / rcSlotSize * 100, 100) + '%',
                      background: slot.oversize ? '#f53f3f' : PALETTE[idx % PALETTE.length]
                    }"
                  />
                </div>

                <a-form-item label="Name" style="margin-bottom:10px;">
                  <a-input
                    v-model="slot.name"
                    :max-length="13"
                    show-word-limit
                    :placeholder="'Slot ' + slot.index"
                    @change="() => slot.name = slot.name.replace(/[^\x00-\x7f]/g,'')"
                  />
                </a-form-item>
                <a-space>
                  <a-button @click="selectRcFile(idx)">
                    <template #icon><icon-upload /></template>
                    {{ slot.firmware ? slot.filename : 'Select .bin file' }}
                  </a-button>
                  <a-tag v-if="slot.firmware && slot.oversize" color="red">TOO LARGE</a-tag>
                  <a-tag v-else-if="slot.firmware" color="green">OK</a-tag>
                  <a-button v-if="slot.firmware" type="text" status="danger" size="small" @click="clearRcSlot(idx)">
                    <template #icon><icon-delete /></template>
                  </a-button>
                </a-space>
              </a-card>
            </a-col>
          </a-row>

          <div style="margin-top:16px; display:flex; justify-content:flex-end;">
            <a-button
              type="primary"
              status="danger"
              size="large"
              :loading="writing"
              :disabled="!rc.slots.some((s:any) => s.firmware && !s.oversize)"
              @click="writeAll"
            >
              <template #icon><icon-thunderbolt /></template>
              Write everything to device
            </a-button>
          </div>
        </div>

      </template>
    </a-card>

    <!-- ── Status log ─────────────────────────────────────────────────────────── -->
    <div
      id="statusArea"
      v-if="state.status"
      style="height:14em; background-color:var(--color-bg-3); color:var(--color-text-3);
             overflow:auto; padding:14px; margin-top:12px;
             font-family:monospace; font-size:0.82rem; border-radius:4px;"
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

// ── Constants ────────────────────────────────────────────────────────────────
const EEPROM_SIZE   = 0x80000;
const FW_BASE       = 0x44000;
const BL_ADDR       = 0x41000;
const META_BASE     = 0x40000;
const CFG_SLOT_BASE = 0x20000;
const CFG_SLOT_SIZE = 0x4000;   // must match CONFIG_SIZE in bootloader eeprom.c

const PALETTE = ['#4CAF50','#7C4DFF','#F44336','#00BCD4','#FF9800','#E91E63'];
const C_LIVE  = '#1565C0';
const C_FREE  = '#BDBDBD';
const C_META  = '#E65100';
const C_BL    = '#B71C1C';

// ── State ────────────────────────────────────────────────────────────────────
type Phase = 'idle' | 'ready';

const phase       = ref<Phase>('idle');
const reading     = ref(false);
const writing     = ref(false);
const fullReconfig= ref(false);
const hovered     = ref<any>(null);
const highlight   = ref(-1);

const device = reactive({
  slotCount: 0,
  lastFw:    0xFF as number,
  slots:     [] as any[],   // [{index, name, start, end, capacity}]
});

// Per-slot replacements (normal mode): idx → {firmware, filename, name, oversize}
const replacements = reactive<Record<number, any>>({});

// Full reconfigure state
const rc = reactive({ numSlots: 4, slots: [] as any[] });

// Bootloader binary (always loaded from /L_BL003.bin)
const bl = ref<Uint8Array | null>(null);
const blName = ref('L_BL003');

const state = reactive({ status: '' });

// ── Helpers ──────────────────────────────────────────────────────────────────
function p5(n: number) { return n.toString(16).toUpperCase().padStart(5, '0'); }

function log(msg: string) {
  state.status += msg + '<br/>';
  nextTick(() => { const el = document.getElementById('statusArea'); if (el) el.scrollTop = el.scrollHeight; });
}

function errMsg(e: any): string {
  if (e instanceof Error) return e.message;
  if (typeof e === 'string') return e;
  return JSON.stringify(e);
}

function slotColor(i: number) { return PALETTE[i % PALETTE.length]; }

function lighten(hex: string, t = 0.60): string {
  const r = parseInt(hex.slice(1,3), 16), g = parseInt(hex.slice(3,5), 16), b = parseInt(hex.slice(5,7), 16);
  return '#' + [r,g,b].map(c => Math.round(c + (255 - c) * t).toString(16).padStart(2,'0')).join('');
}

function isEqual(a: Uint8Array, b: Uint8Array) {
  return a.length === b.length && a.every((v, i) => v === b[i]);
}

// ── Bootloader load ───────────────────────────────────────────────────────────
async function loadBL() {
  try {
    const res = await fetch('/L_BL003.bin');
    if (!res.body) return;
    const reader = res.body.getReader();
    const chunks: number[] = [];
    while (true) { const { done, value } = await reader.read(); if (done) break; chunks.push(...value); }
    const buf = new Uint8Array(0x3000); buf.set(chunks, 0); bl.value = buf;
    blName.value = 'L_BL003';
  } catch { /* silent */ }
}

onMounted(loadBL);

// ── EEPROM map regions ────────────────────────────────────────────────────────
function seg(name: string, short: string, start: number, size: number,
             color: string, desc: string, extra: any = {}): any {
  return { name, short, start, size, end: start + size - 1,
           flex: Math.max(Math.round(size / 64), 4), color, bgColor: color, desc,
           fillPct: 0, slotIdx: null, ...extra };
}

const mapMgmt = computed<any[]>(() => {
  const n  = device.slotCount || 0;
  const rs: any[] = [];
  rs.push(seg('Live Config (0x0000–0x3FFF)', 'Config', 0x00000, 0x4000, C_LIVE,
    'Active radio config: channels, settings, calibration, and firmware-specific data. ' +
    'The bootloader saves this entire 16 KB region before every firmware switch and restores the target slot\'s snapshot on boot.'));
  rs.push(seg('Free', '', 0x04000, 0x1C000, C_FREE, 'Unused free zone — 112 KB.'));
  for (let i = 0; i < n; i++) {
    rs.push(seg(`Config Slot ${i+1}`, `Cfg${i+1}`,
      CFG_SLOT_BASE + i * CFG_SLOT_SIZE, CFG_SLOT_SIZE, lighten(slotColor(i)),
      `Config snapshot for firmware slot ${i+1}. Saved here before switching away, restored when switching back. ` +
      `Blanked (all 0xFF) when a new firmware is written to slot ${i+1} so the first boot gets clean defaults.`));
  }
  const cfgEnd = CFG_SLOT_BASE + n * CFG_SLOT_SIZE;
  if (cfgEnd < META_BASE)
    rs.push(seg('Free', '', cfgEnd, META_BASE - cfgEnd, C_FREE,
      `Free zone — ${((META_BASE - cfgEnd) / 1024).toFixed(0)} KB.`));
  rs.push(seg('Metadata', 'Meta', META_BASE, 0x1000, C_META,
    '0x40000: slot count (1 B) · 0x40001: boot mode flag · 0x40002: last-run slot index · ' +
    '0x40008: bootloader name (8 B) · 0x40020+: slot table (32 B/slot: 16 B name | 4 B start LE | 4 B end LE | 8 B padding)'));
  rs.push(seg('Bootloader B', 'BL-B', BL_ADDR, 0x3000, C_BL,
    'Losehu Bootloader B (L_BL003.bin, ~11 KB). Executed by Bootloader A in MCU flash when radio powers on with MENU held. ' +
    'Handles firmware selection, config save/restore, and UART flashing.'));
  return rs;
});

const mapFw = computed<any[]>(() => {
  const rs: any[] = [];
  const slots = device.slots;
  for (let i = 0; i < slots.length; i++) {
    const s   = slots[i];
    const cap = s.capacity;
    const used = s.end - s.start + 1;
    const repl = replacements[i];
    const fillSz = repl ? Math.ceil(repl.firmware.length / 0x40) * 0x40 : used;
    const col = slotColor(i);
    rs.push({
      name:     `FW Slot ${s.index}${s.name ? ' · ' + s.name : ''}${repl ? ' [→ ' + repl.filename + ']' : ''}`,
      short:    s.name || `Slot ${s.index}`,
      start:    s.start, end: s.start + cap - 1, size: cap,
      flex:     Math.max(Math.round(cap / 64), 8),
      color:    col, bgColor: lighten(col, 0.68),
      desc:     `Slot ${s.index} "${s.name}" — ${(used/1024).toFixed(1)} KB used of ${(cap/1024).toFixed(1)} KB capacity. ` +
                `Range 0x${p5(s.start)}–0x${p5(s.start + cap - 1)}. Config snapshot @ 0x${(CFG_SLOT_BASE+(i)*CFG_SLOT_SIZE).toString(16).toUpperCase()}. ` +
                (repl ? `Replacement loaded: ${repl.filename} (${(repl.firmware.length/1024).toFixed(1)} KB).` : 'Click to jump to slot card.'),
      fillPct:  Math.min(fillSz / cap * 100, 100),
      slotIdx:  i,
    });
  }
  const tailStart = slots.length > 0 ? slots[slots.length-1].start + slots[slots.length-1].capacity : FW_BASE;
  if (tailStart < EEPROM_SIZE)
    rs.push(seg('Free', 'Free', tailStart, EEPROM_SIZE - tailStart, C_FREE,
      `Unused EEPROM — ${((EEPROM_SIZE - tailStart)/1024).toFixed(1)} KB.`));
  return rs;
});

const mapLegend = computed(() => {
  const items: { label: string; color: string }[] = [
    { label: 'Live Config',   color: C_LIVE },
    { label: 'Free / Unused', color: C_FREE },
  ];
  for (let i = 0; i < device.slotCount; i++)
    items.push({ label: `Config Slot ${i+1}`, color: lighten(slotColor(i)) });
  items.push({ label: 'Metadata', color: C_META }, { label: 'Bootloader B', color: C_BL });
  for (let i = 0; i < device.slots.length; i++)
    items.push({ label: `FW Slot ${i+1}${device.slots[i].name ? ' · '+device.slots[i].name : ''}`, color: slotColor(i) });
  return items;
});

// ── Read device ───────────────────────────────────────────────────────────────
async function readBytes(addr: number, size: number): Promise<Uint8Array> {
  const out = new Uint8Array(size);
  for (let off = 0; off < size;) {
    const n = Math.min(0x40, size - off);
    const d = await eeprom_read(appStore.connectPort, addr + off, n, appStore.configuration?.uart);
    out.set(d.slice(0, n), off); off += n;
  }
  return out;
}

function u32le(buf: Uint8Array, off: number) {
  return (buf[off] | buf[off+1]<<8 | buf[off+2]<<16 | buf[off+3]<<24) >>> 0;
}

async function readDevice() {
  if (!appStore.connectState) { alert(sessionStorage.getItem('noticeConnectK5')); return; }
  if (appStore.configuration?.uart !== 'losehu') {
    log('ERROR: requires losehu bootloader mode — MENU + power on, then reconnect.'); return;
  }
  reading.value = true;
  setLoading(true);
  state.status = '';
  // Reset state
  Object.assign(device, { slotCount: 0, lastFw: 0xFF, slots: [] });
  Object.keys(replacements).forEach(k => delete replacements[+k]);
  fullReconfig.value = false;

  try {
    await eeprom_init(appStore.connectPort);

    const countBuf = await readBytes(0x40000, 1);
    const count = countBuf[0];
    if (!count || count > 64) {
      log('No valid multiboot metadata found. Use "Full reconfigure" to set up the device for the first time.');
      phase.value = 'ready';
      device.slotCount = 0;
      fullReconfig.value = true;
      rebuildRcSlots();
      return;
    }

    const lastFwBuf = await readBytes(0x40002, 1);
    device.lastFw = lastFwBuf[0];

    const table = await readBytes(0x40020, count * 32);
    const raw = Array.from({ length: count }, (_, i) => {
      const b = i * 32;
      let name = '';
      for (let j = 0; j < 13 && table[b+j]; j++) name += String.fromCharCode(table[b+j]);
      return { name: name || `Slot ${i+1}`, start: u32le(table, b+16), end: u32le(table, b+20) };
    });

    device.slots = raw.map((r, i) => {
      const nextStart = i+1 < raw.length ? raw[i+1].start : EEPROM_SIZE;
      return {
        index: i+1, name: r.name, start: r.start, end: r.end,
        capacity: nextStart - r.start,
        startHex: '0x' + p5(r.start), endHex: '0x' + p5(r.end),
      };
    });
    device.slotCount = count;

    log(`Read OK: ${count} slot(s), last booted slot ${device.lastFw < 0xFF ? device.lastFw : '(none)'}.`);
    device.slots.forEach((s: any) =>
      log(`  Slot ${s.index}: "${s.name}"  ${s.startHex}–${s.endHex}  cap ${(s.capacity/1024).toFixed(1)} KB`));

    phase.value = 'ready';
  } catch (e: any) {
    log('Error reading device: ' + errMsg(e));
  }
  reading.value = false;
  setLoading(false);
}

// ── Map click ─────────────────────────────────────────────────────────────────
function scrollToSlot(idx: number) {
  if (fullReconfig.value) return;
  highlight.value = idx;
  nextTick(() => {
    document.getElementById('slot-card-'+idx)?.scrollIntoView({ behavior: 'smooth', block: 'center' });
    setTimeout(() => { highlight.value = -1; }, 2500);
  });
}

function slotCardStyle(idx: number): Record<string, string> {
  if (highlight.value === idx) return { borderColor: '#1890ff', borderWidth: '2px' };
  if (replacements[idx]) return { borderColor: PALETTE[idx % PALETTE.length] };
  return {};
}

// ── Normal mode: individual slot replacement ──────────────────────────────────
const anyReplacements = computed(() =>
  Object.values(replacements).some((r: any) => r && !r.oversize));
const updatedCount = computed(() =>
  Object.values(replacements).filter((r: any) => r && !r.oversize).length);

function pickReplacement(idx: number) {
  const slot = device.slots[idx];
  const input = document.createElement('input');
  input.type = 'file'; input.accept = '.bin';
  input.onchange = async () => {
    if (!input.files?.length) return;
    const file = input.files[0];
    const fw   = unpack(new Uint8Array(await file.arrayBuffer()));
    const padded = Math.ceil(fw.length / 0x40) * 0x40;
    replacements[idx] = {
      firmware: fw, filename: file.name, oversize: padded > slot.capacity,
      name: file.name.replace(/\.bin$/i,'').replace(/[^\x00-\x7f]/g,'').substring(0,13),
    };
  };
  input.click();
}

function clearReplacement(idx: number) { delete replacements[idx]; }

async function flashSingleSlot(idx: number) {
  const slot = device.slots[idx];
  const repl = replacements[idx];
  if (!slot || !repl || repl.oversize) return;
  if (!appStore.connectState) { alert(sessionStorage.getItem('noticeConnectK5')); return; }
  if (appStore.configuration?.uart !== 'losehu') { log('ERROR: bootloader mode required.'); return; }

  writing.value = true;
  const fw     = repl.firmware as Uint8Array;
  const padded = new Uint8Array(Math.ceil(fw.length / 0x40) * 0x40).fill(0xFF);
  padded.set(fw, 0);
  const newEnd = slot.start + padded.length - 1;

  log(`--- Flash slot ${slot.index} "${slot.name}" → "${repl.name}" ---`);
  try {
    await eeprom_init(appStore.connectPort);

    for (let i = slot.start; i < slot.start + padded.length; i += 0x40) {
      const chunk  = padded.slice(i - slot.start, i - slot.start + 0x40);
      await eeprom_write(appStore.connectPort, i, chunk, chunk.length, appStore.configuration?.uart);
      const verify = await eeprom_read(appStore.connectPort, i, chunk.length, appStore.configuration?.uart);
      if (!isEqual(chunk, verify)) { log(`Write error @ 0x${i.toString(16)}, retrying…`); i -= 0x40; continue; }
      log(`Writing… ${(((i-slot.start)/padded.length)*100).toFixed(1)}%`);
    }

    // Update metadata entry for this slot
    const metaBase = 0x40020 + 32 * (slot.index - 1);
    const meta = new Uint8Array(32).fill(0x00);
    meta.set(stringToUint8Array(repl.name.substring(0, 13)), 0);
    meta.set(hexReverseStringToUint8Array(slot.start.toString(16)), 16);
    meta.set(hexReverseStringToUint8Array(newEnd.toString(16)), 20);
    await eeprom_write(appStore.connectPort, metaBase,        meta.slice(0, 0x10), 0x10, appStore.configuration?.uart);
    await eeprom_write(appStore.connectPort, metaBase + 0x10, meta.slice(0x10),    0x10, appStore.configuration?.uart);
    log(`Metadata updated: "${repl.name}" @ 0x${p5(slot.start)}–0x${p5(newEnd)}`);

    // Clear config slot so first boot uses firmware defaults, not stale snapshot
    await clearConfigSlot(slot.index);

    log('Done! Rebooting…');
    await eeprom_reboot(appStore.connectPort);
    clearReplacement(idx);
    await readDevice();
  } catch (e: any) { log('Error: ' + errMsg(e)); }
  writing.value = false;
}

async function flashAllUpdated() {
  const toFlash = Object.entries(replacements)
    .filter(([, r]: any) => r && !r.oversize)
    .map(([idx, r]) => ({ idx: +idx, repl: r }));
  if (!toFlash.length) return;

  writing.value = true;
  try {
    await eeprom_init(appStore.connectPort);
    for (const { idx, repl } of toFlash) {
      const slot = device.slots[idx];
      const fw   = repl.firmware as Uint8Array;
      const padded = new Uint8Array(Math.ceil(fw.length / 0x40) * 0x40).fill(0xFF);
      padded.set(fw, 0);
      const newEnd = slot.start + padded.length - 1;

      log(`--- Slot ${slot.index} "${slot.name}" → "${repl.name}" ---`);
      for (let i = slot.start; i < slot.start + padded.length; i += 0x40) {
        const chunk  = padded.slice(i - slot.start, i - slot.start + 0x40);
        await eeprom_write(appStore.connectPort, i, chunk, chunk.length, appStore.configuration?.uart);
        const verify = await eeprom_read(appStore.connectPort, i, chunk.length, appStore.configuration?.uart);
        if (!isEqual(chunk, verify)) { log(`Write error @ 0x${i.toString(16)}, retrying…`); i -= 0x40; continue; }
        log(`Slot ${slot.index} ${(((i-slot.start)/padded.length)*100).toFixed(1)}%`);
      }
      const metaBase = 0x40020 + 32 * (slot.index - 1);
      const meta = new Uint8Array(32).fill(0x00);
      meta.set(stringToUint8Array(repl.name.substring(0, 13)), 0);
      meta.set(hexReverseStringToUint8Array(slot.start.toString(16)), 16);
      meta.set(hexReverseStringToUint8Array(newEnd.toString(16)), 20);
      await eeprom_write(appStore.connectPort, metaBase,        meta.slice(0, 0x10), 0x10, appStore.configuration?.uart);
      await eeprom_write(appStore.connectPort, metaBase + 0x10, meta.slice(0x10),    0x10, appStore.configuration?.uart);
      await clearConfigSlot(slot.index);
    }
    log('All slots written. Rebooting…');
    await eeprom_reboot(appStore.connectPort);
    Object.keys(replacements).forEach(k => delete replacements[+k]);
    await readDevice();
  } catch (e: any) { log('Error: ' + errMsg(e)); }
  writing.value = false;
}

// ── Config slot clear (bug fix) ───────────────────────────────────────────────
async function clearConfigSlot(slotIndex: number) {
  const addr  = CFG_SLOT_BASE + (slotIndex - 1) * CFG_SLOT_SIZE;
  const blank = new Uint8Array(0x40).fill(0xFF);
  log(`Clearing config slot ${slotIndex} @ 0x${addr.toString(16).toUpperCase()}…`);
  for (let i = addr; i < addr + CFG_SLOT_SIZE; i += 0x40)
    await eeprom_write(appStore.connectPort, i, blank, 0x40, appStore.configuration?.uart);
}

// ── Bootloader only ───────────────────────────────────────────────────────────
async function writeBootloaderOnly() {
  if (!appStore.connectState) { alert(sessionStorage.getItem('noticeConnectK5')); return; }
  if (!bl.value) { log('Bootloader binary not loaded — check /L_BL003.bin is served.'); return; }
  writing.value = true; state.status = '';
  try {
    await eeprom_init(appStore.connectPort);
    log('--- Writing bootloader B ---');
    await writeRange(BL_ADDR, bl.value, 'Bootloader');
    log('Done! Rebooting…');
    await eeprom_reboot(appStore.connectPort);
  } catch (e: any) { log('Error: ' + errMsg(e)); }
  writing.value = false;
}

// ── Full reconfigure mode ─────────────────────────────────────────────────────
function toggleReconfig() {
  fullReconfig.value = !fullReconfig.value;
  if (fullReconfig.value) {
    rc.numSlots = Math.max(device.slotCount, 4);
    rebuildRcSlots();
  }
}

const rcSlotSize = computed(() => {
  const raw = Math.floor((EEPROM_SIZE - FW_BASE) / rc.numSlots);
  return Math.floor(raw / 0x40) * 0x40;
});
const rcSlotSizeKb = computed(() => (rcSlotSize.value / 1024).toFixed(1));

function rcSlotStart(i: number) { return FW_BASE + i * rcSlotSize.value; }

function rebuildRcSlots() {
  const prev = rc.slots.slice();
  rc.slots = Array.from({ length: rc.numSlots }, (_, i) => {
    const s: any = {
      index: i+1, start: rcSlotStart(i), end: rcSlotStart(i) + rcSlotSize.value - 1,
      name: '', filename: '', firmware: null, oversize: false,
    };
    if (prev[i]) {
      Object.assign(s, { name: prev[i].name, filename: prev[i].filename, firmware: prev[i].firmware });
      if (s.firmware) s.oversize = Math.ceil(s.firmware.length / 0x40) * 0x40 > rcSlotSize.value;
    }
    return s;
  });
}

function clearRcSlot(idx: number) {
  Object.assign(rc.slots[idx], { firmware: null, filename: '', oversize: false });
}

function selectRcFile(idx: number) {
  const input = document.createElement('input');
  input.type = 'file'; input.accept = '.bin';
  input.onchange = async () => {
    if (!input.files?.length) return;
    const file = input.files[0];
    const fw   = unpack(new Uint8Array(await file.arrayBuffer()));
    const slot = rc.slots[idx];
    slot.filename = file.name;
    slot.firmware = fw;
    slot.oversize = Math.ceil(fw.length / 0x40) * 0x40 > rcSlotSize.value;
    if (!slot.name) slot.name = file.name.replace(/\.bin$/i,'').replace(/[^\x00-\x7f]/g,'').substring(0,13);
  };
  input.click();
}

// ── Write all (full reconfigure) ──────────────────────────────────────────────
async function writeRange(start: number, data: Uint8Array | number[], label: string) {
  const arr = data instanceof Uint8Array ? data : new Uint8Array(data);
  for (let i = start; i < start + arr.length; i += 0x40) {
    const chunk  = arr.slice(i - start, i - start + 0x40);
    await eeprom_write(appStore.connectPort, i, chunk, chunk.length, appStore.configuration?.uart);
    const verify = await eeprom_read(appStore.connectPort, i, chunk.length, appStore.configuration?.uart);
    if (!isEqual(chunk, verify)) { log(`${label} write error @ 0x${i.toString(16)}, retrying…`); i -= 0x40; continue; }
    log(`${label} ${(((i-start)/arr.length)*100).toFixed(1)}%`);
  }
}

async function writeAll() {
  if (!appStore.connectState) { alert(sessionStorage.getItem('noticeConnectK5')); return; }
  if (!bl.value) { log('Bootloader binary not loaded.'); return; }
  const filled = rc.slots.filter((s: any) => s.firmware && !s.oversize);
  if (!filled.length) { log('No valid firmware assigned to any slot.'); return; }

  writing.value = true; state.status = '';
  try {
    const sz = await check_eeprom(appStore.connectPort, appStore.configuration?.uart);
    if (sz < 0x80000) { alert('Only 4Mbit (512 KB) EEPROM supported.'); writing.value = false; return; }
    await eeprom_init(appStore.connectPort);

    log('--- Writing bootloader B ---');
    await writeRange(BL_ADDR, bl.value, 'Bootloader');

    // Reset last-FW index so bootloader starts clean
    await eeprom_write(appStore.connectPort, 0x40002, new Uint8Array([0xFF]), 1, appStore.configuration?.uart);

    // Metadata
    await writeRange(META_BASE, new Uint8Array([filled.length]), 'FW count');
    const blNameArr = new Uint8Array(8);
    blNameArr.set(stringToUint8Array(blName.value.substring(0, 8)));
    await writeRange(0x40008, blNameArr, 'BL name');

    const metaBuf: number[] = [];
    for (const slot of filled) {
      const pad = new Uint8Array(Math.ceil(slot.firmware.length / 0x40) * 0x40).fill(0xFF);
      pad.set(slot.firmware, 0); slot._padded = pad;
      const nameArr = new Uint8Array(16);
      nameArr.set(stringToUint8Array((slot.name || `Slot${slot.index}`).substring(0, 13)));
      const startArr = new Uint8Array(4);
      startArr.set(hexReverseStringToUint8Array(slot.start.toString(16)));
      const endArr = new Uint8Array(4);
      endArr.set(hexReverseStringToUint8Array((slot.start + pad.length - 1).toString(16)));
      metaBuf.push(...nameArr, ...startArr, ...endArr, ...new Uint8Array(8));
    }
    log('--- Writing slot metadata ---');
    await writeRange(0x40020, new Uint8Array(metaBuf), 'Metadata');

    for (const slot of filled) {
      log(`--- Slot ${slot.index} "${slot.name}" @ 0x${slot.start.toString(16).toUpperCase()} ---`);
      await writeRange(slot.start, slot._padded, slot.name || `Slot${slot.index}`);
      await clearConfigSlot(slot.index);
    }

    log('--- Done! Rebooting… ---');
    await eeprom_reboot(appStore.connectPort);
    fullReconfig.value = false;
    await readDevice();
  } catch (e: any) { log('Error: ' + errMsg(e)); }
  writing.value = false;
}
</script>

<script lang="ts">
export default { name: 'BL' };
</script>

<style scoped lang="less">
.container { padding: 0 20px 20px 20px; }

.title-hint {
  font-weight: normal;
  font-size: 0.82rem;
  margin-left: 8px;
  color: var(--color-text-3);
}

/* Map */
.map-bar {
  display: flex;
  height: 52px;
  border-radius: 4px;
  overflow: hidden;
  border: 1px solid rgba(0,0,0,0.12);
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
  position: relative; z-index: 1;
  font-size: 10px; font-weight: 700;
  color: rgba(255,255,255,0.93);
  white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
  padding: 0 4px;
  text-shadow: 0 1px 3px rgba(0,0,0,0.55);
  pointer-events: none; letter-spacing: 0.02em;
}

.fw-fill {
  position: absolute; inset: 0 auto 0 0;
  opacity: 0.72; pointer-events: none;
}

.zone-header {
  display: flex; align-items: center; gap: 8px; margin-bottom: 5px;
}

.zone-badge {
  font-size: 10px; font-weight: 800; letter-spacing: 0.05em;
  padding: 2px 7px; border-radius: 3px; text-transform: uppercase;
  &.mgmt-badge { background: #FFF3E0; color: #BF360C; }
  &.fw-badge   { background: #E8F5E9; color: #1B5E20; }
}

.zone-addr {
  font-size: 11px; font-family: monospace; color: var(--color-text-3);
}

.map-info {
  display: flex; align-items: center; flex-wrap: wrap; gap: 6px;
  margin-top: 10px; padding: 8px 12px;
  background: var(--color-bg-2);
  border-left: 3px solid var(--color-border-2);
  border-radius: 2px; font-size: 0.83rem; min-height: 38px;
  transition: border-color 0.15s;
  .info-swatch { display: inline-block; width: 12px; height: 12px; border-radius: 2px; flex-shrink: 0; }
  .info-addr   { font-family: monospace; font-size: 0.8rem; color: var(--color-text-2); }
  .info-size   { color: var(--color-text-3); font-size: 0.82rem; }
  .info-sep    { color: var(--color-text-4); }
  .info-desc   { color: var(--color-text-2); flex: 1; min-width: 200px; }
  .info-placeholder { color: var(--color-text-3); font-size: 0.82rem; }
}

.map-legend {
  display: flex; flex-wrap: wrap; gap: 5px 14px; margin-top: 10px;
  .legend-item { display: flex; align-items: center; gap: 5px; font-size: 0.8rem; color: var(--color-text-2); }
  .legend-swatch { width: 12px; height: 12px; border-radius: 2px; flex-shrink: 0; }
}

/* Slot cards */
.slot-card { transition: border-color 0.25s, border-width 0.15s; }

.slot-subtitle {
  font-weight: normal; font-size: 0.85rem;
  margin-left: 8px; color: var(--color-text-3);
}

.slot-info { font-size: 0.84rem; }
.info-row  { display: flex; align-items: center; flex-wrap: wrap; }

.fill-track {
  height: 6px; background: var(--color-fill-2);
  border-radius: 3px; overflow: hidden;
}
.fill-inner {
  height: 100%; border-radius: 3px; transition: width 0.3s;
}
</style>
