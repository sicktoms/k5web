<template>
  <div class="container">
    <Breadcrumb :items="[$t('menu.list'), $t('bl')]" />
    <a-card class="general-card" :loading="loading">
      <template #title>
        <div style="color: red; font-weight: bold;">
          {{ $t('bl.warning') }}
          <span style="font-size: 0.9rem;">【{{ $t('bl.readme') }}<t-link theme="primary" href="https://github.com/losehu/uv-k5-bootloader-custom/releases" target="_blank">https://github.com/losehu/uv-k5-bootloader-custom/releases</t-link>】</span>
        </div>
        {{ $t('bl') }} {{ $t('bl.modeNotice') }}
      </template>

      <!-- Slot count + write button row -->
      <a-row style="margin-bottom: 20px; align-items: center;" :gutter="16">
        <a-col :flex="'none'">
          <span style="margin-right: 8px;">{{ $t('bl.numSlots') }}</span>
          <a-select v-model="state.numSlots" style="width: 80px;" @change="rebuildSlots">
            <a-option v-for="n in [1,2,3,4,5,6]" :key="n" :value="n">{{ n }}</a-option>
          </a-select>
          <span style="margin-left: 16px; color: var(--color-text-3); font-size: 0.85rem;">
            {{ $t('bl.slotSize') }}: {{ slotSizeKb }} KB &nbsp;·&nbsp; {{ $t('bl.slotBase') }}: 0x44000
          </span>
        </a-col>
        <a-col :flex="'auto'" style="text-align: right; display: flex; gap: 8px; justify-content: flex-end;">
          <a-button :loading="state.writing" @click="writeBootloaderOnly">
            <template #icon><icon-upload /></template>
            {{ $t('bl.writeBlOnly') }}
          </a-button>
          <a-button type="primary" status="danger" :loading="state.writing" @click="writeAll"
            :disabled="!state.slots.some((s:any) => s.firmware)">
            <template #icon><icon-thunderbolt /></template>
            {{ $t('bl.writeAll') }}
          </a-button>
        </a-col>
      </a-row>

      <!-- Slot cards -->
      <a-row :gutter="[16, 16]">
        <a-col v-for="(slot, idx) in state.slots" :key="idx" :span="12">
          <a-card :bordered="true" :style="slot.firmware ? { borderColor: slot.color } : {}">
            <template #title>
              <span :style="slot.firmware ? { color: slot.color } : {}">
                {{ $t('bl.slot') }} {{ slot.index }}
              </span>
              <span v-if="slot.firmware" style="font-weight:normal; font-size:0.82rem; margin-left:8px; color: var(--color-text-3);">
                0x{{ slot.start.toString(16).toUpperCase() }} – 0x{{ slot.end.toString(16).toUpperCase() }}
                &nbsp;({{ (slot.firmware.length / 1024).toFixed(1) }} / {{ slotSizeKb }} KB)
              </span>
            </template>
            <template #extra>
              <div v-if="slot.firmware" :style="'width:12px;height:12px;border-radius:2px;background:'+slot.color" />
            </template>

            <!-- Name -->
            <a-form-item :label="$t('bl.slotName')" style="margin-bottom:10px">
              <a-input v-model="slot.name" :max-length="13" show-word-limit
                :placeholder="'Slot ' + slot.index"
                @change="() => slot.name = slot.name.replace(/[^\x00-\x7f]/g, '')" />
            </a-form-item>

            <!-- Firmware file -->
            <a-space>
              <a-button @click="selectFile(idx)">
                <template #icon><icon-upload /></template>
                {{ slot.firmware ? slot.filename : $t('tool.selectFirmware') }}
              </a-button>
              <a-tag v-if="slot.firmware && slot.oversize" color="red">{{ $t('bl.oversize') }}</a-tag>
              <a-tag v-else-if="slot.firmware" color="green">OK</a-tag>
              <a-button v-if="slot.firmware" type="text" status="danger" size="small" @click="clearSlot(idx)">
                <template #icon><icon-delete /></template>
              </a-button>
            </a-space>
          </a-card>
        </a-col>
      </a-row>
    </a-card>

    <!-- Status log -->
    <div id="statusArea"
      style="height: 20em; background-color: var(--color-bg-3); color: var(--color-text-3);
             overflow: auto; padding: 20px; margin-top: 10px; font-family: monospace; font-size: 0.82rem;"
      v-html="state.status" />
  </div>
</template>

<script lang="ts" setup>
import { reactive, computed, onMounted, nextTick } from 'vue';
import useLoading from '@/hooks/loading';
import { useAppStore } from '@/store';
import {
  eeprom_write, eeprom_reboot, eeprom_init, eeprom_read,
  stringToUint8Array, check_eeprom, hexReverseStringToUint8Array, unpack
} from '@/utils/serial.js';

const appStore = useAppStore();
const { loading, setLoading } = useLoading(false);

// ── EEPROM layout constants ──────────────────────────────────────────────────
const EEPROM_SIZE   = 0x80000;  // 512 KB
const FW_BASE       = 0x44000;  // first byte available for firmware binaries
const BL_ADDR       = 0x41000;  // bootloader B
const META_BASE     = 0x40000;  // metadata region start

// ── state ────────────────────────────────────────────────────────────────────
const state: any = reactive({
  numSlots: 4,
  slots: [],
  bl: undefined,
  blName: '',
  writing: false,
  status: '',
});

// ── computed ─────────────────────────────────────────────────────────────────
// Slot size: divide available space equally, rounded down to 64-byte boundary
const slotSize = computed(() => {
  const raw = Math.floor((EEPROM_SIZE - FW_BASE) / state.numSlots);
  return Math.floor(raw / 0x40) * 0x40;
});
const slotSizeKb = computed(() => (slotSize.value / 1024).toFixed(1));

function slotStart(idx: number) { return FW_BASE + idx * slotSize.value; }
function slotEnd(idx: number)   { return slotStart(idx) + slotSize.value - 1; }

// ── helpers ──────────────────────────────────────────────────────────────────
function log(msg: string) {
  state.status += msg + '<br/>';
  nextTick(() => {
    const el = document.getElementById('statusArea');
    if (el) el.scrollTop = el.scrollHeight;
  });
}

function randomColor() {
  const letters = '0123456789ABCDEF';
  let c = '#';
  for (let i = 0; i < 6; i++) c += letters[Math.floor(Math.random() * 16)];
  return c;
}

function isEqual(a: Uint8Array, b: Uint8Array) {
  return a.length === b.length && a.every((v, i) => v === b[i]);
}

// ── slot management ──────────────────────────────────────────────────────────
function makeSlot(idx: number) {
  return {
    index: idx + 1,
    start: slotStart(idx),
    end:   slotEnd(idx),
    name: '',
    filename: '',
    firmware: null as Uint8Array | null,
    color: randomColor(),
    oversize: false,
  };
}

function rebuildSlots() {
  // Keep existing firmware assignments where possible, recalculate addresses
  const prev = state.slots.slice();
  state.slots = Array.from({ length: state.numSlots }, (_: any, i: number) => {
    const s = makeSlot(i);
    if (prev[i]) {
      s.name     = prev[i].name;
      s.filename = prev[i].filename;
      s.firmware = prev[i].firmware;
      s.color    = prev[i].color;
      if (s.firmware) {
        const padded = Math.ceil(s.firmware.length / 0x40) * 0x40;
        s.oversize = padded > slotSize.value;
      }
    }
    return s;
  });
}

function clearSlot(idx: number) {
  const s = state.slots[idx];
  s.firmware = null;
  s.filename = '';
  s.oversize = false;
}

onMounted(() => {
  rebuildSlots();
  loadBL();
});

// ── bootloader load ──────────────────────────────────────────────────────────
const loadBL = async () => {
  setLoading(true);
  const blFilename = 'L_BL003.bin';
  state.blName = blFilename;
  const res = await fetch('/' + blFilename);
  if (res.body) {
    const reader = res.body.getReader();
    const chunks: number[] = [];
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      chunks.push(...value);
    }
    const bl = new Uint8Array(0x3000);
    bl.set(chunks, 0);
    state.bl = bl;
  }
  setLoading(false);
};

// ── file selection ───────────────────────────────────────────────────────────
function selectFile(idx: number) {
  const input = document.createElement('input');
  input.type = 'file';
  input.accept = '.bin';
  input.onchange = async () => {
    if (!input.files?.length) return;
    const file = input.files[0];
    const raw = new Uint8Array(await file.arrayBuffer());
    const unpacked = unpack(raw);
    const padded = Math.ceil(unpacked.length / 0x40) * 0x40;
    const slot = state.slots[idx];
    slot.filename = file.name;
    slot.firmware = unpacked;
    slot.oversize = padded > slotSize.value;
    if (!slot.name) {
      slot.name = file.name.replace(/\.bin$/i, '').replace(/[^\x00-\x7f]/g, '').substring(0, 13);
    }
    if (slot.oversize) {
      log(`WARNING: slot ${slot.index}: "${file.name}" is ${(padded/1024).toFixed(1)} KB but slot capacity is ${slotSizeKb.value} KB.`);
    }
  };
  input.click();
}

// ── write helpers ────────────────────────────────────────────────────────────
async function writeRange(start: number, data: Uint8Array | number[], remark: string) {
  const arr = data instanceof Uint8Array ? data : new Uint8Array(data);
  for (let i = start; i < arr.length + start; i += 0x40) {
    const chunk = arr.slice(i - start, i - start + 0x40);
    await eeprom_write(appStore.connectPort, i, chunk, chunk.length, appStore.configuration?.uart);
    const verify = await eeprom_read(appStore.connectPort, i, chunk.length, appStore.configuration?.uart);
    if (!isEqual(chunk, verify)) {
      log(remark + ' write error at 0x' + i.toString(16).toUpperCase() + ', retrying...');
      i -= 0x40;
      continue;
    }
    log(remark + ' ' + (((i - start) / arr.length) * 100).toFixed(1) + '%');
  }
}

// ── write bootloader only ────────────────────────────────────────────────────
async function writeBootloaderOnly() {
  if (appStore.connectState !== true) { alert(sessionStorage.getItem('noticeConnectK5')); return; }
  state.writing = true;
  state.status = '';
  try {
    const eepromSize = await check_eeprom(appStore.connectPort, appStore.configuration?.uart);
    if (eepromSize < 0x80000) {
      alert('Only 4Mbit (512KB) EEPROM is supported.');
      state.writing = false;
      return;
    }
    await eeprom_init(appStore.connectPort);
    log('--- Writing bootloader B only ---');
    await writeRange(BL_ADDR, state.bl, 'Bootloader');
    log('--- Done! Rebooting... ---');
    await eeprom_reboot(appStore.connectPort);
  } catch (e: any) {
    log('Error: ' + (e instanceof Error ? e.message : String(e)));
  }
  state.writing = false;
}

// ── write all ────────────────────────────────────────────────────────────────
async function writeAll() {
  if (appStore.connectState !== true) { alert(sessionStorage.getItem('noticeConnectK5')); return; }

  const filledSlots = state.slots.filter((s: any) => s.firmware && !s.oversize);
  if (filledSlots.length === 0) { log('No valid firmware assigned to any slot.'); return; }

  state.writing = true;
  state.status = '';

  try {
    const eepromSize = await check_eeprom(appStore.connectPort, appStore.configuration?.uart);
    if (eepromSize < 0x80000) {
      alert('Only 4Mbit (512KB) EEPROM is supported.');
      state.writing = false;
      return;
    }

    await eeprom_init(appStore.connectPort);

    // 1. Write bootloader B
    log('--- Writing bootloader B ---');
    await writeRange(BL_ADDR, state.bl, 'Bootloader');

    // 2. Firmware count
    await writeRange(META_BASE, new Uint8Array([filledSlots.length]), 'FW count');

    // 3. Bootloader name
    const blNameArr = new Uint8Array(8);
    blNameArr.set(stringToUint8Array(state.blName.split('.')[0]));
    await writeRange(0x40008, blNameArr, 'BL name');

    // 4. Slot metadata (32 bytes each at 0x40020)
    //    Format: 16B name | 4B start LE | 4B end LE | 8B padding
    const metaBuf: number[] = [];
    filledSlots.forEach((slot: any) => {
      const padded = new Uint8Array(Math.ceil(slot.firmware.length / 0x40) * 0x40).fill(0xff);
      padded.set(slot.firmware, 0);
      slot._padded = padded;  // cache for step 5

      const nameArr = new Uint8Array(16);
      nameArr.set(stringToUint8Array((slot.name || ('Slot' + slot.index)).substring(0, 13)));
      const startArr = new Uint8Array(4);
      startArr.set(hexReverseStringToUint8Array(slot.start.toString(16)));
      const endArr = new Uint8Array(4);
      const actualEnd = slot.start + padded.length - 1;
      endArr.set(hexReverseStringToUint8Array(actualEnd.toString(16)));
      metaBuf.push(...nameArr, ...startArr, ...endArr, ...new Uint8Array(8));
    });
    log('--- Writing slot metadata ---');
    await writeRange(0x40020, new Uint8Array(metaBuf), 'Metadata');

    // 5. Write firmware binaries
    for (const slot of filledSlots) {
      log(`--- Writing slot ${slot.index}: "${slot.name}" @ 0x${slot.start.toString(16).toUpperCase()} ---`);
      await writeRange(slot.start, slot._padded, slot.name);
    }

    log('--- Done! Rebooting... ---');
    await eeprom_reboot(appStore.connectPort);
  } catch (e: any) {
    log('Error: ' + (e instanceof Error ? e.message : String(e)));
  }
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
</style>
