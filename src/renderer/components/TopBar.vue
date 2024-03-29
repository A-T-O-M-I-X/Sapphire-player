<script lang="ts" setup>
import { amethyst, useState } from "@/amethyst";
import ControlButtons from "@/components/input/ControlButtons.vue";
import UpdateButton from "@/components/input/UpdateButton.vue";
import Menu from "@/components/menu/MenuContainer.vue";
import MenuOption from "@/components/menu/MenuOption.vue";
import MenuSplitter from "@/components/menu/MenuSplitter.vue";
import ProcessorUsageMeter from "@/components/ProcessorUsageMeter.vue";
import { AudioFileIcon, DiscordIcon, GitHubIcon, MusicFolderIcon, ResetIcon, ZoomInIcon, ZoomOutIcon, RemoveIcon, ResizeIcon, DownloadingUpdatesIcon, SettingsIcon, BookshelfIcon, LoadingIcon } from "@/icons/material";
import { useFps } from "@vueuse/core";
import { player } from "@/logic/player";
import AmethystLogo from "@/icons/AmethystLogo.vue";
import { onMounted, ref } from "vue";
import { countDomElements, refreshWindow } from "@/logic/dom";
import BaseChip from "./BaseChip.vue";

const min = ref(Number.POSITIVE_INFINITY);
const max = ref(Number.NEGATIVE_INFINITY);
const fpsCounter = useFps({every: 60});
const fps = ref(0);
const domSize = ref(0);
const latency = ref(0);
const cpuUsage = ref({
  node: 0,
  renderer: 0
});

type ProcessorUsage = {node: number, renderer: number};

onMounted(() => {
  setInterval(() => {
    fps.value = fpsCounter.value;
    if (fps.value > max.value) max.value = fps.value;
    if (fps.value < min.value) min.value = fps.value;
    domSize.value = countDomElements();
    player.getLatency().then(l => latency.value = l);
    // TODO: multiplatform support
    window.electron.ipcRenderer.invoke<ProcessorUsage>("percent-cpu-usage").then(usage => cpuUsage.value = usage);
  }, 1000);
});

// TODO: move this to queue or something

const state = useState();
// const electron = useElectron();
</script>

<template>
  <div
    class="borderBottom z-100 font-main drag text-12px select-none flex justify-between items-center"
    :class="[state.state.isFocused ? 'text-primary-1000' : 'text-primary-900']"
  >
    <div
      class="flex no-drag h-full items-center"
    
      :class="[amethyst.getCurrentOperatingSystem() == 'mac' && 'pl-16']"
    >
      <div
        class="logo w-40px items-center flex justify-center cursor-heart-pointer"
      >
        <amethyst-logo
          class="w-4 h-4 min-h-4 min-w-4"
        />
      </div>
      <Menu title="File">
        <menu-option
          :shortcuts="['CTRL', 'O']"
          title="Open audio..."
          :icon="AudioFileIcon"
          @click="amethyst.openAudioFilesAndAddToQueue"
        />
        <menu-option
          :shortcuts="['CTRL', 'SHIFT', 'O']"
          title="Open audio folder..."
          :icon="MusicFolderIcon"
          @click="amethyst.openAudioFoldersAndAddToQueue"
        />
      </Menu>
      <Menu title="Utility">
        <menu-option
          title="Zoom in"
          :icon="ZoomInIcon"
          :shortcuts="['CTRL', 'SHIFT', '+']"
        />
        <menu-option
          title="Zoom out"
          :icon="ZoomOutIcon"
          :shortcuts="['CTRL', '-']"
        />
        <menu-option
          title="Reset zoom"
          :icon="ResizeIcon"
          :shortcuts="['CTRL', '0']"
        />

        <menu-splitter />
        <menu-option
          :shortcuts="['CTRL', 'SHIFT', 'X']"
          title="Clear queue"
          :icon="RemoveIcon"
          @click="player.queue.clear()"
        />
        <menu-option
          :shortcuts="['CTRL', 'SHIFT', 'Z']"
          title="Clear errored / deleted"
          :icon="RemoveIcon"
          @click="player.queue.clearErrored()"
        />
        <menu-splitter />
        <menu-option
          :shortcuts="['CTRL', 'ALT', 'R']"
          title="Refresh all metadata"
          :icon="ResetIcon"
          @click="player.queue.fetchAsyncData(true)"
        />
        <menu-option
          :shortcuts="['CTRL', 'R']"
          title="Refresh window"
          :icon="ResetIcon"
          @click="refreshWindow"
        />

        <menu-splitter 
          v-if="amethyst.getCurrentPlatform() === 'desktop'"
        />
      </Menu>
      <Menu title="View">
        <menu-option
          title="Settings"
          :icon="SettingsIcon"
          @click="$router.push({ name: 'settings.appearance' })"
        />
      </Menu>
      <Menu
        v-if="amethyst.IS_DEV"
        title="Debug"
      >
        <menu-option
          title="Set 'updateReady' to 'true'"
          @click="state.state.updateReady = true;"
        />
        <menu-option
          title="Set 'updateReady' to 'false'"
          @click="state.state.updateReady = false;"
        />
      </Menu>
    </div>

    <p class="absolute flex items-center gap-1 left-1/2 transform-gpu -translate-x-1/2">
      <LoadingIcon
        v-if="state.state.isCheckingForUpdates"
        class="h-3 animate-spin w-3 min-h-3 min-w-3"
      />
      Sapphire 
      <strong class="opacity-50 font-normal capitalize">{{ amethyst.getCurrentPlatform() }}</strong>
      <BaseChip v-if="amethyst.IS_DEV">
        dev
      </BaseChip>
      <strong class="opacity-50 font-normal">v{{ amethyst.VERSION }}</strong>
    </p>

    <div class="flex gap-1.25 h-6 items-center overflow-hidden font-aseprite whitespace-nowrap">
      <div
        v-if="state.settings.value.showDebugStats"
        class="w-56 flex gap-1 justify-end no-drag" 
        @click="min = Number.POSITIVE_INFINITY; max = Number.NEGATIVE_INFINITY;"
      >
        <div class="hidden lg:inline font-aseprite text-primary-900 text-opacity-50">
          {{ domSize }}<strong class="text-primary-900 text-opacity-25">DOM </strong>
          {{ player.getBufferSize() }}<strong class="text-primary-900 text-opacity-25">smp</strong>
          {{ latency.toFixed(2) }}<strong class="text-primary-900 text-opacity-25">ms</strong>
        </div>
        <div 
          :class="[
            fps < 30 && 'text-rose-500',
            fps >= 30 && fps < max && 'text-yellow-300',
            fps >= (max*0.8) && 'text-green-500',
          ]"
          class="font-aseprite"
        >
          {{ fps }}fps
        </div>
        <div
          class="hidden lg:inline font-aseprite text-primary-900 text-opacity-50"
        >
          {{ min }}<strong class="text-primary-900 text-opacity-25">min</strong> {{ max }}<strong class="text-primary-900 text-opacity-25">max</strong>
        </div>
      </div>
    
      <update-button
        v-if="state.state.updateReady"
        @click="amethyst.performWindowAction('close')"
      />
        
      <template v-if="state.settings.value.showDebugStats">
        <processor-usage-meter
          v-for="value of Object.values(cpuUsage)"
          :key="value"
          :value="value"
        />
      </template>
      <control-buttons
        v-if="amethyst.getCurrentPlatform() === 'desktop' && amethyst.getCurrentOperatingSystem() != 'mac'"
        :is-maximized="state.state.isMaximized"
        @close="amethyst.performWindowAction('close')"
        @minimize="amethyst.performWindowAction('minimize')"
        @maximize="amethyst.performWindowAction('maximize')"
        @unmaximize="amethyst.performWindowAction('unmaximize')"
      />
    </div>
  </div>
</template>

<style lang="postcss">
select::-ms-expand {
  display: none;
}

.logo img {
  @apply transform active: rotate-360 active:scale-50 transition duration-200 ml-1 h-4;
}
</style>
