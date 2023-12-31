<script setup lang="ts">
import { Icon } from '@/types/general'
import type { AnyDBRecord, DBTable, ParentTable } from '@/types/database'
import useLogger from '@/composables/useLogger'
import useDialogs from '@/composables/useDialogs'
import useRouting from '@/composables/useRouting'
import useUIStore from '@/stores/ui'
import DB from '@/services/Database'

const props = defineProps<{
  parentTable: ParentTable
  record: AnyDBRecord
}>()

const uiStore = useUIStore()
const { log } = useLogger()
const { goToEdit } = useRouting()
const { confirmDialog, dismissDialog, inspectDialog, chartsDialog } = useDialogs()

function viewPreviousNote(note: string) {
  dismissDialog('Previous Note', note, Icon.NOTE)
}

function onToggleFavorite(id: string, name: string, current: boolean) {
  const action = current ? 'Unfavorite' : 'Favorite'
  const message = `Do you want to ${action.toLocaleLowerCase()} ${name}?`
  const icon = current ? Icon.FAVORITE_OFF : Icon.FAVORITE_ON

  confirmDialog(action, message, icon, 'info', async () => {
    try {
      await DB.toggleFavorite(props.parentTable, id)
      log.info(`${name} unfavorited`, { id, name })
    } catch (error) {
      log.error('Unfavorite update failed', error)
    }
  })
}

async function onInspect(id: string) {
  const record = await DB.getRecord(props.parentTable, id)

  if (record) {
    inspectDialog(record, props.parentTable)
  } else {
    log.error('Failed to find record', { id })
  }
}

function onCharts(parentTable: ParentTable, id: string) {
  chartsDialog(id, parentTable)
}

function onDeactivate(table: DBTable, id: string) {
  confirmDialog(
    'Deactivate Record',
    'Would you like to deactivate this record?',
    Icon.CANCEL,
    'warning',
    async () => {
      try {
        await DB.toggleActive(table, id)
        log.info('Deactivated record', { table, id })
      } catch (error) {
        log.error('Deactivation failed', error)
      }
    }
  )
}
</script>

<template>
  <div class="absolute-top-right q-ma-xs">
    <QBtn
      v-show="record.previousChild?.note"
      dense
      round
      flat
      color="primary"
      :icon="Icon.NOTE"
      @click="viewPreviousNote(record?.previousChild?.note || '')"
    />

    <QBtn
      v-if="record.activated"
      dense
      round
      flat
      color="warning"
      :icon="Icon.CANCEL"
      @click="onDeactivate(parentTable, record.id)"
    />

    <QBtn
      v-else
      dense
      round
      flat
      :color="record.favorited ? 'warning' : 'grey'"
      :icon="record.favorited ? Icon.FAVORITE_ON : Icon.FAVORITE_OFF"
      @click="onToggleFavorite(record.id, record.name, record.favorited)"
    />

    <QBtn round flat :icon="Icon.MENU_VERTICAL">
      <QMenu auto-close anchor="top right" transition-show="flip-right" transition-hide="flip-left">
        <QList>
          <QItem clickable @click="onCharts(parentTable, record.id)">
            <QItemSection avatar>
              <QIcon color="accent" :name="Icon.CHARTS" />
            </QItemSection>
            <QItemSection>Charts</QItemSection>
          </QItem>

          <QItem clickable @click="onInspect(record.id)">
            <QItemSection avatar>
              <QIcon color="primary" :name="Icon.INSPECT" />
            </QItemSection>
            <QItemSection>Inspect</QItemSection>
          </QItem>

          <QItem
            :disable="record.activated"
            clickable
            @click="goToEdit(uiStore.dashboardSelection, record.id)"
          >
            <QItemSection avatar>
              <QIcon color="warning" :name="Icon.EDIT" />
            </QItemSection>
            <QItemSection>Edit</QItemSection>
          </QItem>
        </QList>
      </QMenu>
    </QBtn>
  </div>
</template>
