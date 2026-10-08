<script setup lang="ts">
import { computed, onUnmounted, ref, watch } from "vue";
import { listen } from "@tauri-apps/api/event";
import { openUrl } from "@tauri-apps/plugin-opener";
import Icon from "./Icon.vue";
import { useUiStore } from "../stores/ui";
import {
  api,
  invokeError,
  type ReleaseInfo,
  type UpdateProgress,
  type UpdateStatus,
} from "../lib/ipc";
import { translate, updateStatusText as localizedUpdateStatus } from "../lib/i18n";
import { marked } from "marked";
import DOMPurify from "dompurify";
import { registerEscapeCloser } from "../lib/escape";

const props = defineProps<{
  open: boolean;
  info: ReleaseInfo | null;
}>();
const emit = defineEmits<{ close: [] }>();

const ui = useUiStore();
const t = (key: string) => translate(ui.lang, key);

const busy = ref(false);
const progress = ref(0);
const statusText = ref("");
let unlistenProgress: (() => void) | null = null;
let unlistenStatus: (() => void) | null = null;

onUnmounted(() => {
  unlistenProgress?.();
  unlistenStatus?.();
});

watch(
  () => props.open,
  (open) => {
    if (!open) {
      busy.value = false;
      progress.value = 0;
      statusText.value = "";
      unlistenProgress?.();
      unlistenStatus?.();
    }
  }
);

async function downloadAndInstall() {
  if (busy.value || !props.info) return;
  busy.value = true;
  progress.value = 0;
  statusText.value = "";
  try {
    unlistenProgress = await listen<UpdateProgress>("update://progress", (e) => {
      progress.value = e.payload.percent;
    });
    unlistenStatus = await listen<UpdateStatus>("update://status", (e) => {
      statusText.value =
        localizedUpdateStatus(ui.lang, e.payload.code, e.payload.assetName) ||
        `${e.payload.code}${e.payload.assetName ? " — " + e.payload.assetName : ""}`;
    });
  } catch {
    // best-effort
  }
  try {
    await api.downloadAndInstallUpdate();
  } catch (e) {
    ui.toast(invokeError(e).message, "error");
  } finally {
    unlistenProgress?.();
    unlistenStatus?.();
    busy.value = false;
  }
}

function openReleasePage() {
  if (props.info?.releaseUrl) {
    void openUrl(props.info.releaseUrl).catch(() => {});
  }
}

const notesHtml = computed(() => {
  if (!props.info?.notes) return null;
  // Render markdown with `marked` and sanitize with DOMPurify to prevent XSS
  const rawHtml = marked.parse(props.info.notes, { async: false });
  return DOMPurify.sanitize(rawHtml as string);
});

// L19-N1: Escape closes the modal while it is open.
let removeEscapeCloser: (() => void) | null = null;
watch(
  () => props.open,
  (open) => {
    if (open && !removeEscapeCloser) {
      removeEscapeCloser = registerEscapeCloser(() => emit("close"));
    } else if (!open && removeEscapeCloser) {
      removeEscapeCloser();
      removeEscapeCloser = null;
    }
  }
);
onUnmounted(() => removeEscapeCloser?.());
</script>

<style scoped>
/* Markdown content styling for release notes */
.update-notes :deep(h1),
.update-notes :deep(h2),
.update-notes :deep(h3),
.update-notes :deep(h4) {
  font-weight: 600;
  margin-top: 0.75rem;
  margin-bottom: 0.375rem;
  line-height: 1.3;
}
.update-notes :deep(h1) { font-size: 1rem; }
.update-notes :deep(h2) { font-size: 0.875rem; }
.update-notes :deep(h3) { font-size: 0.8125rem; }
.update-notes :deep(h4) { font-size: 0.75rem; }

.update-notes :deep(p) {
  margin-top: 0.375rem;
  margin-bottom: 0.375rem;
}

.update-notes :deep(ul),
.update-notes :deep(ol) {
  padding-left: 1.25rem;
  margin-top: 0.375rem;
  margin-bottom: 0.375rem;
}
.update-notes :deep(li) {
  margin-bottom: 0.1875rem;
}

.update-notes :deep(strong) { font-weight: 600; }
.update-notes :deep(em) { font-style: italic; }

.update-notes :deep(code) {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
  font-size: 0.75rem;
  background: color-mix(in srgb, currentColor 12%, transparent);
  padding: 0.125rem 0.25rem;
  border-radius: 0.25rem;
}
.update-notes :deep(pre) {
  margin: 0.5rem 0;
  padding: 0.5rem;
  border-radius: 0.375rem;
  background: color-mix(in srgb, currentColor 8%, transparent);
  overflow-x: auto;
  font-size: 0.7rem;
  line-height: 1.5;
}
.update-notes :deep(pre code) {
  background: none;
  padding: 0;
  font-size: inherit;
}

.update-notes :deep(blockquote) {
  border-left: 2px solid color-mix(in srgb, currentColor 30%, transparent);
  padding-left: 0.75rem;
  margin: 0.5rem 0;
  color: color-mix(in srgb, currentColor 70%, transparent);
  font-style: italic;
}

.update-notes :deep(a) {
  color: var(--color-primary, #3b82f6);
  text-decoration: underline;
  text-underline-offset: 2px;
}
.update-notes :deep(a:hover) { opacity: 0.8; }

.update-notes :deep(hr) {
  border: none;
  border-top: 1px solid color-mix(in srgb, currentColor 20%, transparent);
  margin: 0.75rem 0;
}
</style>

<template>
  <Teleport to="body">
    <Transition name="modal">
      <div
        v-if="props.open && props.info"
        class="fixed inset-0 z-50 flex items-center justify-center bg-scrim/60 p-4 backdrop-blur-sm"
        @click.self="emit('close')"
      >
        <div class="modal-surface flex max-h-[80vh] w-full max-w-md flex-col">
          <div class="flex items-center justify-between border-b border-line px-5 py-3">
            <h2 class="text-base font-semibold">{{ t("updateAvailable") }}</h2>
            <button
              type="button"
              class="icon-btn !h-7 !w-7"
              :aria-label="t('close')"
              @click="emit('close')"
            >
              <Icon name="close" :size="16" />
            </button>
          </div>

          <div class="min-h-0 flex-1 overflow-y-auto p-5">
            <div class="mb-4 flex items-center gap-3">
              <div class="flex h-10 w-10 shrink-0 items-center justify-center rounded-full bg-primary/15">
                <Icon name="cloud" :size="20" class="text-primary" />
              </div>
              <div class="min-w-0">
                <p class="text-sm font-medium">
                  {{ t("updateNewVersion").replace("{version}", props.info.version) }}
                </p>
                <p class="truncate text-xs text-muted">{{ props.info.name }}</p>
              </div>
            </div>

            <div
              v-if="notesHtml"
              class="mb-4 rounded-md border border-line bg-card/50 p-3 text-xs leading-relaxed text-fg/80 update-notes"
            >
              <p class="mb-2 text-[11px] font-medium uppercase tracking-wide text-muted">
                {{ t("updateReleaseNotes") }}
              </p>
              <div class="max-h-48 overflow-y-auto" v-html="notesHtml"></div>
            </div>

            <button
              v-if="props.info.releaseUrl"
              type="button"
              class="mb-4 flex items-center gap-1.5 text-xs text-primary transition hover:text-primary-hover"
              @click="openReleasePage"
            >
              <Icon name="open" :size="12" />
              {{ t("updateViewOnGitHub") }}
            </button>
          </div>

          <div class="flex items-center gap-2 border-t border-line px-5 py-3">
            <template v-if="busy">
              <div class="min-w-0 flex-1">
                <div class="progress-track">
                  <div class="progress-fill" :style="{ width: Math.min(progress, 100) + '%' }"></div>
                </div>
                <p v-if="statusText" class="mt-1 truncate text-xs text-muted/80">
                  {{ statusText }}
                </p>
              </div>
              <span class="shrink-0 text-xs font-medium text-muted">
                {{ Math.round(Math.min(progress, 100)) }}%
              </span>
            </template>
            <template v-else>
              <button type="button" class="btn btn-outline" @click="emit('close')">
                {{ t("dismiss") }}
              </button>
              <button type="button" class="btn btn-primary" @click="downloadAndInstall">
                {{ t("updateDownloadAndInstall") }}
              </button>
            </template>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>
