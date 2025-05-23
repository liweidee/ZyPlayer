<template>
  <div class="setting-drive-container">
    <common-setting
      row-key="id"
      :op="op"
      :data="tableConfig.data"
      :columns="COLUMNS"
      :pagination="pagination"
      @op-change="handleOpChange"
      @page-change="handlePageChange"
      @op-search="handleOpSearch"
    >
      <template #name="{ row }">
        <t-badge v-if="row.id === tableConfig.default" size="small" dot :count="999">
          {{ row.name }}
        </t-badge>
        <span v-else>{{ row.name }}</span>
      </template>
      <template #showAll="{ row }">
        <t-tag v-if="row.showAll" theme="success" shape="round" variant="light-outline">
          {{ $t('pages.setting.drive.all') }}
        </t-tag>
        <t-tag v-else theme="warning" shape="round" variant="light-outline">
          {{ $t('pages.setting.drive.video') }}
        </t-tag>
      </template>
      <template #isActive="{ row }">
        <t-switch v-model="row.isActive" :disabled="row.key === 'debug'"  @change="handleOpDefault(row.id)" />
      </template>
      <template #op="slotProps">
        <t-space>
          <t-link theme="primary" @click="handleOpChange('default', slotProps.row.id)">{{ $t('pages.setting.table.default') }}</t-link>
          <t-link theme="primary" @click="handleOpChange('edit', slotProps.row)">{{ $t('pages.setting.table.edit') }}</t-link>
          <t-popconfirm :content="$t('pages.setting.table.deleteTip')" @confirm="handleOpChange('delete', [slotProps.row.id])">
            <t-link theme="danger">{{ $t('pages.setting.table.delete') }}</t-link>
          </t-popconfirm>
        </t-space>
      </template>
    </common-setting>

    <dialog-form-view v-model:visible="active.dialogForm" :data="formData" :type="active.formType" @submit="handleDialogUpdate" />
  </div>
</template>

<script setup lang="ts">
import { MessagePlugin } from 'tdesign-vue-next';
import { onActivated, onMounted, ref, reactive, computed } from 'vue';

import { t } from '@/locales';
import { fetchDrivePage, putDrive, delDrive, addDrive, putDriveDefault } from '@/api/drive';
import emitter from '@/utils/emitter';

import { COLUMNS } from './constants';

import DialogFormView from './components/DialogForm.vue';
import CommonSetting from '@/components/common-setting/table/index.vue';


const op = computed(() => {
  return[
    {
      label: t('pages.setting.header.add'),
      value: 'add'
    },
    {
      label: t('pages.setting.header.enable'),
      value: 'enable'
    },
    {
      label: t('pages.setting.header.disable'),
      value: 'disable'
    },
    {
      label: t('pages.setting.header.delete'),
      value: 'delete'
    }
  ]
});

const active = reactive({
  dialogForm: false,
  formType: 'add',
  opId: ''
});
const formData = ref({});
const searchValue = ref<string>('');
const pagination = reactive({
  defaultPageSize: 20,
  total: 0,
  defaultCurrent: 1,
  pageSize: 20,
  current: 1,
  theme: "simple"
});
const tableConfig = ref({
  data: [],
  rawData: [],
  sort: {},
  filter: {
    type: [],
  },
  select: [],
  default: ''
});

onMounted(() => {
  reqFetch(pagination.current, pagination.pageSize, searchValue.value);
});

onActivated(() => {
  const isListenedRefreshTableData = emitter.all.get('refreshDriveTable');
  if (!isListenedRefreshTableData) emitter.on('refreshDriveTable', () => {
    console.log('[setting][drive][bus][refresh]');
    defaultSet();
    refreshTable();
  });
});

const defaultSet = () => {
  pagination.defaultCurrent = 1;
  pagination.defaultPageSize = 20;
  pagination.current = 1;
  pagination.pageSize = 20;
  pagination.total = 0;

  tableConfig.value = {
    data: [],
    rawData: [],
    sort: {},
    filter: {
      type: [],
    },
    select: [],
    default: ''
  };
};

const refreshTable = () => {
  reqFetch(pagination.current, pagination.pageSize, searchValue.value);
};

const reqFetch = async (page, pageSize, kw) => {
  try {
    const res = await fetchDrivePage({
      kw, page, pageSize,
    });
    if (res?.["default"]) {
      tableConfig.value.default = res.default;
    }
    if (res?.["data"]) {
      tableConfig.value.data = res.data;
      tableConfig.value.rawData = res.data;
    }
    if (res?.["total"]) {
      pagination.total = res.total;
    }
  } catch (err: any) {
    console.log('[setting][drive][reqFetch][error]', err);
    MessagePlugin.error(`${t('pages.setting.form.fail')}: ${err.message}`);
  }
};

const reqPut = async (index, doc) => {
  try {
    await putDrive({ ids: index, doc });
    MessagePlugin.success(`${t('pages.setting.form.success')}`);
  } catch (err: any) {
    console.log('[setting][drive][reqPut][error]', err);
    MessagePlugin.error(`${t('pages.setting.form.fail')}: ${err.message}`);
  }
};

const reqAdd = async (doc) => {
  try {
    await addDrive(doc);
    MessagePlugin.success(`${t('pages.setting.form.success')}`);
  } catch (err: any) {
    console.log('[setting][drive][reqAdd][error]', err);
    MessagePlugin.error(`${t('pages.setting.form.fail')}: ${err.message}`);
  }
};

const reqDefault = async (key) => {
  try {
    await putDriveDefault(key);
    MessagePlugin.success(`${t('pages.setting.form.success')}`);
  } catch (err: any) {
    console.log('[setting][drive][defaultEvent][error]', err);
    MessagePlugin.error(`${t('pages.setting.form.fail')}: ${err.message}`);
  }
};

const reqDel = async (index) => {
  try {
    await delDrive({ ids: index });
    MessagePlugin.success(`${t('pages.setting.form.success')}`);
  } catch (err: any) {
    console.log('[setting][drive][reqDel][error]', err);
    MessagePlugin.error(`${t('pages.setting.form.fail')}: ${err.message}`);
  }
};

const handleOpDefault = async (id) => {
  const item: any = tableConfig.value.data.find((item: any) => item.id === id);
  handleOpChange(item.isActive ? 'enable' : 'disable', [id]);
};

const handleOpChange = async (type, doc) => {
  if (doc.length === 0 && ['enable', 'disable', 'delete'].includes(type)) {
    MessagePlugin.warning(t('pages.setting.message.noSelectData'));
    return;
  };

  if (type === 'add') {
    active.formType = 'add';
    formData.value = {
      name: '',
      key: '',
      server: '',
      startPage: '',
      search: false,
      headers: null,
      params: null,
      showAll: false,
      isActive: true
    };
    active.dialogForm = true;
  } else if (type === 'enable') {
    await reqPut(doc, { isActive: true });
  } else if (type === 'disable') {
    await reqPut(doc, { isActive: false });
  } else if (type === 'delete') {
    await reqDel(doc);
  } else if (type === 'default') {
    const activeItem: any = tableConfig.value.data.find((item:any) => item.id === doc)
    if (!activeItem || !activeItem.isActive) {
      MessagePlugin.warning(t('pages.setting.message.defaultDisable'));
      return;
    };
    await reqDefault(doc);
  } else if (type === 'edit') {
    active.formType = 'edit';
    active.opId = doc.id;
    delete doc.id;
    formData.value = doc;
    active.dialogForm = true;
  };

  if (['enable', 'disable', 'delete', 'default'].includes(type)) {
    refreshTable();
    emitter.emit('refreshDriveConfig');
  };
};

const handleDialogUpdate = async (type: string, doc: object) => {
  if (type === 'table') {
    if (active.formType === 'add') {
      await reqAdd(doc);
    } else {
      await reqPut([active.opId], doc);
    };
  };

  refreshTable();
  emitter.emit('refreshDriveConfig');
};

const handleOpSearch = (value: string) => {
  searchValue.value = value;
  pagination.current = 1;
  refreshTable();
};

const handlePageChange = (page: number, pageSize: number) => {
  pagination.current = page;
  pagination.pageSize = pageSize;
  refreshTable();
};
</script>

<style lang="less" scoped></style>
