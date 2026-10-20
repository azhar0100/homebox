<script setup lang="ts">
  import { ref, reactive, computed, watch, onUnmounted } from 'vue';
  import {
    provideDialogContext,
    type DialogID,
    type DialogParamsMap,
  } from './utils';

  const activeDialog = ref<DialogID | null>(null);
  const activeAlerts = reactive<string[]>([]);
  const openDialogCallbacks = new Map<DialogID, (params: any) => void>();

  // Temporary diagnostics for the isolated navigation reproduction; remove before the final fix.
  let nextRegistrationId = 0;
  const registrationIds = new Map<DialogID, number>();
  function traceDialog(event: string, dialogId: DialogID | null, details: Record<string, unknown>) {
    if (typeof window === "undefined" || localStorage.getItem("homebox:trace-dialogs") !== "1") return;
    console.info("[dialog-trace] " + JSON.stringify({
      event,
      dialogId,
      path: window.location.pathname,
      time: Math.round(performance.now()),
      ...details,
    }));
  }

  watch(activeDialog, (current, previous) => {
    traceDialog("active-change", current, { previousDialogId: previous, activeDialog: current });
  }, { flush: "sync" });

  const removeAfterEach = useRouter().afterEach((to, from, failure) => {
    traceDialog("route", activeDialog.value, {
      fromPath: from.path,
      toPath: to.path,
      activeDialog: activeDialog.value,
    });
    if (!failure) closeDialog();
  });
  onUnmounted(removeAfterEach);

  // onClose for the currently-open dialog (only one dialog can be active)
  let activeOnCloseCallback: ((result?: any) => void) | undefined;

  const registerOpenDialogCallback = <T extends DialogID>(
    dialogId: T,
    callback: (params?: T extends keyof DialogParamsMap ? DialogParamsMap[T] : undefined) => void
  ) => {
    const registrationId = ++nextRegistrationId;
    traceDialog("register", dialogId, { registrationId, replacing: registrationIds.get(dialogId) ?? null });
    registrationIds.set(dialogId, registrationId);
    openDialogCallbacks.set(dialogId, callback as (params: any) => void);
    return () => {
      const currentRegistrationId = registrationIds.get(dialogId) ?? null;
      const ownsCurrentCallback = openDialogCallbacks.get(dialogId) === callback;
      // A replacement may already own this dialog ID when the old component unmounts.
      if (ownsCurrentCallback) {
        openDialogCallbacks.delete(dialogId);
        registrationIds.delete(dialogId);
      }
      traceDialog("cleanup", dialogId, {
        registrationId,
        currentRegistrationId,
        ownsCurrentCallback,
        callbackPresentAfterCleanup: openDialogCallbacks.has(dialogId),
        registrationIdAfterCleanup: registrationIds.get(dialogId) ?? null,
      });
    };
  };

  const openDialog = <T extends DialogID>(dialogId: T, options?: any) => {
    if (activeAlerts.length > 0) return;

    activeDialog.value = dialogId;
    activeOnCloseCallback = options?.onClose;

    const openCallback = openDialogCallbacks.get(dialogId);
    traceDialog("open", dialogId, {
      currentRegistrationId: registrationIds.get(dialogId) ?? null,
      hasCallback: !!openCallback,
    });
    if (openCallback) {
      openCallback(options?.params);
    }
  };

  function closeDialog(dialogId?: DialogID, result?: any) {
    // No dialogId passed -> close current active dialog without result
    if (!dialogId) {
      if (activeDialog.value) {
        // call onClose (if any) with no result
        activeOnCloseCallback?.(undefined);
        activeOnCloseCallback = undefined;
      }
      activeDialog.value = null;
      return;
    }

    // dialogId passed -> if it's the active dialog, call onClose with result
    if (activeDialog.value && activeDialog.value === dialogId) {
      activeOnCloseCallback?.(result);
      activeOnCloseCallback = undefined;
      activeDialog.value = null;
    }
  }

  const addAlert = (alertId: string) => {
    activeAlerts.push(alertId);
  };

  const removeAlert = (alertId: string) => {
    const index = activeAlerts.indexOf(alertId);
    if (index !== -1) activeAlerts.splice(index, 1);
  };

  // Provide context to child components
  provideDialogContext({
    activeDialog: computed(() => activeDialog.value),
    registerOpenDialogCallback,
    openDialog,
    closeDialog,
    activeAlerts: computed(() => activeAlerts),
    addAlert,
    removeAlert,
  });
</script>

<template>
  <slot />
</template>
