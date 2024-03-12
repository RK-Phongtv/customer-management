<script setup>
import { useCustomerStore } from '@/stores/customers'
import ConfirmDialog from '@/components/ConfirmDialog.vue'
import { storeToRefs } from 'pinia'
import { useI18n } from 'vue-i18n'
import { LOCALE_KEY, CONSTANT } from '@/constants'
import { ref } from 'vue'
import Loading from 'vue3-loading-overlay'
import 'vue3-loading-overlay/dist/vue3-loading-overlay.css'
import { showErrorMsg, getPageText, getHeaders, showSuccessMsg } from '@/utils'
import UserModel from '@/components/UserModel.vue'
import TextField from '@/components/TextField.vue'
import Button from '@/components/Button.vue'
import Chip from '@/components/Chip.vue'

const { t } = useI18n()
const useStore = useCustomerStore()
const { userList, totalElements, hasError, isLoading } = storeToRefs(useStore)
const { getCustomersAction, removeCustomerAction } = useStore
const itemSelected = ref([])
const isRemoveMul = ref(false)
const itemsPerPage = ref(15)
const selectedItems = ref([])
const refConfirmDialog = ref(null)
const refUserModel = ref(null)
const options = ref({ page: 1 })
const userName = ref('')
const tags = ref('')
const pageText = ref('')

async function loadItems({ page, itemsPerPage, sortBy }) {
  options.value.page = page
  options.value.sortBy = sortBy[0]

  await getCustomersAction({
    name: userName.value,
    tags: tags.value,
    page: page - 1,
    sortBy: `${sortBy[0]?.key || CONSTANT.CUSTOMER_ID_KEY}:${sortBy[0]?.order || CONSTANT.ORDER}`,
    limit: itemsPerPage
  })

  if (hasError.value) {
    showErrorMsg(t(LOCALE_KEY.MESSAGE_ERROR))
  }

  pageText.value = getPageText(
    itemsPerPage,
    options.value.page,
    totalElements.value,
    t(LOCALE_KEY.PAGE_TEXT)
  )
}

async function onRemoveItem(item) {
  let msg = t(LOCALE_KEY.REMOVE_MULTIPLE_ITEMS_CONFIRM_MSG)

  if (item) {
    isRemoveMul.value = false
    itemSelected.value = item
    msg = t(LOCALE_KEY.REMOVE_ITEM_CONFIRM_MSG)
  } else {
    isRemoveMul.value = true
  }

  await refConfirmDialog.value.onOpen(t(LOCALE_KEY.TITLE_DIALOG_CONFIRM), msg)
}

async function onAgree() {
  await removeCustomerAction(
    !isRemoveMul.value ? [itemSelected.value.customer_id] : selectedItems.value
  )

  if (hasError.value) {
    showErrorMsg(t(LOCALE_KEY.MESSAGE_ERROR))
  } else {
    showSuccessMsg(t(LOCALE_KEY.REMOVE_ITEM_SUCCESS_MSG))

    if (isRemoveMul.value) selectedItems.value = []

    onReloadTable()
  }
}

function onClearSearchCondition() {
  userName.value = ''
  tags.value = ''
}

function onSearch() {
  loadItems({
    page: 1,
    itemsPerPage: itemsPerPage.value,
    sortBy: [{ key: CONSTANT.CUSTOMER_ID_KEY, order: CONSTANT.ORDER }]
  })
}

async function onUpdateItem(item) {
  await refUserModel.value.onOpen(item)
}

async function onOpenAddDialog() {
  await refUserModel.value.onOpen()
}

async function onReloadTable() {
  if (options.value.page !== 1) {
    options.value.page = 1
  } else {
    let sortBy = options.value.sortBy

    sortBy = [
      {
        key: !sortBy ? CONSTANT.CUSTOMER_ID_KEY : sortBy.key,
        order: !sortBy ? CONSTANT.ORDER : sortBy.order
      }
    ]

    await loadItems({
      page: 1,
      itemsPerPage: itemsPerPage.value,
      sortBy
    })
  }
}
</script>

<template>
  <v-container>
    <v-label class="my-2">
      <v-icon>mdi-account-search</v-icon>
      <span>{{ $t(LOCALE_KEY.FIND_USER_LABEL) }}</span>
    </v-label>
    <v-row>
      <v-col cols="12" sm="5">
        <TextField v-model:value="userName" :label="$t(LOCALE_KEY.USER_NAME_LABEL)" hide-details />
      </v-col>
      <v-col cols="12" sm="5">
        <TextField
          v-model:value="tags"
          :label="$t(LOCALE_KEY.TAGS)"
          :hint="$t(LOCALE_KEY.NOTICE_TAGS)"
        />
      </v-col>
      <v-col cols="12" sm="2" class="d-flex">
        <Button icon="mdi-magnify" size="small" classProp="blue-bg text-white" @on-click="onSearch">
          <v-icon>mdi-magnify</v-icon>
        </Button>
        <Button
          icon="mdi-close"
          size="small"
          classProp="grey-bg text-white ml-1"
          @on-click="onClearSearchCondition"
        >
          <v-icon>mdi-close</v-icon>
        </Button>
      </v-col>
    </v-row>
    <div class="my-6">
      <div cols="12" sm="12" class="d-flex align-center">
        <Button classProp="blue-bg text-white text-capitalize" @on-click="onOpenAddDialog">
          <v-icon color="white" size="small">mdi-plus-circle</v-icon>
          <span class="ml-1">{{ $t(LOCALE_KEY.ADD_BTN) }}</span>
        </Button>

        <Button
          :disabled="!selectedItems.length"
          classProp="red-bg text-white text-capitalize ml-2"
          @on-click="onRemoveItem"
        >
          <v-icon color="white" size="small">mdi-delete-circle</v-icon>
          <span class="ml-1">{{ $t(LOCALE_KEY.REMOVE_BTN) }}</span>
        </Button>
      </div>
    </div>
    <v-data-table-server
      v-model="selectedItems"
      fixed-header
      height="600"
      v-model:items-per-page="itemsPerPage"
      :item-value="CONSTANT.CUSTOMER_ID_KEY"
      :item-key="CONSTANT.CUSTOMER_ID_KEY"
      @update:options="loadItems"
      show-select
      :headers="getHeaders(t)"
      :items="userList"
      :items-length="totalElements"
      :loading="isLoading"
      :no-data-text="$t(LOCALE_KEY.NO_DATA_TEXT)"
      :items-per-page-text="$t(LOCALE_KEY.TOTAL_ITEM_PER_PAGE)"
      :items-per-page-options="CONSTANT.ITEM_PER_PAGE_OPTION"
      :loading-text="$t(LOCALE_KEY.LOADING_DATA)"
      :page-text="pageText"
      :page="options.page"
    >
      <template v-slot:item.tags="{ value }">
        <v-tooltip v-if="value" :text="value.toString()">
          <template v-slot:activator="{ props }">
            <div v-bind="props">
              <Chip
                v-if="value[0]"
                :value="value[0]"
                :numberChip="CONSTANT.NUMBER_CHIP"
                :lengthItemChip="CONSTANT.CHIP_LENGTH"
              />
            </div>
          </template>
        </v-tooltip>
      </template>
      <template v-slot:item.edit="{ item }">
        <v-icon class="cursor-pointer" color="green" @click="onUpdateItem(item)">mdi-pencil</v-icon>
      </template>
      <template v-slot:item.delete="{ item }">
        <v-icon class="cursor-pointer" color="red" @click="onRemoveItem(item)">mdi-delete</v-icon>
      </template>
    </v-data-table-server>
    <ConfirmDialog ref="refConfirmDialog" @on-agree="onAgree" />
    <Loading :active="isLoading" />
    <UserModel ref="refUserModel" @on-reload-table="onReloadTable" />
  </v-container>
</template>

<style lang="scss" scoped>
.v-container {
  width: 1500px;
  margin-top: 50px;
}

.blue-bg {
  background-color: rgb(24, 103, 192) !important;
}

.red-bg {
  background-color: #eb3939 !important;
}

.grey-bg {
  background-color: #6b6060 !important;
}

.text-white {
  color: #fff !important;
}

.v-label {
  opacity: 1;
  font-size: 25px;
}

.v-btn--disabled {
  background-color: #eb3939 !important;
}

:deep(.v-table.v-table--fixed-header > .v-table__wrapper > table > thead > tr > th) {
  background-color: rgb(24, 103, 192);
  color: #fff;
}

:deep(.v-table .v-table__wrapper > table > tbody > tr > td) {
  border-left: thin solid #ccccccb8;
  height: 12px;
}

:deep(.v-table .v-table__wrapper > table > tbody > tr:last-child > td) {
  border-bottom: thin solid #ccccccb8;
}

:deep(.v-table .v-table__wrapper > table > tbody > tr > td:last-child) {
  border-right: thin solid #ccccccb8;
}

:deep(.v-data-table .v-table__wrapper > table > thead > tr > th.v-data-table__th--sortable:hover) {
  color: #fff;
}

:deep(.v-selection-control__input > .v-icon) {
  opacity: 1;
}

:deep(.v-table.v-table--fixed-header > .v-table__wrapper > table > thead > tr > th) {
  border-right: thin solid #ccccccb8;
}
</style>