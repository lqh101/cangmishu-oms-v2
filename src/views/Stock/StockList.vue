<template>
  <div class="stock-list">
    <!-- 筛选模块 -->
    <div class="product-search">
      <!-- 搜索过滤区域 -->
      <div class="row items-center">
        <!-- 关键词搜索模块 -->
        <div class="row items-center no-wrap search-group">
          <q-select
            outlined
            dense
            v-model="filters.search_type"
            :options="searchTypeOptions"
            emit-value
            map-options
            option-value="value"
            option-label="label"
            class="search-type-select"
          />
          <q-input
            outlined
            dense
            v-model="filters.keywords"
            :placeholder="t('批量搜索用逗号隔开')"
            class="keywords-input"
          />
          <q-select
            outlined
            dense
            v-model="filters.search_mode"
            :options="searchModeOptions"
            emit-value
            map-options
            option-value="value"
            option-label="label"
            class="search-mode-select"
          />
        </div>

        <div class="q-ml-md">
          <q-btn color="primary" :label="t('搜索')" @click="handleSearch" />
        </div>
      </div>
    </div>

    <!-- 表格容器 -->
    <div class="stock-container">
      <div class="row justify-end q-mb-md">
        <q-btn
          color="primary"
          :label="t('导出')"
          icon="file_download"
          @click="handleExport"
          :disable="!selectedRows.length"
        />
      </div>

      <!-- 数据表格 -->
      <div class="bg-white rounded-borders">
        <q-table
          :rows="tableData"
          :columns="columns"
          row-key="id"
          flat
          bordered
          :rows-per-page-options="[10, 20, 50]"
          v-model:pagination="tablePagination"
          hide-pagination
          :loading="loading"
          selection="multiple"
          v-model:selected="selectedRows"
        >
          <!-- 自定义SKU单元格 -->
          <template v-slot:body-cell-sku="props">
            <q-td :props="props">
              <div class="row items-start">
                <img :src="props.row.image || 'https://picsum.photos/200'" class="q-mr-md" style="width: 60px; height: 60px; object-fit: cover; border-radius: 4px;" />
                <div class="col">
                  <div class="text-weight-medium">{{ props.row.sku }}</div>
                  <div class="text-grey-7 q-mt-xs">{{ t('名称') }}：{{ props.row.product_name }}</div>
                  <div class="text-grey-7">{{ t('规格') }}：{{ props.row.name }}</div>
                </div>
              </div>
            </q-td>
          </template>

          <!-- 表格底部合计行 -->
          <template v-slot:bottom-row>
            <q-tr>
              <q-td colspan="2">{{ t('合计') }}</q-td>
              <q-td align="right">{{ totalInTransit }}</q-td>
              <q-td align="right">{{ totalPendingReceipt }}</q-td>
              <q-td align="right">{{ totalPendingShelf }}</q-td>
              <q-td align="right">{{ totalLocked }}</q-td>
              <q-td align="right">{{ totalAvailable }}</q-td>
              <q-td align="right">{{ totalDefective }}</q-td>
              <q-td align="right">{{ totalQuantity }}</q-td>
            </q-tr>
          </template>
        </q-table>

        <!-- 分页 -->
        <div class="q-pa-md">
          <Pagination
            :total-count="pagination.total"
            v-model:page="pagination.page"
            :max-page="pagination.maxPage"
            v-model:rows-per-page="pagination.rowsPerPage"
            @page-change="handlePageChange"
          />
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from "vue";
import { useQuasar } from "quasar";
import { useI18n } from "vue-i18n";
import Pagination from "@/components/Pagination.vue";
import api from "@/api/index";

const $q = useQuasar();
const { t } = useI18n();

// 筛选条件
const filters = ref({
  start_date: '',
  end_date: '',
  search_type: 'sku',
  keywords: '',
  search_mode: 'exact'
});

// 搜索类型选项
const searchTypeOptions = [
  { label: 'SKU', value: 'sku' },
  { label: t('商品名称'), value: 'product_name' }
];

// 搜索模式选项
const searchModeOptions = [
  { label: t('精确搜索'), value: 'exact' },
  { label: t('前缀搜索'), value: 'prefix' },
  { label: t('模糊搜索'), value: 'fuzzy' }
];

// 表格数据
const columns = [
  { 
    name: "sku", 
    label: "SKU信息", 
    field: "sku", 
    align: "left",
    style: 'width: 300px'
  },
  { 
    name: "in_transit_qty", 
    label: t('在途'), 
    field: "in_transit_qty", 
    align: "right",
    sortable: true
  },
  { 
    name: "pending_receipt_qty", 
    label: t('待收货'), 
    field: "pending_receipt_qty", 
    align: "right",
    sortable: true
  },
  { 
    name: "pending_shelf_qty", 
    label: t('待上架'), 
    field: "pending_shelf_qty", 
    align: "right",
    sortable: true
  },
  { 
    name: "locked_qty", 
    label: t('已锁定'), 
    field: "locked_qty", 
    align: "right",
    sortable: true
  },
  { 
    name: "available_qty", 
    label: t('可用库存'), 
    field: "available_qty", 
    align: "right",
    sortable: true
  },
  { 
    name: "defective_qty", 
    label: t('不良品'), 
    field: "defective_qty", 
    align: "right",
    sortable: true
  },
  { 
    name: "total_qty", 
    label: t('库存总量'), 
    field: "total_qty", 
    align: "right",
    sortable: true
  }
];

// 表格数据
const tableData = ref([]);

const loading = ref(false);
const selectedRows = ref([]);

// 分页
const pagination = ref({
  page: 1,
  maxPage: 1,
  rowsPerPage: 50,
  total: 3
});

// 表格分页配置
const tablePagination = ref({
  sortBy: '',
  descending: false,
  page: 1,
  rowsPerPage: 0
});

// 计算合计数据
const totalInTransit = computed(() => tableData.value.reduce((sum, row) => sum + (row.in_transit_qty || 0), 0));
const totalPendingReceipt = computed(() => tableData.value.reduce((sum, row) => sum + (row.pending_receipt_qty || 0), 0));
const totalPendingShelf = computed(() => tableData.value.reduce((sum, row) => sum + (row.pending_shelf_qty || 0), 0));
const totalLocked = computed(() => tableData.value.reduce((sum, row) => sum + (row.locked_qty || 0), 0));
const totalAvailable = computed(() => tableData.value.reduce((sum, row) => sum + (row.available_qty || 0), 0));
const totalDefective = computed(() => tableData.value.reduce((sum, row) => sum + (row.defective_qty || 0), 0));
const totalQuantity = computed(() => tableData.value.reduce((sum, row) => sum + (row.total_qty || 0), 0));

// 获取数据
const fetchData = async () => {
  try {
    loading.value = true;
    const params = {
      page: pagination.value.page,
      page_size: pagination.value.rowsPerPage,
      search_type: filters.value.search_type,
      keywords: filters.value.keywords,
      search_mode: filters.value.search_mode
    };

    const response = await api.getStocksList(params);
    if (response?.success) {
      tableData.value = response.data.items;
      // 更新分页信息
      const meta = response.data.meta;
      pagination.value = {
        page: meta.current_page,
        maxPage: meta.last_page,
        rowsPerPage: meta.per_page,
        total: meta.total
      };
    }
  } catch (error) {
    console.error('获取库存列表失败:', error);
    $q.notify({
      message: t('获取库存列表失败'),
      color: 'negative'
    });
  } finally {
    loading.value = false;
  }
};

// 处理搜索
const handleSearch = () => {
  pagination.value.page = 1;
  fetchData();
};

// 处理导出
const handleExport = () => {
  const selectedIds = selectedRows.value.map(row => row.id);
  console.log('导出选中的ID:', selectedIds);
  // TODO: 调用导出API
};

// 处理分页变更
const handlePageChange = ({ page, rowsPerPage }) => {
  pagination.value.page = page;
  pagination.value.rowsPerPage = rowsPerPage;
  fetchData();
};

// 监听排序变化
watch([() => tablePagination.value.sortBy, () => tablePagination.value.descending], ([newSortBy, newDescending]) => {
  fetchData();
});

// 组件挂载时获取数据
onMounted(() => {
  fetchData();
});
</script>

<style lang="scss" scoped>
.stock-list {
  padding: 16px;

  .product-search {
    background-color: white;
    padding: 16px;
    border-radius: 8px;
    margin-bottom: 16px;

    .time-group, .search-group {
      :deep(.q-field__control) {
        border: 1px solid rgba(0, 0, 0, 0.12) !important;
        height: 40px;
      }

      :deep(.q-field--outlined .q-field__control:before) {
        border: none;
      }

      :deep(.q-field--outlined .q-field__control:after) {
        border: none;
      }
    }

    .date-range {
      .row {
        margin: 0;
      }

      .date-input {
        :deep(.q-field__control) {
          border-radius: 0 !important;
        }
      }

      .start-date {
        :deep(.q-field__control) {
          border-right: none !important;
          border-radius: 4px 0 0 4px !important;
        }
      }

      .end-date {
        :deep(.q-field__control) {
          border-left: none !important;
          border-radius: 0 4px 4px 0 !important;
        }
      }

      .date-separator {
        padding: 0 4px;
        display: flex;
        align-items: center;
        background: #fff;
        border-top: 1px solid rgba(0, 0, 0, 0.12);
        border-bottom: 1px solid rgba(0, 0, 0, 0.12);
      }
    }

    .search-group {
      .search-type-select {
        min-width: fit-content;
        :deep(.q-field__control) {
          border-radius: 4px 0 0 4px !important;
          border-right: none !important;
        }
      }

      .keywords-input {
        flex: 1;
        :deep(.q-field__control) {
          border-radius: 0 !important;
          border-right: none !important;
        }
      }

      .search-mode-select {
        min-width: fit-content;
        :deep(.q-field__control) {
          border-radius: 0 4px 4px 0 !important;
        }
      }
    }
  }

  .stock-container {
    background-color: white;
    padding: 16px;
    border-radius: 8px;
  }

  .q-table th {
    font-weight: 500;
  }
  
  .q-table tbody td {
    height: 56px;
  }
}
</style> 