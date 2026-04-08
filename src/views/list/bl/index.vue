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
        <span class="zone-badge mgmt-badge">Management{{ device.slots.some((s:any) => s.start < FW_BASE) ? ' · Zone A FW' : '' }}</span>
        <span class="zone-addr">0x00000 – 0x43FFF</span>
      </div>
      <div class="map-bar">
        <div
          v-for="(r, i) in mapMgmt"
          :key="'m'+i"
          :class="['map-seg', r.slotIdx != null ? 'clickable' : '']"
          :style="{ flex: r.flex, background: r.color }"
          @mouseenter="hovered = r"
          @mouseleave="hovered = null"
          @click="r.slotIdx != null && scrollToSlot(r.slotIdx)"
        >
          <div v-if="r.fillPct > 0" class="fw-fill" :style="{ width: r.fillPct + '%', background: r.color }" />
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
          The firmware version shown should be <strong>L_BL005</strong> (or similar).
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
            <a-button :loading="writing" @click="writeBootloaderOnly" :disabled="!bl">
              <template #icon><icon-upload /></template>
              Flash bootloader only
            </a-button>
            <a-button @click="loadCustomBL" :disabled="writing">
              <template #icon><icon-file /></template>
              Load custom BL
            </a-button>
            <span v-if="blName" style="font-size:0.82rem; color:var(--color-text-3); align-self:center;">
              <a-tag :color="blIsCustom ? 'orange' : 'blue'" size="small">{{ blName }}</a-tag>
            </span>
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

        <!-- ── Calibration restore ───────────────────────────────────────────── -->
        <a-row style="margin-bottom:16px; align-items:center; padding:10px 12px; background:var(--color-bg-2); border-radius:4px; border:1px solid var(--color-border-2);" :gutter="12">
          <a-col :flex="'auto'" style="display:flex; align-items:center; gap:10px;">
            <span style="font-weight:500;">Calibration</span>
            <a-tag v-if="calibFile" color="orange" size="small">{{ calibFile.name }}</a-tag>
            <a-tag v-else color="gray" size="small">using live device calibration</a-tag>
            <span style="font-size:0.82rem; color:var(--color-text-3);">injected into all config slots on connect and flash</span>
          </a-col>
          <a-col :flex="'none'" style="display:flex; gap:8px;">
            <a-button size="small" :disabled="writing" @click="loadCalibFile">
              <template #icon><icon-upload /></template>
              Upload .bin
            </a-button>
            <a-button v-if="calibFile" size="small" :disabled="writing" @click="calibFile = null">
              <template #icon><icon-close /></template>
              Clear
            </a-button>
            <a-button size="small" :disabled="writing || !appStore.connectState" @click="applyCalibNow">
              <template #icon><icon-save /></template>
              Apply to all slots
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
                <template v-if="rcZoneACount > 0">
                  {{ rcZoneACount }}×{{ (ZONE_A_FW_SLOT_SIZE/1024).toFixed(0) }} KB @ 0x{{ ZONE_A_FW_BASE.toString(16).toUpperCase() }} &nbsp;+&nbsp; {{ rcZoneBCount }}×{{ rcSlotSizeKb }} KB @ 0x44000
                </template>
                <template v-else>
                  {{ rc.numSlots }}×{{ rcSlotSizeKb }} KB · first slot @ 0x44000
                </template>
              </span>
            </a-col>
          </a-row>

          <a-row :gutter="[16,16]">
            <a-col v-for="(slot, idx) in rc.slots" :key="idx" :span="12">
              <a-card class="slot-card" :style="slot.firmware ? { borderColor: PALETTE[idx % PALETTE.length] } : {}">
                <template #title>
                  <span :style="{ color: PALETTE[idx % PALETTE.length] }">
                    Slot {{ slot.index }}
                    <span v-if="slot.zoneA" style="font-size:0.75rem; font-weight:normal; margin-left:4px; color:var(--color-text-3);">Zone A</span>
                  </span>
                  <span v-if="slot.firmware" class="slot-subtitle">
                    {{ (slot.firmware.length / 1024).toFixed(1) }} / {{ (slot.capacity / 1024).toFixed(1) }} KB
                  </span>
                </template>
                <template #extra>
                  <div v-if="slot.firmware" :style="{ width:'12px', height:'12px', borderRadius:'2px', background: PALETTE[idx%PALETTE.length] }" />
                </template>

                <div v-if="slot.firmware" class="fill-track" style="margin-bottom:10px;">
                  <div class="fill-inner"
                    :style="{
                      width: Math.min(Math.ceil(slot.firmware.length/0x40)*0x40 / slot.capacity * 100, 100) + '%',
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

    <!-- ── Progress / status ─────────────────────────────────────────────────── -->
    <div v-if="prog.visible || statusMsg" class="progress-area">
      <div v-if="statusMsg" class="status-msg" :class="statusMsgType">{{ statusMsg }}</div>
      <div v-if="prog.visible" class="prog-row">
        <div class="prog-label">
          <span class="prog-name">{{ prog.label }}</span>
          <span class="prog-pct">{{ prog.pct.toFixed(0) }}%</span>
        </div>
        <div class="prog-track">
          <div class="prog-fill" :style="{ width: prog.pct + '%', background: prog.color }" />
        </div>
      </div>
    </div>
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
const EEPROM_SIZE        = 0x80000;
const FW_BASE            = 0x44000;
const BL_ADDR            = 0x41000;
const META_BASE          = 0x40000;
const CFG_SLOT_BASE      = 0x10000;   // config slots start after original 64 KB EEPROM (0x0000–0x0FFFF)
const CFG_SLOT_SIZE      = 0x2000;    // 8 KB per slot — must match CONFIG_SIZE in bootloader eeprom.c
const ORIG_EEPROM_SIZE   = 0x10000;   // first 64 KB: original radio EEPROM space
const ZONE_A_FW_BASE     = 0x1C000;   // Zone A firmware starts at 0x1C000 (config slots end at 0x18000)
const ZONE_A_FW_SLOT_SIZE= 0xF000;    // 60 KB per Zone A slot (2×60 KB = 0x1E000, ends at 0x3A000 < SSB @ 0x3C228)
const ZONE_A_FW_COUNT    = 2;         // two Zone A slots (0x1C000 and 0x2B000)
const ZONE_A_FW_END      = ZONE_A_FW_BASE + ZONE_A_FW_COUNT * ZONE_A_FW_SLOT_SIZE; // 0x3A000
const SSB_PATCH_ADDR     = 0x3C228;   // written by chi page's "Set SSB patch" button
const SSB_PATCH_SIZE     = 15832;     // size of ssb.bin (0x3DD8 bytes, ends at 0x3FFFF)
const CALIB_ADDR         = 0x1E00;    // hardware calibration block (RSSI @ 0x1EC0, battery @ 0x1F40)
const CALIB_SIZE         = 0x0200;    // 512 bytes (0x1E00–0x1FFF)
const CALIB_BACKUP_ADDR  = 0x3A000;  // calibration backup: after Zone A (ends 0x39FFF), before SSB (0x3C228)

const PALETTE = ['#4CAF50','#7C4DFF','#F44336','#00BCD4','#FF9800','#E91E63'];
const C_LIVE  = '#1565C0';
const C_FREE  = '#BDBDBD';
const C_META  = '#E65100';
const C_BL    = '#B71C1C';
const C_SSB   = '#00796B';
const C_CALIB = '#F57F17';  // amber — calibration region

// ── State ────────────────────────────────────────────────────────────────────
type Phase = 'idle' | 'ready';

const phase       = ref<Phase>('idle');
const reading     = ref(false);
const writing     = ref(false);
const fullReconfig= ref(false);
const hovered     = ref<any>(null);
const highlight   = ref(-1);

const device = reactive({
  slotCount:      0,
  lastFw:         0xFF as number,
  slots:          [] as any[],   // [{index, name, start, end, capacity}]
  ssbPatchPresent: false,
});

// Per-slot replacements (normal mode): idx → {firmware, filename, name, oversize}
const replacements = reactive<Record<number, any>>({});

// Full reconfigure state
const rc = reactive({ numSlots: 4, slots: [] as any[] });
const rcZoneACount = computed(() => Math.max(0, rc.numSlots - 4));
const rcZoneBCount = computed(() => Math.min(rc.numSlots, 4));

// Bootloader binary — loaded from /bootloader.bin on mount, or replaced by user file picker
const bl = ref<Uint8Array | null>(null);
const blName = ref('');
const blIsCustom = ref(false);
const calibFile = ref<{ name: string; data: Uint8Array } | null>(null);


// ── Single progress bar ───────────────────────────────────────────────────────
const prog = reactive({ visible: false, label: '', pct: 0, color: '#1890ff' });
const statusMsg     = ref('');
const statusMsgType = ref('');

function setStatus(msg: string, type: 'error' | 'success' | '' = '') {
  statusMsg.value = msg; statusMsgType.value = type;
}
function showBar(color: string) {
  prog.visible = true; prog.pct = 0; prog.color = color; prog.label = '';
}
function hideBar() { prog.visible = false; }
// Update label + pct and yield to the renderer so the bar actually moves.
async function tickBar(label: string, pct: number) {
  prog.label = label;
  prog.pct   = Math.min(100, pct);
  await nextTick();
  // One extra microtask yield so the browser can paint before the next serial op.
  await new Promise(r => setTimeout(r, 0));
}

// ── Helpers ──────────────────────────────────────────────────────────────────
function p5(n: number) { return n.toString(16).toUpperCase().padStart(5, '0'); }

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
    const res = await fetch('/bootloader.bin');
    if (!res.body) return;
    const reader = res.body.getReader();
    const chunks: number[] = [];
    while (true) { const { done, value } = await reader.read(); if (done) break; chunks.push(...value); }
    const buf = new Uint8Array(0x3000); buf.set(chunks, 0); bl.value = buf;
    blName.value = 'bootloader.bin (built-in)';
    blIsCustom.value = false;
  } catch { /* silent */ }
}

function loadCustomBL() {
  const input = document.createElement('input');
  input.type = 'file'; input.accept = '.bin';
  input.onchange = async () => {
    if (!input.files?.length) return;
    const file = input.files[0];
    const raw  = new Uint8Array(await file.arrayBuffer());
    const buf  = new Uint8Array(0x3000); buf.set(raw.slice(0, 0x3000), 0);
    bl.value = buf;
    blName.value = file.name;
    blIsCustom.value = true;
  };
  input.click();
}

onMounted(loadBL);

// ── EEPROM map regions ────────────────────────────────────────────────────────
function seg(name: string, short: string, start: number, size: number,
             color: string, desc: string, extra: any = {}): any {
  return { name, short, start, size, end: start + size - 1,
           flex: Math.max(Math.round(size / 64), 4), color, bgColor: color, desc,
           fillPct: 0, slotIdx: null, ...extra };
}

function calibBackupSeg() {
  return seg('Calibration Backup', 'Cal BK', CALIB_BACKUP_ADDR, CALIB_SIZE, C_CALIB,
    `Reserved area at 0x${CALIB_BACKUP_ADDR.toString(16).toUpperCase()} (512 B). Unused — calibration is now part of each firmware's 8 KB config slot snapshot.`);
}

const mapMgmt = computed<any[]>(() => {
  const n  = device.slotCount || 0;
  const rs: any[] = [];

  // Live config — split at the calibration block
  rs.push(seg('Live Config (0x0000–0x1DFF)', 'Config', 0x00000, CALIB_ADDR, C_LIVE,
    'Active radio config: channel memories, VFO state, and user settings. ' +
    'Part of the 8 KB per-firmware snapshot saved and restored by the bootloader on every firmware switch.'));
  rs.push(seg('RF Calibration (0x1E00–0x1FFF)', 'Cal', CALIB_ADDR, CALIB_SIZE, C_CALIB,
    'Hardware-specific RSSI and battery calibration. Included in each firmware\'s config slot snapshot (8 KB). ' +
    'Saved and restored automatically on every firmware switch.'));

  // Original 64 KB EEPROM remainder: channel names (0x4000), memories (0x8000), etc.
  rs.push(seg('Original EEPROM Data (0x2000–0xFFFF)', 'Channels/Mem',
    0x2000, ORIG_EEPROM_SIZE - 0x2000, C_FREE,
    'Remainder of the original 64 KB EEPROM: extended config, channel names (0x4000–0x7FFF), ' +
    'channel data (0x8000–0xEFFF), FM/misc (0xF000–0xFFFF). Shared across all firmwares — not backed up per slot.'));

  // Config slots (start at 0x10000, after original 64 KB EEPROM)
  for (let i = 0; i < n; i++) {
    rs.push(seg(`Config Slot ${i+1}`, `Cfg${i+1}`,
      CFG_SLOT_BASE + i * CFG_SLOT_SIZE, CFG_SLOT_SIZE, lighten(slotColor(i)),
      `Config snapshot for firmware slot ${i+1}. Saved here before switching away, restored when switching back. ` +
      `Blanked (all 0xFF) when a new firmware is written to slot ${i+1} so the first boot gets clean defaults.`));
  }
  const cfgEnd = n > 0 ? CFG_SLOT_BASE + n * CFG_SLOT_SIZE : CFG_SLOT_BASE;

  // Zone A firmware slots: slots stored at addresses in [ZONE_A_FW_BASE, FW_BASE)
  const zoneASlots = device.slots
    .filter((s: any) => s.start >= ZONE_A_FW_BASE && s.start < FW_BASE)
    .sort((a: any, b: any) => a.start - b.start);

  let cursor = cfgEnd;

  if (zoneASlots.length > 0) {
    // Free between cursor and first Zone A firmware slot
    if (cursor < zoneASlots[0].start)
      rs.push(seg('Free', '', cursor, zoneASlots[0].start - cursor, C_FREE,
        `Unused — ${((zoneASlots[0].start - cursor) / 1024).toFixed(1)} KB.`));

    for (let i = 0; i < zoneASlots.length; i++) {
      const s   = zoneASlots[i];
      const idx = device.slots.indexOf(s);
      const col = slotColor(idx);
      const cap = s.capacity;
      const used = s.end - s.start + 1;
      const repl = replacements[idx];
      const fillSz = repl ? Math.ceil(repl.firmware.length / 0x40) * 0x40 : used;
      rs.push({
        name:    `FW Slot ${s.index}${s.name ? ' · ' + s.name : ''} (Zone A)`,
        short:   s.name || `Slot ${s.index}`,
        start:   s.start, end: s.start + cap - 1, size: cap,
        flex:    Math.max(Math.round(cap / 64), 8),
        color:   col, bgColor: lighten(col, 0.68),
        desc:    `Zone A firmware slot ${s.index}${s.name ? ' "' + s.name + '"' : ''} — ${(used/1024).toFixed(1)} KB used of ${(cap/1024).toFixed(1)} KB capacity. ` +
                 `Range 0x${p5(s.start)}–0x${p5(s.start + cap - 1)}. ` +
                 `Config snapshot at 0x${(CFG_SLOT_BASE + idx * CFG_SLOT_SIZE).toString(16).toUpperCase()}. ` +
                 (repl ? `Replacement loaded: ${repl.filename} (${(repl.firmware.length/1024).toFixed(1)} KB).` : 'Click to jump to slot card.'),
        fillPct: Math.min(fillSz / cap * 100, 100),
        slotIdx: idx,
      });
      cursor = s.start + cap;

      // Free between this Zone A slot and the next
      const nextBoundary = i + 1 < zoneASlots.length ? zoneASlots[i + 1].start : CALIB_BACKUP_ADDR;
      if (cursor < nextBoundary)
        rs.push(seg('Free', '', cursor, nextBoundary - cursor, C_FREE,
          `Unused — ${((nextBoundary - cursor) / 1024).toFixed(1)} KB.`));
    }
    // Calibration backup after Zone A
    cursor = CALIB_BACKUP_ADDR + CALIB_SIZE;
    rs.push(calibBackupSeg());
    // SSB patch (if present)
    if (device.ssbPatchPresent) {
      if (cursor < SSB_PATCH_ADDR)
        rs.push(seg('Free', '', cursor, SSB_PATCH_ADDR - cursor, C_FREE,
          `Unused — ${((SSB_PATCH_ADDR - cursor) / 1024).toFixed(1)} KB.`));
      rs.push(seg('SI4732 SSB patch', 'SSB', SSB_PATCH_ADDR, SSB_PATCH_SIZE, C_SSB,
        `SI4732 SSB patch. ${(SSB_PATCH_SIZE / 1024).toFixed(1)} KB — does not overlap with multiboot regions.`));
      const ssbEnd = SSB_PATCH_ADDR + SSB_PATCH_SIZE;
      if (ssbEnd < META_BASE)
        rs.push(seg('Free', '', ssbEnd, META_BASE - ssbEnd, C_FREE,
          `Unused — ${((META_BASE - ssbEnd) / 1024).toFixed(1)} KB.`));
    } else if (cursor < META_BASE) {
      rs.push(seg('Free', '', cursor, META_BASE - cursor, C_FREE,
        `Unused — ${((META_BASE - cursor) / 1024).toFixed(1)} KB.`));
    }
  } else {
    // No Zone A firmware slots
    if (cursor < CALIB_BACKUP_ADDR)
      rs.push(seg('Free', '', cursor, CALIB_BACKUP_ADDR - cursor, C_FREE,
        `Unused — ${((CALIB_BACKUP_ADDR - cursor) / 1024).toFixed(1)} KB.`));
    cursor = CALIB_BACKUP_ADDR + CALIB_SIZE;
    rs.push(calibBackupSeg());
    if (device.ssbPatchPresent) {
      if (cursor < SSB_PATCH_ADDR)
        rs.push(seg('Free', '', cursor, SSB_PATCH_ADDR - cursor, C_FREE,
          `Unused — ${((SSB_PATCH_ADDR - cursor) / 1024).toFixed(1)} KB.`));
      rs.push(seg('SI4732 SSB patch', 'SSB', SSB_PATCH_ADDR, SSB_PATCH_SIZE, C_SSB,
        `SI4732 SSB patch. ${(SSB_PATCH_SIZE / 1024).toFixed(1)} KB at 0x${SSB_PATCH_ADDR.toString(16).toUpperCase()}–0x${(SSB_PATCH_ADDR + SSB_PATCH_SIZE - 1).toString(16).toUpperCase()}.`));
      const ssbEnd = SSB_PATCH_ADDR + SSB_PATCH_SIZE;
      if (ssbEnd < META_BASE)
        rs.push(seg('Free', '', ssbEnd, META_BASE - ssbEnd, C_FREE,
          `Unused — ${((META_BASE - ssbEnd) / 1024).toFixed(1)} KB.`));
    } else if (cursor < META_BASE) {
      rs.push(seg('Free', '', cursor, META_BASE - cursor, C_FREE,
        `Unused free zone — ${((META_BASE - cursor) / 1024).toFixed(0)} KB.`));
    }
  }

  rs.push(seg('Metadata', 'Meta', META_BASE, 0x1000, C_META,
    '0x40000: slot count (1 B) · 0x40001: boot mode flag · 0x40002: last-run slot index · ' +
    '0x40008: bootloader name (8 B) · 0x40020+: slot table (32 B/slot: 16 B name | 4 B start LE | 4 B end LE | 8 B padding)'));
  rs.push(seg('Bootloader B', 'BL-B', BL_ADDR, 0x3000, C_BL,
    'Losehu Bootloader B (bootloader.bin, ~11 KB). Executed by Bootloader A in MCU flash when radio powers on with MENU held. ' +
    'Handles firmware selection, config save/restore, and UART flashing.'));
  return rs;
});

const mapFw = computed<any[]>(() => {
  const rs: any[] = [];
  // Zone B only: firmware slots stored at addresses >= FW_BASE (0x44000)
  const zoneBSlots = device.slots.filter((s: any) => s.start >= FW_BASE);
  for (const s of zoneBSlots) {
    const idx    = device.slots.indexOf(s);
    const cap    = s.capacity;
    const used   = s.end - s.start + 1;
    const repl   = replacements[idx];
    const fillSz = repl ? Math.ceil(repl.firmware.length / 0x40) * 0x40 : used;
    const col    = slotColor(idx);
    rs.push({
      name:    `FW Slot ${s.index}${s.name ? ' · ' + s.name : ''}${repl ? ' [→ ' + repl.filename + ']' : ''}`,
      short:   s.name || `Slot ${s.index}`,
      start:   s.start, end: s.start + cap - 1, size: cap,
      flex:    Math.max(Math.round(cap / 64), 8),
      color:   col, bgColor: lighten(col, 0.68),
      desc:    `Slot ${s.index}${s.name ? ' "' + s.name + '"' : ''} — ${(used/1024).toFixed(1)} KB used of ${(cap/1024).toFixed(1)} KB capacity. ` +
               `Range 0x${p5(s.start)}–0x${p5(s.start + cap - 1)}. Config snapshot @ 0x${(CFG_SLOT_BASE + idx * CFG_SLOT_SIZE).toString(16).toUpperCase()}. ` +
               (repl ? `Replacement loaded: ${repl.filename} (${(repl.firmware.length/1024).toFixed(1)} KB).` : 'Click to jump to slot card.'),
      fillPct: Math.min(fillSz / cap * 100, 100),
      slotIdx: idx,
    });
  }
  const tailStart = zoneBSlots.length > 0
    ? zoneBSlots[zoneBSlots.length - 1].start + zoneBSlots[zoneBSlots.length - 1].capacity
    : FW_BASE;
  if (tailStart < EEPROM_SIZE)
    rs.push(seg('Free', 'Free', tailStart, EEPROM_SIZE - tailStart, C_FREE,
      `Unused EEPROM — ${((EEPROM_SIZE - tailStart)/1024).toFixed(1)} KB.`));
  return rs;
});

const mapLegend = computed(() => {
  const items: { label: string; color: string }[] = [
    { label: 'Live Config',    color: C_LIVE  },
    { label: 'RF Calibration', color: C_CALIB },
    { label: 'Free / Unused',  color: C_FREE  },
  ];
  for (let i = 0; i < device.slotCount; i++)
    items.push({ label: `Config Slot ${i+1}`, color: lighten(slotColor(i)) });
  if (device.ssbPatchPresent)
    items.push({ label: 'SI4732 SSB patch', color: C_SSB });
  items.push({ label: 'Metadata', color: C_META }, { label: 'Bootloader B', color: C_BL });
  for (let i = 0; i < device.slots.length; i++) {
    const s     = device.slots[i];
    const label = `FW Slot ${s.index}${s.name ? ' · ' + s.name : ''}${s.start < FW_BASE ? ' (Zone A)' : ''}`;
    items.push({ label, color: slotColor(i) });
  }
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
    setStatus('ERROR: requires losehu bootloader mode — MENU + power on, then reconnect.', 'error'); return;
  }
  reading.value = true;
  setLoading(true);
  hideBar();
  statusMsg.value = '';
  // Reset state
  Object.assign(device, { slotCount: 0, lastFw: 0xFF, slots: [], ssbPatchPresent: false });
  Object.keys(replacements).forEach(k => delete replacements[+k]);
  fullReconfig.value = false;

  try {
    await eeprom_init(appStore.connectPort);

    // Detect SSB patch: read 8 bytes at its start; if not all 0xFF it's present.
    const ssbProbe = await readBytes(SSB_PATCH_ADDR, 8);
    device.ssbPatchPresent = ssbProbe.some(b => b !== 0xFF);

    const countBuf = await readBytes(0x40000, 1);
    const count = countBuf[0];

    if (!count || count > 64) {
      setStatus('No multiboot metadata found. Use "Full reconfigure" to set up the device.', '');
      phase.value = 'ready';
      device.slotCount = 0;
      fullReconfig.value = true;
      rebuildRcSlots();
      reading.value = false; setLoading(false); return;
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
      let capacity = nextStart - r.start;
      // Cap Zone A slots to ZONE_A_FW_SLOT_SIZE to prevent writes from
      // overrunning the SSB patch area that starts at 0x3C228.
      if (r.start >= ZONE_A_FW_BASE && r.start < FW_BASE)
        capacity = Math.min(capacity, ZONE_A_FW_SLOT_SIZE);
      return {
        index: i+1, name: r.name, start: r.start, end: r.end, capacity,
        startHex: '0x' + p5(r.start), endHex: '0x' + p5(r.end),
      };
    });
    device.slotCount = count;
    phase.value = 'ready';

    // Inject calibration into every config slot (uploaded file or live device).
    if (count > 0) {
      const calibData = await getCalibData();
      if (calibData) await writeCalibToAllSlots(calibData, count);
    }

  } catch (e: any) {
    setStatus('Error reading device: ' + errMsg(e), 'error');
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
  if (appStore.configuration?.uart !== 'losehu') { setStatus('ERROR: bootloader mode required.', 'error'); return; }

  writing.value = true;
  statusMsg.value = '';
  const color = PALETTE[idx % PALETTE.length];
  showBar(color);

  const fw     = repl.firmware as Uint8Array;
  const padded = new Uint8Array(Math.ceil(fw.length / 0x40) * 0x40).fill(0xFF);
  padded.set(fw, 0);
  const newEnd = slot.start + padded.length - 1;

  try {
    await eeprom_init(appStore.connectPort);

    // Firmware
    for (let i = slot.start; i < slot.start + padded.length; i += 0x40) {
      const chunk  = padded.slice(i - slot.start, i - slot.start + 0x40);
      await eeprom_write(appStore.connectPort, i, chunk, chunk.length, appStore.configuration?.uart);
      const verify = await eeprom_read(appStore.connectPort, i, chunk.length, appStore.configuration?.uart);
      if (!isEqual(chunk, verify)) { i -= 0x40; continue; }
      const pct = (i - slot.start + 0x40) / padded.length * 100;
      await tickBar(`Writing ${repl.name} to 0x${p5(i)}`, pct);
    }

    // Metadata
    await tickBar(`Writing metadata to 0x${p5(0x40020 + 32 * (slot.index - 1))}`, 100);
    const metaBase = 0x40020 + 32 * (slot.index - 1);
    const meta = new Uint8Array(32).fill(0x00);
    meta.set(stringToUint8Array(repl.name.substring(0, 13)), 0);
    meta.set(hexReverseStringToUint8Array(slot.start.toString(16)), 16);
    meta.set(hexReverseStringToUint8Array(newEnd.toString(16)), 20);
    await eeprom_write(appStore.connectPort, metaBase,        meta.slice(0, 0x10), 0x10, appStore.configuration?.uart);
    await eeprom_write(appStore.connectPort, metaBase + 0x10, meta.slice(0x10),    0x10, appStore.configuration?.uart);

    // Config slot clear
    await clearConfigSlotWithProg(slot.index, color);

    await tickBar('Done', 100);
    setStatus('Flash complete.', 'success');
    clearReplacement(idx);
    await eeprom_reboot(appStore.connectPort);
  } catch (e: any) { setStatus('Error: ' + errMsg(e), 'error'); hideBar(); }
  writing.value = false;
}

async function flashAllUpdated() {
  const toFlash = Object.entries(replacements)
    .filter(([, r]: any) => r && !r.oversize)
    .map(([idx, r]) => ({ idx: +idx, repl: r }));
  if (!toFlash.length) return;

  writing.value = true;
  statusMsg.value = '';
  showBar('#1890ff');

  try {
    await eeprom_init(appStore.connectPort);
    for (const { idx, repl } of toFlash) {
      const slot  = device.slots[idx];
      const color = PALETTE[idx % PALETTE.length];
      prog.color  = color;
      const fw    = repl.firmware as Uint8Array;
      const padded = new Uint8Array(Math.ceil(fw.length / 0x40) * 0x40).fill(0xFF);
      padded.set(fw, 0);
      const newEnd = slot.start + padded.length - 1;

      for (let i = slot.start; i < slot.start + padded.length; i += 0x40) {
        const chunk  = padded.slice(i - slot.start, i - slot.start + 0x40);
        await eeprom_write(appStore.connectPort, i, chunk, chunk.length, appStore.configuration?.uart);
        const verify = await eeprom_read(appStore.connectPort, i, chunk.length, appStore.configuration?.uart);
        if (!isEqual(chunk, verify)) { i -= 0x40; continue; }
        const pct = (i - slot.start + 0x40) / padded.length * 100;
        await tickBar(`Writing ${repl.name} to 0x${p5(i)}`, pct);
      }

      prog.color = '#607D8B';
      await tickBar(`Writing metadata to 0x${p5(0x40020 + 32 * (slot.index - 1))}`, 100);
      const metaBase = 0x40020 + 32 * (slot.index - 1);
      const meta = new Uint8Array(32).fill(0x00);
      meta.set(stringToUint8Array(repl.name.substring(0, 13)), 0);
      meta.set(hexReverseStringToUint8Array(slot.start.toString(16)), 16);
      meta.set(hexReverseStringToUint8Array(newEnd.toString(16)), 20);
      await eeprom_write(appStore.connectPort, metaBase,        meta.slice(0, 0x10), 0x10, appStore.configuration?.uart);
      await eeprom_write(appStore.connectPort, metaBase + 0x10, meta.slice(0x10),    0x10, appStore.configuration?.uart);

      prog.color = color;
      await clearConfigSlotWithProg(slot.index, color);
    }

    await tickBar('Done', 100);
    setStatus('Flash complete.', 'success');
    Object.keys(replacements).forEach(k => delete replacements[+k]);
    await eeprom_reboot(appStore.connectPort);
  } catch (e: any) { setStatus('Error: ' + errMsg(e), 'error'); hideBar(); }
  writing.value = false;
}

// ── Config slot clear ─────────────────────────────────────────────────────────
// Write live calibration into every config slot at offset +0x1E00.
// This guarantees the bootloader can always restore valid calibration
// on every firmware switch, even on first boot of a freshly cleared slot.
async function writeCalibToAllSlots(calibData: Uint8Array, slotCount: number) {
  prog.color = C_CALIB;
  for (let i = 1; i <= slotCount; i++) {
    const addr = CFG_SLOT_BASE + (i - 1) * CFG_SLOT_SIZE + CALIB_ADDR;
    await writeRangeWithProg(addr, calibData, `cal → slot ${i}`);
  }
}

// Return calibration data: uploaded file if set, otherwise live device.
async function getCalibData(): Promise<Uint8Array | null> {
  if (calibFile.value) return calibFile.value.data;
  const data = await readBytes(CALIB_ADDR, CALIB_SIZE);
  return data.some(b => b !== 0xFF) ? data : null;
}

function loadCalibFile() {
  const input = document.createElement('input');
  input.type = 'file'; input.accept = '.bin';
  input.onchange = async () => {
    if (!input.files?.length) return;
    const file = input.files[0];
    const raw = new Uint8Array(await file.arrayBuffer());
    if (raw.length !== CALIB_SIZE) {
      setStatus(`Calibration file must be exactly ${CALIB_SIZE} bytes (got ${raw.length}).`, 'error');
      return;
    }
    calibFile.value = { name: file.name, data: raw };
    setStatus(`Calibration file "${file.name}" loaded. Click "Apply to all slots" to write.`, 'success');
  };
  input.click();
}

async function applyCalibNow() {
  if (!appStore.connectState) { alert(sessionStorage.getItem('noticeConnectK5')); return; }
  if (device.slotCount < 1) { setStatus('Read device first.', 'error'); return; }
  writing.value = true; statusMsg.value = '';
  showBar(C_CALIB);
  try {
    await eeprom_init(appStore.connectPort);
    const data = await getCalibData();
    if (!data) { setStatus('No calibration data available.', 'error'); hideBar(); writing.value = false; return; }
    await writeCalibToAllSlots(data, device.slotCount);
    await tickBar('Done', 100);
    setStatus('Calibration written to all config slots.', 'success');
  } catch (e: any) { setStatus('Error: ' + errMsg(e), 'error'); hideBar(); }
  writing.value = false;
}

async function clearConfigSlotWithProg(slotIndex: number, color: string) {
  const addr  = CFG_SLOT_BASE + (slotIndex - 1) * CFG_SLOT_SIZE;
  const blank = new Uint8Array(0x40).fill(0xFF);
  const total = CFG_SLOT_SIZE / 0x40;
  prog.color  = color;
  let done = 0;
  for (let i = addr; i < addr + CFG_SLOT_SIZE; i += 0x40) {
    await eeprom_write(appStore.connectPort, i, blank, 0x40, appStore.configuration?.uart);
    await tickBar(`Clearing config slot ${slotIndex} at 0x${p5(i)}`, ++done / total * 100);
  }
}


// ── Bootloader only ───────────────────────────────────────────────────────────
async function writeBootloaderOnly() {
  if (!appStore.connectState) { alert(sessionStorage.getItem('noticeConnectK5')); return; }
  if (!bl.value) { setStatus('Bootloader binary not loaded — use "Load custom BL" to select a file.', 'error'); return; }
  writing.value = true; statusMsg.value = '';
  showBar(C_BL);
  try {
    await eeprom_init(appStore.connectPort);
    await writeRangeWithProg(BL_ADDR, bl.value, 'bootloader');
    await tickBar('Done', 100);
    setStatus('Flash complete.', 'success');
    await eeprom_reboot(appStore.connectPort);
  } catch (e: any) { setStatus('Error: ' + errMsg(e), 'error'); hideBar(); }
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

// Zone B slot size: divide Zone B space among Zone B slots (min 4, max numSlots if ≤4).
const rcSlotSize = computed(() => {
  const nB  = rcZoneBCount.value;
  const raw = Math.floor((EEPROM_SIZE - FW_BASE) / Math.max(1, nB));
  return Math.floor(raw / 0x40) * 0x40;
});
const rcSlotSizeKb = computed(() => (rcSlotSize.value / 1024).toFixed(1));

function rcSlotStart(i: number) {
  // Slots 0..rcZoneACount-1 → Zone A (0x1C000, 0x2B000, …)
  // Slots rcZoneACount..numSlots-1 → Zone B (0x44000, …)
  if (i < rcZoneACount.value)
    return ZONE_A_FW_BASE + i * ZONE_A_FW_SLOT_SIZE;
  return FW_BASE + (i - rcZoneACount.value) * rcSlotSize.value;
}

function rebuildRcSlots() {
  const prev = rc.slots.slice();
  rc.slots = Array.from({ length: rc.numSlots }, (_, i) => {
    const isZoneA = i < rcZoneACount.value;
    const cap     = isZoneA ? ZONE_A_FW_SLOT_SIZE : rcSlotSize.value;
    const start   = rcSlotStart(i);
    const s: any  = {
      index: i+1, start, end: start + cap - 1, capacity: cap, zoneA: isZoneA,
      name: '', filename: '', firmware: null, oversize: false,
    };
    if (prev[i]) {
      Object.assign(s, { name: prev[i].name, filename: prev[i].filename, firmware: prev[i].firmware });
      if (s.firmware) s.oversize = Math.ceil(s.firmware.length / 0x40) * 0x40 > cap;
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
    slot.oversize = Math.ceil(fw.length / 0x40) * 0x40 > slot.capacity;
    if (!slot.name) slot.name = file.name.replace(/\.bin$/i,'').replace(/[^\x00-\x7f]/g,'').substring(0,13);
  };
  input.click();
}

// ── Write range with single progress bar ──────────────────────────────────────
async function writeRangeWithProg(start: number, data: Uint8Array | number[], name: string) {
  const arr = data instanceof Uint8Array ? data : new Uint8Array(data);
  for (let i = start; i < start + arr.length; i += 0x40) {
    const chunk  = arr.slice(i - start, i - start + 0x40);
    await eeprom_write(appStore.connectPort, i, chunk, chunk.length, appStore.configuration?.uart);
    const verify = await eeprom_read(appStore.connectPort, i, chunk.length, appStore.configuration?.uart);
    if (!isEqual(chunk, verify)) { i -= 0x40; continue; }
    const pct = (i - start + 0x40) / arr.length * 100;
    await tickBar(`Writing ${name} to 0x${p5(i)}`, pct);
  }
}

// ── Full reconfigure ──────────────────────────────────────────────────────────
async function writeAll() {
  if (!appStore.connectState) { alert(sessionStorage.getItem('noticeConnectK5')); return; }
  if (!bl.value) { setStatus('Bootloader binary not loaded.', 'error'); return; }
  const filled = rc.slots.filter((s: any) => s.firmware && !s.oversize);
  if (!filled.length) { setStatus('No valid firmware assigned to any slot.', 'error'); return; }

  writing.value = true; statusMsg.value = '';
  showBar(C_BL);

  try {
    const sz = await check_eeprom(appStore.connectPort, appStore.configuration?.uart);
    if (sz < 0x80000) { alert('Only 4Mbit (512 KB) EEPROM supported.'); writing.value = false; return; }
    await eeprom_init(appStore.connectPort);

    await writeRangeWithProg(BL_ADDR, bl.value, 'bootloader');

    // Reset last-FW index so bootloader starts clean
    await eeprom_write(appStore.connectPort, 0x40002, new Uint8Array([0xFF]), 1, appStore.configuration?.uart);

    // Build metadata buffer
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
    await eeprom_write(appStore.connectPort, META_BASE, new Uint8Array([filled.length]), 1, appStore.configuration?.uart);
    const blNameArr = new Uint8Array(8);
    blNameArr.set(stringToUint8Array(blName.value.substring(0, 8)));
    await eeprom_write(appStore.connectPort, 0x40008, blNameArr, 8, appStore.configuration?.uart);

    prog.color = '#607D8B';
    await writeRangeWithProg(0x40020, new Uint8Array(metaBuf), 'metadata');

    for (let fi = 0; fi < filled.length; fi++) {
      const slot  = filled[fi];
      const blIdx = fi + 1;                              // bootloader now_menu = position in filled
      const color = PALETTE[(slot.index - 1) % PALETTE.length];
      prog.color  = color;
      await writeRangeWithProg(slot.start, slot._padded, slot.name || `Slot${slot.index}`);
      await clearConfigSlotWithProg(blIdx, color);
    }

    // Inject calibration into every config slot (uploaded file or live device).
    const calibData = await getCalibData();
    if (calibData) await writeCalibToAllSlots(calibData, filled.length);

    await tickBar('Done', 100);
    setStatus('Flash complete.', 'success');
    await eeprom_reboot(appStore.connectPort);
    fullReconfig.value = false;
  } catch (e: any) { setStatus('Error: ' + errMsg(e), 'error'); hideBar(); }
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

/* Progress area */
.progress-area {
  margin-top: 12px;
  padding: 14px 16px;
  background: var(--color-bg-3);
  border-radius: 6px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.status-msg {
  font-size: 0.85rem;
  padding: 6px 10px;
  border-radius: 4px;
  background: var(--color-bg-2);
  color: var(--color-text-2);
  &.error   { background: #fff1f0; color: #cf1322; border: 1px solid #ffa39e; }
  &.success { background: #f6ffed; color: #389e0d; border: 1px solid #b7eb8f; }
}

.prog-row {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.prog-label {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  font-size: 0.82rem;
}

.prog-name {
  color: var(--color-text-2);
  font-weight: 500;
}

.prog-pct {
  color: var(--color-text-3);
  font-size: 0.78rem;
  font-variant-numeric: tabular-nums;
}

.prog-track {
  height: 8px;
  background: var(--color-fill-2);
  border-radius: 4px;
  overflow: hidden;
}

.prog-fill {
  height: 100%;
  border-radius: 4px;
  transition: width 0.15s ease-out;
}
</style>
