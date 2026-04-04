<template>
  <div class="container">
    <Breadcrumb :items="[$t('menu.list'), $t('blupdate.title')]" />
    <a-card class="general-card">
      <template #title>
        {{ $t('blupdate.title') }}
        <span style="font-weight:normal; font-size:0.9rem; margin-left:8px; color: var(--color-text-3);">
          {{ $t('blupdate.modeNotice') }}
        </span>
      </template>

      <!-- Bootloader mode instruction -->
      <a-alert type="info" style="margin-bottom:16px">
        {{ $t('blupdate.howToConnect') }}
      </a-alert>

      <!-- Wrong protocol warning -->
      <a-alert
        v-if="appStore.connectState && appStore.configuration?.uart !== 'losehu'"
        type="error"
        style="margin-bottom:16px"
      >
        {{ $t('blupdate.wrongProtocol', { version: appStore.firmwareVersion || '?' }) }}
      </a-alert>

      <!-- Step 1: Read slots from device -->
      <a-space direction="vertical" style="width:100%">
        <a-button type="primary" :loading="state.reading" @click="readSlots">
          <template #icon><icon-refresh /></template>
          {{ $t('blupdate.readSlots') }}
        </a-button>

        <!-- Slot list -->
        <a-table
          v-if="state.slots.length > 0"
          :data="state.slots"
          :pagination="false"
          :bordered="{ cell: true }"
          style="margin-top:12px"
        >
          <template #columns>
            <a-table-column :title="$t('blupdate.colIndex')" data-index="index" :width="60" />
            <a-table-column :title="$t('blupdate.colName')" data-index="name" />
            <a-table-column :title="$t('blupdate.colStart')" data-index="startHex" />
            <a-table-column :title="$t('blupdate.colEnd')" data-index="endHex" />
            <a-table-column :title="$t('blupdate.colSize')" data-index="sizeKb" />
            <a-table-column :title="$t('blupdate.colAction')">
              <template #cell="{ record }">
                <a-button
                  size="small"
                  :type="state.selectedSlot?.index === record.index ? 'primary' : 'outline'"
                  @click="selectSlot(record)"
                >
                  {{ state.selectedSlot?.index === record.index ? $t('blupdate.selected') : $t('blupdate.select') }}
                </a-button>
              </template>
            </a-table-column>
          </template>
        </a-table>

        <!-- Step 2: Select new firmware file -->
        <template v-if="state.selectedSlot">
          <a-divider />
          <div style="display:flex; align-items:center; gap:12px; flex-wrap:wrap">
            <a-button @click="selectFile">
              <template #icon><icon-upload /></template>
              {{ state.newFirmware ? state.newFirmwareName : $t('blupdate.selectFirmware') }}
            </a-button>
            <span v-if="state.newFirmware" style="color: var(--color-text-3); font-size:0.85rem;">
              {{ (state.newFirmware.length / 1024).toFixed(1) }} KB
            </span>
          </div>

          <!-- Size warning -->
          <a-alert
            v-if="state.newFirmware && state.sizeError"
            type="error"
            style="margin-top:8px"
          >
            {{ $t('blupdate.sizeError', { max: ((state.selectedSlot.end - state.selectedSlot.start + 1) / 1024).toFixed(1) }) }}
          </a-alert>

          <!-- Name override -->
          <div v-if="state.newFirmware && !state.sizeError" style="margin-top:12px; display:flex; align-items:center; gap:8px;">
            <span>{{ $t('blupdate.slotName') }}</span>
            <a-input
              v-model="state.newName"
              :max-length="13"
              show-word-limit
              style="width:220px"
              @change="sanitizeName"
            />
          </div>

          <!-- Write button -->
          <a-button
            v-if="state.newFirmware && !state.sizeError"
            type="primary"
            status="danger"
            style="margin-top:16px"
            :loading="state.writing"
            @click="writeSlot"
          >
            <template #icon><icon-thunderbolt /></template>
            {{ $t('blupdate.writeSlot', { index: state.selectedSlot.index, name: state.selectedSlot.name }) }}
          </a-button>
        </template>
      </a-space>
    </a-card>

    <!-- Status log -->
    <div
      id="statusArea"
      style="height:16em; background-color:var(--color-bg-3); color:var(--color-text-3);
             overflow:auto; padding:16px; margin-top:10px; font-family:monospace; font-size:0.82rem;"
      v-html="state.status"
    />
  </div>
</template>

<script lang="ts" setup>
import { reactive } from 'vue';
import { nextTick } from 'vue';
import { useAppStore } from '@/store';
import {
  eeprom_init,
  eeprom_read,
  eeprom_write,
  eeprom_reboot,
  check_eeprom,
  stringToUint8Array,
  hexReverseStringToUint8Array,
  unpack,
} from '@/utils/serial.js';

const appStore = useAppStore();

const state: any = reactive({
  reading: false,
  writing: false,
  slots: [],
  selectedSlot: null,
  newFirmware: null,
  newFirmwareName: '',
  newName: '',
  sizeError: false,
  status: '',
});

// ── helpers ────────────────────────────────────────────────────────────────

function log(msg: string) {
  state.status += msg + '<br/>';
  nextTick(() => {
    const el = document.getElementById('statusArea');
    if (el) el.scrollTop = el.scrollHeight;
  });
}

function errMsg(e: any): string {
  if (e instanceof Error) return e.message;
  if (typeof e === 'string') return e;
  return JSON.stringify(e);
}

// Read arbitrary byte count from EEPROM in 0x40-byte chunks
async function readBytes(addr: number, size: number): Promise<Uint8Array> {
  const out = new Uint8Array(size);
  let offset = 0;
  while (offset < size) {
    const chunk = Math.min(0x40, size - offset);
    const data = await eeprom_read(appStore.connectPort, addr + offset, chunk, appStore.configuration?.uart);
    out.set(data.slice(0, chunk), offset);
    offset += chunk;
  }
  return out;
}

function isEqual(a: Uint8Array, b: Uint8Array): boolean {
  if (a.length !== b.length) return false;
  return a.every((v, i) => v === b[i]);
}

// Read little-endian uint32 from 4-byte array
function readU32LE(buf: Uint8Array, offset: number): number {
  return buf[offset] | (buf[offset+1] << 8) | (buf[offset+2] << 16) | (buf[offset+3] << 24);
}

// ── read slots from EEPROM metadata ────────────────────────────────────────

async function readSlots() {
  if (appStore.connectState !== true) {
    alert(sessionStorage.getItem('noticeConnectK5'));
    return;
  }
  // Metadata lives at 0x40000+ which requires 32-bit addressing (losehu protocol).
  // This is only available when the losehu bootloader is running (hold MENU on power-on).
  if (appStore.configuration?.uart !== 'losehu') {
    log('ERROR: This tool requires the losehu bootloader to be running.');
    log('Power off the radio, hold the MENU button, then power on to enter bootloader mode.');
    log('Then reconnect in k5web — it should show version "L_BL003".');
    return;
  }
  state.reading = true;
  state.slots = [];
  state.selectedSlot = null;
  state.status = '';
  try {
    await eeprom_init(appStore.connectPort);

    // 0x40000: firmware count (1 byte)
    const countBuf = await readBytes(0x40000, 1);
    const count = countBuf[0];
    if (count === 0 || count > 64) {
      log('No valid multiboot metadata found. Have you written firmwares via the Multi-booting page?');
      state.reading = false;
      return;
    }
    log(`Found ${count} firmware slot(s) in metadata.`);

    // 0x40020: slot table, 32 bytes per slot
    // Each entry: 16 bytes name + 4 bytes start + 4 bytes end + 8 bytes padding
    const tableBuf = await readBytes(0x40020, count * 32);

    const slots = [];
    for (let i = 0; i < count; i++) {
      const base = i * 32;
      // Name: up to 13 bytes, null-terminated
      let name = '';
      for (let j = 0; j < 13; j++) {
        if (tableBuf[base + j] === 0) break;
        name += String.fromCharCode(tableBuf[base + j]);
      }
      const start = readU32LE(tableBuf, base + 16);
      const end   = readU32LE(tableBuf, base + 20);
      slots.push({
        index: i + 1,
        name: name || `Slot ${i + 1}`,
        start,
        end,
        startHex: '0x' + start.toString(16).toUpperCase().padStart(5, '0'),
        endHex:   '0x' + end.toString(16).toUpperCase().padStart(5, '0'),
        sizeKb:   ((end - start + 1) / 1024).toFixed(1) + ' KB',
      });
      log(`Slot ${i+1}: "${name}" @ 0x${start.toString(16).toUpperCase()}–0x${end.toString(16).toUpperCase()} (${((end-start+1)/1024).toFixed(1)} KB)`);
    }
    state.slots = slots;
  } catch (e: any) {
    log('Error reading metadata: ' + errMsg(e));
  }
  state.reading = false;
}

// ── slot / file selection ───────────────────────────────────────────────────

function selectSlot(slot: any) {
  state.selectedSlot = slot;
  state.newFirmware = null;
  state.newFirmwareName = '';
  state.newName = slot.name;
  state.sizeError = false;
}

function sanitizeName() {
  state.newName = state.newName.replace(/[^\x00-\x7f]/g, '');
}

function selectFile() {
  const input = document.createElement('input');
  input.type = 'file';
  input.accept = '.bin';
  input.onchange = async () => {
    if (!input.files || input.files.length === 0) return;
    const file = input.files[0];
    const raw = new Uint8Array(await file.arrayBuffer());
    const unpacked = unpack(raw);
    state.newFirmwareName = file.name;
    state.newFirmware = unpacked;
    state.newName = file.name.replace(/\.bin$/i, '').replace(/[^\x00-\x7f]/g, '').substring(0, 13);

    // Check it fits in the existing slot's address range
    const slotCapacity = state.selectedSlot.end - state.selectedSlot.start + 1;
    const needed = Math.ceil(unpacked.length / 0x40) * 0x40;
    state.sizeError = needed > slotCapacity;
    if (state.sizeError) {
      log(`ERROR: firmware is ${(needed/1024).toFixed(1)} KB but slot capacity is ${(slotCapacity/1024).toFixed(1)} KB.`);
    } else {
      log(`Selected: "${file.name}" (${(unpacked.length/1024).toFixed(1)} KB unpacked) → fits in slot ${state.selectedSlot.index}.`);
    }
  };
  input.click();
}

// ── write single slot ───────────────────────────────────────────────────────

async function writeSlot() {
  if (!state.selectedSlot || !state.newFirmware) return;
  if (appStore.connectState !== true) {
    alert(sessionStorage.getItem('noticeConnectK5'));
    return;
  }
  if (appStore.configuration?.uart !== 'losehu') {
    log('ERROR: Bootloader mode required. Power off, hold MENU, power on, then reconnect.');
    return;
  }

  state.writing = true;
  const slot = state.selectedSlot;
  const firmware: Uint8Array = state.newFirmware;

  // Pad firmware to 64-byte boundary with 0xFF
  const padded = new Uint8Array(Math.ceil(firmware.length / 0x40) * 0x40).fill(0xFF);
  padded.set(firmware, 0);

  const newEnd = slot.start + padded.length - 1;

  log(`--- Writing slot ${slot.index} (was "${slot.name}") ---`);
  log(`Target address: 0x${slot.start.toString(16).toUpperCase()}–0x${newEnd.toString(16).toUpperCase()}`);

  try {
    await eeprom_init(appStore.connectPort);

    // Write firmware binary
    for (let i = slot.start; i < slot.start + padded.length; i += 0x40) {
      const chunk = padded.slice(i - slot.start, i - slot.start + 0x40);
      await eeprom_write(appStore.connectPort, i, chunk, chunk.length, appStore.configuration?.uart);
      const verify = await eeprom_read(appStore.connectPort, i, chunk.length, appStore.configuration?.uart);
      if (!isEqual(chunk, verify)) {
        log(`Write error at 0x${i.toString(16).toUpperCase()}, retrying...`);
        i -= 0x40;
        continue;
      }
      const pct = (((i - slot.start) / padded.length) * 100).toFixed(1);
      log(`Writing... ${pct}%`);
    }

    // Update metadata for this slot only:
    //   base = 0x40020 + 32*(slot.index-1)
    //   bytes 0–15:  name (padded with 0x00)
    //   bytes 16–19: start address (little-endian)
    //   bytes 20–23: end address (little-endian)
    //   bytes 24–31: padding 0x00
    const metaBase = 0x40020 + 32 * (slot.index - 1);
    const metaBuf = new Uint8Array(32).fill(0x00);

    const nameBytes = stringToUint8Array(state.newName.substring(0, 13));
    metaBuf.set(nameBytes, 0);

    metaBuf.set(hexReverseStringToUint8Array(slot.start.toString(16)), 16);
    metaBuf.set(hexReverseStringToUint8Array(newEnd.toString(16)), 20);

    // Write metadata in two 0x10-byte chunks (EEPROM write granularity)
    await eeprom_write(appStore.connectPort, metaBase,      metaBuf.slice(0, 0x10), 0x10, appStore.configuration?.uart);
    await eeprom_write(appStore.connectPort, metaBase+0x10, metaBuf.slice(0x10),    0x10, appStore.configuration?.uart);

    log(`Metadata updated: "${state.newName}" @ 0x${slot.start.toString(16).toUpperCase()}–0x${newEnd.toString(16).toUpperCase()}`);
    log('Done! Rebooting radio...');

    await eeprom_reboot(appStore.connectPort);

    // Refresh slot display
    state.selectedSlot = null;
    state.newFirmware = null;
    await readSlots();

  } catch (e: any) {
    log('Error: ' + errMsg(e));
  }
  state.writing = false;
}
</script>

<script lang="ts">
export default { name: 'BLUpdate' };
</script>

<style scoped lang="less">
.container {
  padding: 0 20px 20px 20px;
}
</style>
