<template>
  <div
    class="flex items-stretch group"
    @contextmenu.prevent="options!.tippy.show()"
  >
    <span
      class="flex items-center justify-center px-4 cursor-pointer"
      @click="emit('edit-environment')"
    >
      <icon-lucide-layers class="svg-icons" />
    </span>
    <span
      class="flex flex-1 min-w-0 py-2 pr-2 cursor-pointer transition group-hover:text-secondaryDark"
      @click="emit('edit-environment')"
    >
      <span class="truncate">
        {{ environment.environment.name }}
      </span>
    </span>
    <span>
      <tippy
        v-if="!isViewer"
        ref="options"
        interactive
        trigger="click"
        theme="popover"
        :on-shown="() => tippyActions!.focus()"
      >
        <ButtonSecondary
          v-tippy="{ theme: 'tooltip' }"
          :title="t('action.more')"
          :icon="IconMoreVertical"
        />
        <template #content="{ hide }">
          <div
            ref="tippyActions"
            class="flex flex-col focus:outline-none"
            tabindex="0"
            role="menu"
            @keyup.e="edit!.$el.click()"
            @keyup.d="duplicate!.$el.click()"
            @keyup.delete="deleteAction!.$el.click()"
            @keyup.escape="options!.tippy().hide()"
          >
            <SmartItem
              ref="edit"
              :icon="IconEdit"
              :label="`${t('action.edit')}`"
              :shortcut="['E']"
              @click="
                () => {
                  emit('edit-environment')
                  hide()
                }
              "
            />
            <SmartItem
              ref="duplicate"
              :icon="IconCopy"
              :label="`${t('action.duplicate')}`"
              :shortcut="['D']"
              @click="
                () => {
                  duplicateEnvironments()
                  hide()
                }
              "
            />
            <SmartItem
              ref="deleteAction"
              :icon="IconTrash2"
              :label="`${t('action.delete')}`"
              :shortcut="['⌫']"
              @click="
                () => {
                  confirmRemove = true
                  hide()
                }
              "
            />
          </div>
        </template>
      </tippy>
    </span>
    <SmartConfirmModal
      :show="confirmRemove"
      :title="`${t('confirm.remove_environment')}`"
      @hide-modal="confirmRemove = false"
      @resolve="removeEnvironment"
    />
  </div>
</template>

<script setup lang="ts">
import { ref } from "vue"
import { pipe } from "fp-ts/function"
import * as TE from "fp-ts/TaskEither"
import { useToast } from "@composables/toast"
import { useI18n } from "~/composables/i18n"
import {
  deleteTeamEnvironment,
  createDuplicateEnvironment as duplicateEnvironment,
} from "~/helpers/backend/mutations/TeamEnvironment"
import { GQLError } from "~/helpers/backend/GQLClient"
import { TeamEnvironment } from "~/helpers/teams/TeamEnvironment"
import IconEdit from "~icons/lucide/edit"
import IconCopy from "~icons/lucide/copy"
import IconTrash2 from "~icons/lucide/trash-2"
import IconMoreVertical from "~icons/lucide/more-vertical"
import { TippyComponent } from "vue-tippy"
import SmartItem from "@components/smart/Item.vue"

const t = useI18n()
const toast = useToast()

const props = defineProps<{
  environment: TeamEnvironment
  isViewer: boolean
}>()

const emit = defineEmits<{
  (e: "edit-environment"): void
}>()

const confirmRemove = ref(false)

const tippyActions = ref<TippyComponent | null>(null)
const options = ref<TippyComponent | null>(null)
const edit = ref<typeof SmartItem | null>(null)
const duplicate = ref<typeof SmartItem | null>(null)
const deleteAction = ref<typeof SmartItem | null>(null)

const removeEnvironment = () => {
  pipe(
    deleteTeamEnvironment(props.environment.id),
    TE.match(
      (err: GQLError<string>) => {
        console.error(err)
        toast.error(`${getErrorMessage(err)}`)
      },
      () => {
        toast.success(`${t("team_environment.deleted")}`)
      }
    )
  )()
}

const duplicateEnvironments = () => {
  pipe(
    duplicateEnvironment(props.environment.id),
    TE.match(
      (err: GQLError<string>) => {
        console.error(err)
        toast.error(`${getErrorMessage(err)}`)
      },
      () => {
        toast.success(`${t("team_environment.duplicate")}`)
      }
    )
  )()
}

const getErrorMessage = (err: GQLError<string>) => {
  if (err.type === "network_error") {
    return t("error.network_error")
  } else {
    switch (err.error) {
      case "team_environment/not_found":
        return t("team_environment.not_found")
      default:
        return t("error.something_went_wrong")
    }
  }
}
</script>
