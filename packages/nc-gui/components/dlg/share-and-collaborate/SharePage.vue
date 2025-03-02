<script lang="ts" setup>
import type { ColumnType, KanbanType, ViewType } from 'nocodb-sdk'
import { ViewTypes } from 'nocodb-sdk'
import { PreFilledMode, useMetas } from '#imports'

const { view: _view, $api } = useSmartsheetStoreOrThrow()
const { $e } = useNuxtApp()
const { getBaseUrl, appInfo } = useGlobal()

const { dashboardUrl } = useDashboard()

const viewStore = useViewsStore()

const { metas } = useMetas()

const workspaceStore = useWorkspace()

const isLocked = inject(IsLockedInj, ref(false))

const isUpdating = ref({
  public: false,
  password: false,
  download: false,
})

const activeView = computed<(ViewType & { meta: object & Record<string, any> }) | undefined>({
  get: () => {
    if (typeof _view.value?.meta === 'string') {
      _view.value.meta = JSON.parse(_view.value.meta)
    }

    return _view.value as ViewType & { meta: object }
  },
  set: (value: ViewType | undefined) => {
    if (typeof _view.value?.meta === 'string') {
      _view.value!.meta = JSON.parse((_view.value.meta as string)!)
    }

    if (typeof value?.meta === 'string') {
      value!.meta = JSON.parse(value.meta as string)
    }

    _view.value = value
  },
})

const isPublicShared = computed(() => {
  return !!activeView.value?.uuid
})

const url = computed(() => {
  return sharedViewUrl() ?? ''
})

const passwordProtectedLocal = ref(false)

const passwordProtected = computed(() => {
  return !!activeView.value?.password || passwordProtectedLocal.value
})

const password = computed({
  get: () => (passwordProtected.value ? activeView.value?.password ?? '' : ''),
  set: async (value) => {
    if (!activeView.value) return

    activeView.value = { ...(activeView.value as any), password: passwordProtected.value ? value : null }

    updateSharedView()
  },
})

const viewTheme = computed({
  get: () => !!activeView.value?.meta.withTheme,
  set: (withTheme) => {
    if (!activeView.value?.meta) return

    activeView.value.meta = {
      ...activeView.value.meta,
      withTheme,
    }
    saveTheme()
  },
})

const togglePasswordProtected = async () => {
  passwordProtectedLocal.value = !passwordProtected.value
  if (!activeView.value) return
  if (isUpdating.value.password) return

  isUpdating.value.password = true
  try {
    if (passwordProtected.value) {
      activeView.value = { ...(activeView.value as any), password: null }
    } else {
      activeView.value = { ...(activeView.value as any), password: '' }
    }

    await updateSharedView()
  } finally {
    isUpdating.value.password = false
  }
}

const withRTL = computed({
  get: () => {
    if (!activeView.value?.meta) return false

    if (typeof activeView.value?.meta === 'string') {
      activeView.value.meta = JSON.parse(activeView.value.meta)
    }

    return !!(activeView.value?.meta as any)?.rtl
  },
  set: (rtl) => {
    if (!activeView.value?.meta) return

    if (typeof activeView.value?.meta === 'string') {
      activeView.value.meta = JSON.parse(activeView.value.meta)
    }

    activeView.value.meta = { ...(activeView.value.meta as any), rtl }
    updateSharedView()
  },
})

const allowCSVDownload = computed({
  get: () => !!(activeView.value?.meta as any)?.allowCSVDownload,
  set: async (allow) => {
    if (!activeView.value?.meta) return

    isUpdating.value.download = true

    try {
      activeView.value.meta = { ...activeView.value.meta, allowCSVDownload: allow }
      await saveAllowCSVDownload()
    } finally {
      isUpdating.value.download = false
    }
  },
})

const surveyMode = computed({
  get: () => !!activeView.value?.meta.surveyMode,
  set: (survey) => {
    if (!activeView.value?.meta) return

    activeView.value.meta = { ...activeView.value.meta, surveyMode: survey }
    saveSurveyMode()
  },
})

const formPreFill = computed({
  get: () => ({
    preFillEnabled: parseProp(activeView.value?.meta)?.preFillEnabled ?? false,
    preFilledMode: parseProp(activeView.value?.meta)?.preFilledMode || PreFilledMode.Default,
  }),
  set: (value) => {
    if (!activeView.value?.meta) return

    if (formPreFill.value.preFillEnabled !== value.preFillEnabled) {
      $e(`a:view:share:prefilled-mode-${value.preFillEnabled ? 'enabled' : 'disabled'}`)
    }

    if (formPreFill.value.preFilledMode !== value.preFilledMode) {
      $e(`a:view:share:${value.preFilledMode}-prefilled-mode`)
    }

    activeView.value.meta = {
      ...activeView.value.meta,
      ...value,
    }
    savePreFilledMode()
  },
})

const handleChangeFormPreFill = (value: { preFillEnabled?: boolean; preFilledMode?: PreFilledMode }) => {
  formPreFill.value = {
    ...formPreFill.value,
    ...value,
  }
}

function sharedViewUrl() {
  if (!activeView.value) return

  let viewType
  switch (activeView.value.type) {
    case ViewTypes.FORM:
      viewType = 'form'
      break
    case ViewTypes.KANBAN:
      viewType = 'kanban'
      break
    case ViewTypes.GALLERY:
      viewType = 'gallery'
      break
    case ViewTypes.MAP:
      viewType = 'map'
      break
    case ViewTypes.CALENDAR:
      viewType = 'calendar'
      break
    default:
      viewType = 'view'
  }

  // get base url for workspace
  const baseUrl = getBaseUrl(workspaceStore.activeWorkspaceId)

  let dashboardUrl1 = dashboardUrl.value
  if (baseUrl) {
    dashboardUrl1 = `${baseUrl}${appInfo.value?.dashboardPath}`
  }

  return encodeURI(
    `${dashboardUrl1}#/nc/${viewType}/${activeView.value.uuid}${surveyMode.value ? '/survey' : ''}${
      viewStore.preFillFormSearchParams && formPreFill.value.preFillEnabled ? `?${viewStore.preFillFormSearchParams}` : ''
    }`,
  )
}

const toggleViewShare = async () => {
  if (!activeView.value?.id) return

  if (activeView.value?.uuid) {
    await $api.dbViewShare.delete(activeView.value.id)

    activeView.value = { ...activeView.value, uuid: undefined, password: undefined }
  } else {
    const response = await $api.dbViewShare.create(activeView.value.id)
    activeView.value = { ...activeView.value, ...(response as any) }

    if (activeView.value!.type === ViewTypes.KANBAN) {
      // extract grouping column meta
      const groupingFieldColumn = metas.value[viewStore.activeView!.fk_model_id].columns!.find(
        (col: ColumnType) => col.id === ((viewStore.activeView!.view! as KanbanType).fk_grp_col_id! as string),
      )

      activeView.value!.meta = { ...activeView.value!.meta, groupingFieldColumn }

      await updateSharedView()
    }
  }
}

const toggleShare = async () => {
  if (isUpdating.value.public) return

  isUpdating.value.public = true
  try {
    return await toggleViewShare()
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  } finally {
    isUpdating.value.public = false
  }
}

async function saveAllowCSVDownload() {
  isUpdating.value.download = true
  try {
    await updateSharedView()
    $e(`a:view:share:${allowCSVDownload.value ? 'enable' : 'disable'}-csv-download`)
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  }
  isUpdating.value.download = false
}

async function saveSurveyMode() {
  await updateSharedView()
  $e(`a:view:share:${surveyMode.value ? 'enable' : 'disable'}-survey-mode`)
}

async function saveTheme() {
  await updateSharedView()
  $e(`a:view:share:${viewTheme.value ? 'enable' : 'disable'}-theme`)
}

async function updateSharedView() {
  try {
    if (!activeView.value?.meta) return
    const meta = activeView.value.meta

    await $api.dbViewShare.update(activeView.value.id!, {
      meta,
      password: activeView.value.password,
    })
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  }

  return true
}

async function savePreFilledMode() {
  await updateSharedView()
}

watchEffect(() => {})
</script>

<template>
  <div class="flex flex-col py-2 px-3 mb-1">
    <div class="flex flex-col w-full mt-2.5 px-3 py-2.5 border-gray-200 border-1 rounded-md gap-y-2">
      <div class="flex flex-row w-full justify-between py-0.5">
        <div class="text-gray-900 font-medium">{{ $t('activity.enabledPublicViewing') }}</div>
        <a-switch
          v-e="['c:share:view:enable:toggle']"
          :checked="isPublicShared"
          :disabled="isLocked"
          :loading="isUpdating.public"
          class="share-view-toggle !mt-0.25"
          data-testid="share-view-toggle"
          @click="toggleShare"
        />
      </div>
      <template v-if="isPublicShared">
        <div class="mt-0.5 border-t-1 border-gray-100 pt-3">
          <GeneralCopyUrl v-model:url="url" />
        </div>
        <div class="flex flex-col justify-between mt-1 py-2 px-3 bg-gray-50 rounded-md">
          <div class="flex flex-row items-center justify-between">
            <div class="flex text-black">{{ $t('activity.restrictAccessWithPassword') }}</div>
            <a-switch
              v-e="['c:share:view:password:toggle']"
              :checked="passwordProtected"
              :loading="isUpdating.password"
              class="share-password-toggle !mt-0.25"
              data-testid="share-password-toggle"
              size="small"
              @click="togglePasswordProtected"
            />
          </div>
          <Transition mode="out-in" name="layout">
            <div v-if="passwordProtected" class="flex gap-2 mt-2 w-2/3">
              <a-input-password
                v-model:value="password"
                :placeholder="$t('placeholder.password.enter')"
                class="!rounded-lg !py-1 !bg-white"
                data-testid="nc-modal-share-view__password"
                size="small"
                type="password"
              />
            </div>
          </Transition>
        </div>
        <div class="flex flex-col justify-between gap-y-3 mt-1 py-2 px-3 bg-gray-50 rounded-md">
          <div
            v-if="
              activeView &&
              [ViewTypes.GRID, ViewTypes.KANBAN, ViewTypes.GALLERY, ViewTypes.MAP, ViewTypes.CALENDAR].includes(activeView.type)
            "
            class="flex flex-row items-center justify-between"
          >
            <div class="flex text-black">{{ $t('activity.allowDownload') }}</div>
            <a-switch
              v-model:checked="allowCSVDownload"
              v-e="['c:share:view:allow-csv-download:toggle']"
              :loading="isUpdating.download"
              class="public-password-toggle !mt-0.25"
              data-testid="share-download-toggle"
              size="small"
            />
          </div>

          <template v-if="activeView?.type === ViewTypes.FORM">
            <div class="flex flex-row items-center justify-between">
              <div class="text-black flex items-center space-x-1">
                <div>
                  {{ $t('activity.surveyMode') }}
                </div>
                <NcTooltip class="flex items-center">
                  <template #title> {{ $t('tooltip.surveyFormInfo') }}</template>
                  <GeneralIcon icon="info" class="flex-none text-gray-600 cursor-pointer"></GeneralIcon>
                </NcTooltip>
              </div>
              <a-switch
                v-model:checked="surveyMode"
                v-e="['c:share:view:surver-mode:toggle']"
                data-testid="nc-modal-share-view__surveyMode"
                size="small"
              >
              </a-switch>
            </div>
            <div v-if="!isEeUI" class="flex flex-row items-center justify-between">
              <div class="text-black">{{ $t('activity.rtlOrientation') }}</div>
              <a-switch
                v-model:checked="withRTL"
                v-e="['c:share:view:rtl-orientation:toggle']"
                data-testid="nc-modal-share-view__RTL"
                size="small"
              >
              </a-switch>
            </div>
          </template>
        </div>
        <div
          v-if="activeView?.type === ViewTypes.FORM"
          class="nc-pre-filled-mode-wrapper flex flex-col justify-between gap-y-3 mt-1 py-2 px-3 bg-gray-50 rounded-md"
        >
          <div class="flex flex-row items-center justify-between">
            <div class="text-black flex items-center space-x-1">
              <div>
                {{ $t('activity.preFilledFields.title') }}
              </div>

              <NcTooltip class="flex items-center">
                <template #title>
                  <div class="text-center">
                    {{ $t('tooltip.preFillFormInfo') }}
                  </div>
                </template>
                <GeneralIcon icon="info" class="flex-none text-gray-600 cursor-pointer"></GeneralIcon>
              </NcTooltip>
            </div>
            <a-switch
              v-e="['c:share:view:surver-mode:toggle']"
              :checked="formPreFill.preFillEnabled"
              data-testid="nc-modal-share-view__preFill"
              size="small"
              @update:checked="handleChangeFormPreFill({ preFillEnabled: $event as boolean })"
            >
            </a-switch>
          </div>

          <a-radio-group
            v-if="formPreFill.preFillEnabled"
            :value="formPreFill.preFilledMode"
            class="nc-modal-share-view-preFillMode"
            data-testid="nc-modal-share-view__preFillMode"
            @update:value="handleChangeFormPreFill({ preFilledMode: $event })"
          >
            <a-radio v-for="mode of Object.values(PreFilledMode)" :key="mode" :value="mode">
              <div class="flex-1">{{ $t(`activity.preFilledFields.${mode}`) }}</div>
            </a-radio>
          </a-radio-group>
        </div>
      </template>
    </div>
  </div>
</template>

<style lang="scss">
.docs-share-public-toggle {
  height: 1.25rem !important;
  min-width: 2.4rem !important;
  width: 2.4rem !important;
  line-height: 1rem;

  .ant-switch-handle {
    height: 1rem !important;
    min-width: 1rem !important;
    line-height: 0.8rem !important;
  }
  .ant-switch-inner {
    height: 1rem !important;
    min-width: 1rem !important;
    line-height: 1rem !important;
  }
}

.nc-modal-share-view-preFillMode {
  @apply flex flex-col;

  .ant-radio-wrapper {
    @apply !m-0 !flex !items-center w-full px-2 py-1 rounded-lg hover:bg-gray-100;
    .ant-radio {
      @apply !top-0;
    }
    .ant-radio + span {
      @apply !flex !pl-4;
    }
  }
}
</style>
