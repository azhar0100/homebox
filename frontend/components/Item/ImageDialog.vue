<script setup lang="ts">
  import { computed, getCurrentInstance, watch } from "vue";
  import { useI18n } from "vue-i18n";
  import { Dialog, DialogContent } from "@/components/ui/dialog";
  import { Button, buttonVariants } from "@/components/ui/button";
  import { useDialog } from "@/components/ui/dialog-provider";
  import { DialogID } from "~/components/ui/dialog-provider/utils";
  import { useConfirm } from "@/composables/use-confirm";
  import { toast } from "@/components/ui/sonner";
  import MdiClose from "~icons/mdi/close";
  import MdiDownload from "~icons/mdi/download";
  import MdiDelete from "~icons/mdi/delete";

  const { t } = useI18n();
  const confirm = useConfirm();

  const { activeDialog, closeDialog, registerOpenDialogCallback } = useDialog();
  const viewerId = getCurrentInstance()?.uid ?? null;

  function traceViewer(event: string) {
    if (typeof window === "undefined" || localStorage.getItem("homebox:trace-dialogs") !== "1") return;
    console.info("[dialog-trace] " + JSON.stringify({
      event,
      dialogId: DialogID.ItemImage,
      path: window.location.pathname,
      time: Math.round(performance.now()),
      viewerId,
      activeDialog: activeDialog.value,
      hasOriginalSrc: !!image.originalSrc,
      hasThumbnailSrc: !!image.thumbnailSrc,
    }));
  }

  const api = useUserApi();

  const image = reactive<{
    attachmentId: string;
    itemId: string;
    originalSrc: string;
    originalType?: string;
    thumbnailSrc?: string;
  }>({
    attachmentId: "",
    itemId: "",
    originalSrc: "",
  });

  const isViewerActive = computed(() => activeDialog.value === DialogID.ItemImage);
  watch(isViewerActive, () => traceViewer("viewer-open-state"), { flush: "sync" });

  onMounted(() => {
    traceViewer("viewer-mounted");
    const cleanup = registerOpenDialogCallback(DialogID.ItemImage, params => {
      image.attachmentId = params.attachmentId;
      image.itemId = params.itemId;
      if (params.type === "preloaded") {
        image.originalSrc = params.originalSrc;
        image.originalType = params.originalType;
        image.thumbnailSrc = params.thumbnailSrc;
      } else if (params.type === "attachment") {
        image.originalSrc = api.authURL(`/entities/${params.itemId}/attachments/${params.attachmentId}`);
        image.originalType = params.mimeType;
        image.thumbnailSrc = params.thumbnailId
          ? api.authURL(`/entities/${params.itemId}/attachments/${params.thumbnailId}`)
          : image.originalSrc;
      }
      traceViewer("viewer-params");
    });

    onUnmounted(() => {
      traceViewer("viewer-unmounted");
      cleanup();
    });
  });

  async function deleteAttachment() {
    const confirmed = await confirm.open(t("items.delete_attachment_confirm"));

    if (confirmed.isCanceled) {
      return;
    }

    const { error } = await api.items.attachments.delete(image.itemId, image.attachmentId);

    if (error) {
      toast.error(t("items.toast.failed_delete_attachment"));
      return;
    }

    closeDialog(DialogID.ItemImage, {
      action: "delete",
      id: image.attachmentId,
    });
    toast.success(t("items.toast.attachment_deleted"));
  }
</script>

<template>
  <Dialog :dialog-id="DialogID.ItemImage">
    <DialogContent
      class="max-h-[90svh] w-auto max-w-[min(calc(100vw_-_1rem),32rem)] border-transparent bg-transparent p-0 md:max-w-lg"
      disable-close
    >
      <picture>
        <source :srcset="image.originalSrc" :type="image.originalType" />
        <img
          :src="image.thumbnailSrc"
          alt="attachment image"
          class="min-w-64 max-w-[min(calc(100vw_-_1rem),32rem)] md:w-auto md:max-w-lg"
        />
      </picture>
      <Button variant="destructive" size="icon" class="absolute right-[84px] top-1" @click="deleteAttachment">
        <MdiDelete />
      </Button>
      <a :class="buttonVariants({ size: 'icon' })" :href="image.originalSrc" download class="absolute right-11 top-1">
        <MdiDownload />
      </a>
      <Button
        size="icon"
        class="absolute right-1 top-1"
        @click="
          closeDialog(DialogID.ItemImage);
          image.originalSrc = '';
        "
      >
        <MdiClose />
      </Button>
    </DialogContent>
  </Dialog>
</template>
