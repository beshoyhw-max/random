<template>
  <div class="dashboard-container">
    <!-- Sidebar -->
    <aside class="sidebar">
      <div class="sidebar-header">
        <h1><span class="highlight">北部非洲人</span><br />进人出预测系统</h1>
      </div>
      <ul class="nav-menu">
        <li
          v-for="(item, i) in navItems"
          :key="i"
          :class="{ active: activeNav === i }"
          @click="activeNav = i"
        >
          <span class="nav-indicator"></span>
          {{ item }}
        </li>
      </ul>
    </aside>

    <!-- Main Content -->
    <main class="main-content">
      <!-- Top Header -->
      <header class="top-header">
        <h2 class="page-title">当前人进人出预测</h2>
        <div class="date-filter">
          <span class="date-label">期间</span>
          <!-- Inline MonthPicker -->
          <div class="month-picker-wrapper" ref="pickerWrapper">
            <button class="month-picker-trigger" @click="togglePicker">
              {{ formatPeriod(currentPeriod) }}
              <span class="picker-arrow" :class="{ open: pickerOpen }">▾</span>
            </button>
            <div v-if="pickerOpen" class="month-picker-dropdown">
              <div class="picker-header">
                <button class="picker-nav" @click="changeYear(-1)">‹</button>
                <span class="picker-year">{{ pickerYear }}</span>
                <button class="picker-nav" @click="changeYear(1)">›</button>
              </div>
              <div class="picker-months">
                <button
                  v-for="(m, idx) in months"
                  :key="idx"
                  class="picker-month"
                  :class="{ selected: isSelected(idx + 1) }"
                  @click="selectMonth(idx + 1)"
                >
                  {{ m }}
                </button>
              </div>
            </div>
          </div>
        </div>
      </header>

      <!-- Stats Cards -->
      <section class="stats-section">
        <div class="stat-card" v-for="(card, index) in statsCards" :key="index">
          <div class="card-header">
            <span class="icon-bulb">💡</span> {{ card.title }}
          </div>
          <div class="card-body">
            <table class="card-table">
              <thead>
                <tr>
                  <th></th>
                  <th>年度预算</th>
                  <th>当前预测</th>
                  <th>预测比预算</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(row, rIndex) in card.data" :key="rIndex">
                  <td class="row-label">
                    <span class="blue-bar"></span>{{ row.label }}
                  </td>
                  <td class="bold-num">{{ row.budget }}</td>
                  <td class="bold-num">{{ row.forecast }}</td>
                  <td class="bold-num">{{ row.diff }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </section>

      <!-- Toolbar -->
      <section class="toolbar">
        <div class="toolbar-left">
          <!-- Normal mode -->
          <button v-if="!editMode" class="btn btn-primary" @click="handleEdit">
            <span class="icon">✏️</span> 启动编辑
          </button>
          <!-- Edit mode -->
          <template v-if="editMode">
            <button class="btn btn-gray" @click="handleCancelEdit">取消编辑</button>
            <button class="btn btn-primary" @click="handleSave">保存</button>
            <button
              class="btn btn-danger"
              :disabled="selectedRows.length === 0"
              @click="handleDelete"
            >删除</button>
            <button class="btn btn-primary" @click="handleAdd">+ 添加</button>
          </template>
          <button class="btn btn-secondary" @click="handleExport">
            <span class="icon">📤</span> 导出
          </button>
        </div>
        <div class="toolbar-right">
          <div class="search-group">
            <select v-model="searchField" class="search-select">
              <option value="name">姓名</option>
              <option value="id">工号</option>
              <option value="dept">部门</option>
            </select>
            <span class="divider">|</span>
            <input
              v-model="searchQuery"
              type="text"
              class="search-input"
              @keyup.enter="handleSearch"
            />
            <button class="search-btn" @click="handleSearch">🔍</button>
          </div>
          <div class="advanced-filter" @click="toggleAdvancedFilter">
            高级筛选 <span class="arrow-down" :class="{ rotated: showAdvanced }">⌄</span>
          </div>
          <div class="action-icons">
            <button class="icon-btn" title="刷新" @click="handleRefresh">🔃</button>
            <button class="icon-btn" title="设置" @click="handleSettings">⚙️</button>
          </div>
        </div>
      </section>

      <!-- Data Table -->
      <section class="table-section">
        <div class="table-wrapper">
          <table class="main-table">
            <thead>
              <tr>
                <th class="th-cell checkbox-col">
                  <span v-if="!editMode" class="col-placeholder">✓</span>
                  <input v-if="editMode" type="checkbox" @change="toggleSelectAll" :checked="selectedRows.length === tableData.length && tableData.length > 0" />
                </th>
                <th v-for="col in columnConfig" :key="col.key" class="th-cell" :class="{ 'frozen-col': frozenColumns.includes(col.key) }">
                  {{ col.label }}
                </th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(row, index) in tableData" :key="index">
                <td class="action-cell">
                  <span class="row-num">{{ index + 1 }}</span>
                  <input v-if="editMode" type="checkbox" class="row-checkbox" :checked="selectedRows.includes(index + 1)" @change="toggleRowSelection(index + 1)" />
                </td>
                <td @click="editMode && startEditCell(index + 1, 'id')" :class="{ 'frozen-col': frozenColumns.includes('id') }">
                  <input v-if="editingCell && editingCell.row === index + 1 && editingCell.col === 'id'" v-model="row.id" @blur="saveCellValue(index + 1, 'id', row.id)" @keyup.enter="saveCellValue(index + 1, 'id', row.id)" @keyup.esc="cancelEditCell" class="cell-input" autofocus />
                  <span v-else>{{ row.id }}</span>
                </td>
                <td @click="editMode && startEditCell(index + 1, 'name')" :class="{ 'frozen-col': frozenColumns.includes('name') }">
                  <input v-if="editingCell && editingCell.row === index + 1 && editingCell.col === 'name'" v-model="row.name" @blur="saveCellValue(index + 1, 'name', row.name)" @keyup.enter="saveCellValue(index + 1, 'name', row.name)" @keyup.esc="cancelEditCell" class="cell-input" autofocus />
                  <span v-else>{{ row.name }}</span>
                </td>
                <td @click="editMode && startEditCell(index + 1, 'nationality')" :class="{ 'frozen-col': frozenColumns.includes('nationality') }">
                  <input v-if="editingCell && editingCell.row === index + 1 && editingCell.col === 'nationality'" v-model="row.nationality" @blur="saveCellValue(index + 1, 'nationality', row.nationality)" @keyup.enter="saveCellValue(index + 1, 'nationality', row.nationality)" @keyup.esc="cancelEditCell" class="cell-input" autofocus />
                  <span v-else>{{ row.nationality }}</span>
                </td>
                <td @click="editMode && startEditCell(index + 1, 'dept2')" :class="{ 'frozen-col': frozenColumns.includes('dept2') }">
                  <input v-if="editingCell && editingCell.row === index + 1 && editingCell.col === 'dept2'" v-model="row.dept2" @blur="saveCellValue(index + 1, 'dept2', row.dept2)" @keyup.enter="saveCellValue(index + 1, 'dept2', row.dept2)" @keyup.esc="cancelEditCell" class="cell-input" autofocus />
                  <span v-else>{{ row.dept2 }}</span>
                </td>
                <td @click="editMode && startEditCell(index + 1, 'dept3')" :class="{ 'frozen-col': frozenColumns.includes('dept3') }">
                  <input v-if="editingCell && editingCell.row === index + 1 && editingCell.col === 'dept3'" v-model="row.dept3" @blur="saveCellValue(index + 1, 'dept3', row.dept3)" @keyup.enter="saveCellValue(index + 1, 'dept3', row.dept3)" @keyup.esc="cancelEditCell" class="cell-input" autofocus />
                  <span v-else>{{ row.dept3 }}</span>
                </td>
                <td @click="editMode && startEditCell(index + 1, 'industry')" :class="{ 'frozen-col': frozenColumns.includes('industry') }">
                  <input v-if="editingCell && editingCell.row === index + 1 && editingCell.col === 'industry'" v-model="row.industry" @blur="saveCellValue(index + 1, 'industry', row.industry)" @keyup.enter="saveCellValue(index + 1, 'industry', row.industry)" @keyup.esc="cancelEditCell" class="cell-input" autofocus />
                  <span v-else>{{ row.industry }}</span>
                </td>
                <td @click="editMode && startEditCell(index + 1, 'position')" :class="{ 'frozen-col': frozenColumns.includes('position') }">
                  <input v-if="editingCell && editingCell.row === index + 1 && editingCell.col === 'position'" v-model="row.position" @blur="saveCellValue(index + 1, 'position', row.position)" @keyup.enter="saveCellValue(index + 1, 'position', row.position)" @keyup.esc="cancelEditCell" class="cell-input" autofocus />
                  <span v-else>{{ row.position }}</span>
                </td>
                <td @click="editMode && startEditCell(index + 1, 'level')" :class="{ 'frozen-col': frozenColumns.includes('level') }">
                  <input v-if="editingCell && editingCell.row === index + 1 && editingCell.col === 'level'" v-model="row.level" @blur="saveCellValue(index + 1, 'level', row.level)" @keyup.enter="saveCellValue(index + 1, 'level', row.level)" @keyup.esc="cancelEditCell" class="cell-input" autofocus />
                  <span v-else>{{ row.level }}</span>
                </td>
                <td @click="editMode && startEditCell(index + 1, 'lastMonthInOffice')" :class="{ 'frozen-col': frozenColumns.includes('lastMonthInOffice') }">
                  <input v-if="editingCell && editingCell.row === index + 1 && editingCell.col === 'lastMonthInOffice'" v-model="row.lastMonthInOffice" @blur="saveCellValue(index + 1, 'lastMonthInOffice', row.lastMonthInOffice)" @keyup.enter="saveCellValue(index + 1, 'lastMonthInOffice', row.lastMonthInOffice)" @keyup.esc="cancelEditCell" class="cell-input" autofocus />
                  <span v-else>{{ row.lastMonthInOffice }}</span>
                </td>
                <td @click="editMode && startEditCell(index + 1, 'forecast0531')" :class="{ 'frozen-col': frozenColumns.includes('forecast0531') }">
                  <input v-if="editingCell && editingCell.row === index + 1 && editingCell.col === 'forecast0531'" v-model="row.forecast0531" @blur="saveCellValue(index + 1, 'forecast0531', row.forecast0531)" @keyup.enter="saveCellValue(index + 1, 'forecast0531', row.forecast0531)" @keyup.esc="cancelEditCell" class="cell-input" autofocus />
                  <span v-else>{{ row.forecast0531 }}</span>
                </td>
                <td @click="editMode && startEditCell(index + 1, 'forecast0630')" :class="{ 'frozen-col': frozenColumns.includes('forecast0630') }">
                  <input v-if="editingCell && editingCell.row === index + 1 && editingCell.col === 'forecast0630'" v-model="row.forecast0630" @blur="saveCellValue(index + 1, 'forecast0630', row.forecast0630)" @keyup.enter="saveCellValue(index + 1, 'forecast0630', row.forecast0630)" @keyup.esc="cancelEditCell" class="cell-input" autofocus />
                  <span v-else>{{ row.forecast0630 }}</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </section>

      <!-- Settings Popup -->
      <div v-if="showSettings" class="settings-overlay" @click.self="showSettings = false">
        <div class="settings-popup">
          <div class="settings-header">
            <h3>列设置</h3>
            <button class="settings-close" @click="showSettings = false">✕</button>
          </div>
          <div class="settings-content">
            <div class="settings-section">
              <h4>显示/隐藏列</h4>
              <div class="column-list">
                <div v-for="col in columnConfig" :key="col.key" class="column-item">
                  <label>
                    <input type="checkbox" :checked="col.visible" @change="toggleColumnVisibility(col.key)" />
                    {{ col.label }}
                  </label>
                </div>
              </div>
            </div>
            <div class="settings-section">
              <h4>冻结列 (左侧)</h4>
              <div class="column-list">
                <div v-for="col in columnConfig" :key="col.key" class="column-item">
                  <label>
                    <input type="checkbox" :checked="col.frozen" @change="toggleColumnFreeze(col.key)" />
                    {{ col.label }}
                  </label>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<script>
export default {
  name: 'Dashboard',
  data() {
    return {
      activeNav: 0,
      navItems: [
        '人进人出预测作业中心',
        '人进人出预测数据看板',
        '人进人出实际数据看板',
        '权限管理'
      ],
      currentPeriod: { year: new Date().getFullYear(), month: new Date().getMonth() + 1 },
      pickerOpen: false,
      pickerYear: new Date().getFullYear(),
      months: ['1月','2月','3月','4月','5月','6月','7月','8月','9月','10月','11月','12月'],
      searchField: 'name',
      searchQuery: '',
      showAdvanced: false,
      statsCards: [
        {
          title: '预测比预算',
          data: [
            { label: '整体', budget: '1200', forecast: '500', diff: '200' },
            { label: '中方', budget: '201', forecast: '300', diff: '500' },
            { label: '本地', budget: '10', forecast: '100', diff: '40' }
          ]
        },
        {
          title: '预测比上月',
          data: [
            { label: '整体', budget: '1200', forecast: '500', diff: '200' },
            { label: '中方', budget: '201', forecast: '300', diff: '500' },
            { label: '本地', budget: '10', forecast: '100', diff: '40' }
          ]
        },
        {
          title: '人进比人出',
          data: [
            { label: '整体', budget: '1200', forecast: '500', diff: '200' },
            { label: '中方', budget: '201', forecast: '300', diff: '500' },
            { label: '本地', budget: '10', forecast: '100', diff: '40' }
          ]
        }
      ],
      tableColumns: [
        '工号', '姓名', '中本', '二级部门', '最小部门', '行业线',
        '岗位名称', '个人职级范围', '上月是否在岗', '0531预测', '0630预测'
      ],
      tableData: [
        { id: 'EMP001', name: '张三', nationality: '中方', dept2: '技术部', dept3: '研发组', industry: 'IT', position: '高级工程师', level: 'P6', lastMonthInOffice: '是', forecast0531: '在岗', forecast0630: '在岗' },
        { id: 'EMP002', name: '李四', nationality: '本地', dept2: '市场部', dept3: '销售组', industry: '销售', position: '销售经理', level: 'P5', lastMonthInOffice: '是', forecast0531: '在岗', forecast0630: '在岗' },
        { id: 'EMP003', name: '王五', nationality: '中方', dept2: '财务部', dept3: '会计组', industry: '金融', position: '财务主管', level: 'P6', lastMonthInOffice: '否', forecast0531: '离岗', forecast0630: '在岗' },
        { id: 'EMP004', name: 'Ahmed', nationality: '本地', dept2: '运营部', dept3: '物流组', industry: '物流', position: '运营专员', level: 'P4', lastMonthInOffice: '是', forecast0531: '在岗', forecast0630: '在岗' },
        { id: 'EMP005', name: '赵六', nationality: '中方', dept2: '技术部', dept3: '测试组', industry: 'IT', position: '测试工程师', level: 'P5', lastMonthInOffice: '是', forecast0531: '在岗', forecast0630: '在岗' },
        { id: 'EMP006', name: 'Mohammed', nationality: '本地', dept2: '人力资源部', dept3: '招聘组', industry: 'HR', position: '招聘专员', level: 'P4', lastMonthInOffice: '是', forecast0531: '在岗', forecast0630: '离岗' },
        { id: 'EMP007', name: '钱七', nationality: '中方', dept2: '技术部', dept3: '运维组', industry: 'IT', position: '运维工程师', level: 'P5', lastMonthInOffice: '否', forecast0531: '离岗', forecast0630: '在岗' }
      ],
      editMode: false,
      selectedRows: [],
      editingCell: null,
      originalData: [],
      showSettings: false,
      columnConfig: [
        { key: 'id', label: '工号', visible: true, frozen: false },
        { key: 'name', label: '姓名', visible: true, frozen: false },
        { key: 'nationality', label: '中本', visible: true, frozen: false },
        { key: 'dept2', label: '二级部门', visible: true, frozen: false },
        { key: 'dept3', label: '最小部门', visible: true, frozen: false },
        { key: 'industry', label: '行业线', visible: true, frozen: false },
        { key: 'position', label: '岗位名称', visible: true, frozen: false },
        { key: 'level', label: '个人职级范围', visible: true, frozen: false },
        { key: 'lastMonthInOffice', label: '上月是否在岗', visible: true, frozen: false },
        { key: 'forecast0531', label: '0531预测', visible: true, frozen: false },
        { key: 'forecast0630', label: '0630预测', visible: true, frozen: false }
      ],
      frozenColumns: []
    }
  },
  computed: {
    dynamicColumns() {
      const month = this.currentPeriod.month
      const months = ['01', '02', '03', '04', '05', '06', '07', '08', '09', '10', '11', '12']
      const dynamicCols = []
      for (let i = month; i <= 12; i++) {
        dynamicCols.push({ key: `forecast${months[i - 1]}`, label: `${i}月预测`, visible: true, frozen: false })
      }
      return dynamicCols
    },
    allColumns() {
      const baseCols = this.columnConfig.filter(c => !c.key.startsWith('forecast'))
      return [...baseCols, ...this.dynamicColumns]
    }
  },
  mounted() {
    document.addEventListener('click', this.handleOutsideClick)
  },
  beforeUnmount() {
    document.removeEventListener('click', this.handleOutsideClick)
  },
  methods: {
    formatPeriod({ year, month }) {
      return `${year}-${String(month).padStart(2, '0')}`
    },
    togglePicker() {
      this.pickerOpen = !this.pickerOpen
      if (this.pickerOpen) {
        this.pickerYear = this.currentPeriod.year
      }
    },
    changeYear(delta) {
      this.pickerYear += delta
    },
    isSelected(month) {
      return this.currentPeriod.year === this.pickerYear && this.currentPeriod.month === month
    },
    selectMonth(month) {
      this.currentPeriod = { year: this.pickerYear, month }
      this.pickerOpen = false
    },
    handleOutsideClick(e) {
      if (this.$refs.pickerWrapper && !this.$refs.pickerWrapper.contains(e.target)) {
        this.pickerOpen = false
      }
    },
    handleEdit() {
      this.originalData = JSON.parse(JSON.stringify(this.tableData))
      this.editMode = true
    },
    handleCancelEdit() {
      this.tableData = JSON.parse(JSON.stringify(this.originalData))
      this.editMode = false
      this.selectedRows = []
      this.editingCell = null
    },
    handleSave() {
      this.originalData = JSON.parse(JSON.stringify(this.tableData))
      this.editMode = false
      this.selectedRows = []
      this.editingCell = null
      console.log('Saved:', this.tableData)
    },
    handleDelete() {
      if (this.selectedRows.length === 0) {
        return
      }
      const indicesToDelete = this.selectedRows.sort((a, b) => b - a)
      indicesToDelete.forEach(index => {
        this.tableData.splice(index - 1, 1)
      })
      this.selectedRows = []
      console.log('Deleted rows:', indicesToDelete)
    },
    handleAdd() {
      console.log('Add new row')
    },
    toggleSelectAll(event) {
      const checked = event.target.checked
      if (checked) {
        this.selectedRows = this.tableData.map((_, i) => i + 1)
      } else {
        this.selectedRows = []
      }
    },
    toggleRowSelection(index) {
      const idx = this.selectedRows.indexOf(index)
      if (idx > -1) {
        this.selectedRows.splice(idx, 1)
      } else {
        this.selectedRows.push(index)
      }
    },
    startEditCell(rowIndex, colKey) {
      this.editingCell = { row: rowIndex, col: colKey }
    },
    saveCellValue(rowIndex, colKey, newValue) {
      this.tableData[rowIndex - 1][colKey] = newValue
      this.editingCell = null
    },
    cancelEditCell() {
      this.editingCell = null
    },
    handleExport() {
      console.log('Export triggered')
    },
    handleSearch() {
      console.log('Search:', this.searchField, this.searchQuery)
    },
    toggleAdvancedFilter() {
      this.showAdvanced = !this.showAdvanced
    },
    handleRefresh() {
      console.log('Refresh triggered')
    },
    handleSettings() {
      this.showSettings = !this.showSettings
    },
    toggleColumnVisibility(key) {
      const col = this.columnConfig.find(c => c.key === key)
      if (col) {
        col.visible = !col.visible
      }
    },
    toggleColumnFreeze(key) {
      const col = this.columnConfig.find(c => c.key === key)
      if (col) {
        col.frozen = !col.frozen
        this.frozenColumns = this.columnConfig.filter(c => c.frozen).map(c => c.key)
      }
    },
    handleRowEdit(rowIndex) {
      console.log('Edit row:', rowIndex)
    }
  }
}
</script>

<style scoped>
/* Reset & Base */
*, *::before, *::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

.dashboard-container {
  display: flex;
  height: 100vh;
  width: 100%;
  overflow: hidden;
  font-family: 'Helvetica Neue', Helvetica, 'PingFang SC', 'Hiragino Sans GB', 'Microsoft YaHei', Arial, sans-serif;
  background-color: #fcfdfe;
  color: #333;
  font-size: 14px;
}

/* Sidebar */
.sidebar {
  width: 240px;
  flex-shrink: 0;
  background: linear-gradient(180deg, #e4eff8 0%, #cce0f5 100%);
  border-right: 1px solid #dcdfe6;
  display: flex;
  flex-direction: column;
  overflow-y: auto;
}

.sidebar-header {
  padding: 30px 20px;
  color: #2b579a;
}

.sidebar-header h1 {
  font-size: 18px;
  font-weight: 600;
  line-height: 1.5;
}

.highlight {
  color: #1a4c8a;
}

.nav-menu {
  list-style: none;
  margin-top: 20px;
}

.nav-menu li {
  padding: 15px 20px;
  font-size: 14px;
  color: #5c7a99;
  cursor: pointer;
  display: flex;
  align-items: center;
  margin-bottom: 5px;
  transition: background-color 0.2s;
}

.nav-menu li:hover:not(.active) {
  background-color: rgba(49, 130, 206, 0.08);
}

.nav-indicator {
  display: inline-block;
  width: 3px;
  height: 16px;
  background-color: #90b3d6;
  margin-right: 10px;
  flex-shrink: 0;
}

.nav-menu li.active {
  background-color: #3182ce;
  color: white;
}

.nav-menu li.active .nav-indicator {
  background-color: white;
}

/* Main Content */
.main-content {
  flex: 1;
  min-width: 0;
  padding: 20px 30px;
  display: flex;
  flex-direction: column;
  overflow-y: auto;
  gap: 20px;
}

/* Top Header */
.top-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-shrink: 0;
}

.page-title {
  font-size: 20px;
  color: #3182ce;
  font-weight: bold;
}

.date-filter {
  display: flex;
  align-items: center;
  gap: 10px;
}

.date-label {
  color: #666;
  font-size: 14px;
}

/* Inline MonthPicker */
.month-picker-wrapper {
  position: relative;
}

.month-picker-trigger {
  border: 1px solid #dcdfe6;
  background: white;
  padding: 7px 12px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  color: #333;
  display: flex;
  align-items: center;
  gap: 6px;
  min-width: 110px;
  justify-content: space-between;
}

.month-picker-trigger:hover {
  border-color: #3182ce;
}

.picker-arrow {
  font-size: 12px;
  color: #999;
  display: inline-block;
  transition: transform 0.2s;
}

.picker-arrow.open {
  transform: rotate(180deg);
}

.month-picker-dropdown {
  position: absolute;
  top: calc(100% + 6px);
  right: 0;
  background: white;
  border: 1px solid #dcdfe6;
  border-radius: 6px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.12);
  z-index: 100;
  width: 220px;
  padding: 12px;
}

.picker-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
}

.picker-year {
  font-weight: 600;
  color: #333;
}

.picker-nav {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 18px;
  color: #666;
  padding: 0 8px;
  line-height: 1;
  border-radius: 4px;
}

.picker-nav:hover {
  background-color: #f0f4f8;
  color: #3182ce;
}

.picker-months {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 6px;
}

.picker-month {
  background: none;
  border: 1px solid transparent;
  border-radius: 4px;
  padding: 7px 4px;
  cursor: pointer;
  font-size: 13px;
  color: #333;
  text-align: center;
  transition: all 0.15s;
}

.picker-month:hover {
  background-color: #e8f2ff;
  border-color: #3182ce;
  color: #3182ce;
}

.picker-month.selected {
  background-color: #3182ce;
  color: white;
  border-color: #3182ce;
}

/* Stats Cards */
.stats-section {
  display: flex;
  gap: 20px;
  flex-shrink: 0;
}

.stat-card {
  flex: 1;
  min-width: 0;
  background: linear-gradient(180deg, #f0f7ff 0%, #e6f0fa 100%);
  border-radius: 8px;
  border: 1px solid #d5e6f7;
  overflow: hidden;
}

.card-header {
  background-color: #3b8cf0;
  color: white;
  padding: 10px 15px;
  font-size: 14px;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 8px;
}

.card-body {
  padding: 12px 15px;
}

.card-table {
  width: 100%;
  border-collapse: collapse;
  text-align: center;
  font-size: 13px;
}

.card-table th {
  color: #666;
  font-weight: normal;
  padding-bottom: 8px;
}

.card-table td {
  padding: 5px 0;
  color: #333;
}

.row-label {
  text-align: left;
}

.row-label-inner {
  display: inline-flex;
  align-items: center;
  gap: 6px;
}

.blue-bar {
  display: inline-block;
  width: 3px;
  height: 12px;
  background-color: #3b8cf0;
  margin-right: 6px;
  vertical-align: middle;
  flex-shrink: 0;
}

.bold-num {
  font-weight: bold;
  color: #1a56a8;
}

/* Toolbar */
.toolbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-shrink: 0;
  background-color: white;
  padding: 10px 0;
  flex-wrap: wrap;
  gap: 10px;
}

.toolbar-left,
.toolbar-right {
  display: flex;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
}

.btn {
  padding: 8px 18px;
  border-radius: 4px;
  border: none;
  cursor: pointer;
  font-size: 14px;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  white-space: nowrap;
  transition: opacity 0.15s;
}

.btn:hover {
  opacity: 0.88;
}

.btn-primary {
  background-color: #2b3a67;
  color: white;
}

.btn-secondary {
  background-color: #58a3f7;
  color: white;
}

.search-group {
  display: flex;
  align-items: center;
  border: 1px solid #dcdfe6;
  border-radius: 4px;
  background: white;
  overflow: hidden;
}

.search-select {
  border: none;
  outline: none;
  padding: 7px 8px;
  color: #606266;
  background: transparent;
  font-size: 13px;
}

.divider {
  color: #dcdfe6;
}

.search-input {
  border: none;
  outline: none;
  padding: 7px 8px;
  width: 140px;
  font-size: 13px;
}

.search-btn {
  background: none;
  border: none;
  cursor: pointer;
  padding: 7px 10px;
  color: #999;
  font-size: 14px;
}

.search-btn:hover {
  color: #3182ce;
}

.advanced-filter {
  color: #606266;
  font-size: 14px;
  cursor: pointer;
  white-space: nowrap;
  user-select: none;
}

.arrow-down {
  display: inline-block;
  font-size: 14px;
  color: #999;
  transition: transform 0.2s;
}

.arrow-down.rotated {
  transform: rotate(180deg);
}

.action-icons {
  display: flex;
  gap: 8px;
}

.icon-btn {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 16px;
  color: #606266;
  padding: 4px;
  border-radius: 4px;
  transition: background-color 0.15s;
}

.icon-btn:hover {
  background-color: #f0f4f8;
}

/* Data Table */
.table-section {
  background-color: white;
  border-radius: 4px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.05);
  overflow: auto;
  flex: 1;
  min-height: 0;
}

.table-wrapper {
  overflow: auto;
  max-height: 100%;
}

.frozen-col {
  position: sticky;
  left: 0;
  background-color: #fafafa;
  z-index: 1;
}

.table-section thead .frozen-col {
  background-color: #f0f0f0;
}

.main-table {
  width: 100%;
  border-collapse: collapse;
  text-align: center;
  font-size: 13px;
  min-width: 900px;
}

.th-cell {
  padding: 14px 10px;
  color: #606266;
  font-weight: 500;
  background-color: #fafafa;
  position: relative;
  white-space: nowrap;
}

.col-divider {
  color: #dcdfe6;
  font-weight: 300;
  position: absolute;
  right: 0;
  top: 50%;
  transform: translateY(-50%);
}

.main-table td {
  padding: 12px 10px;
  border-bottom: 1px solid #ebeef5;
}

/* Fixed: action-cell uses inline-flex inside a td — wrap content in a span */
.action-cell {
  white-space: nowrap;
}

.action-cell-inner {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.row-num {
  min-width: 18px;
  color: #606266;
  font-size: 12px;
}

.edit-icon {
  cursor: pointer;
  color: #409eff;
  font-size: 12px;
}

.row-checkbox {
  cursor: pointer;
}

.checkbox-col {
  width: 50px;
}

.table-section td {
  cursor: default;
}

.table-section td:hover {
  background-color: #f5f7fa;
}

.cell-input {
  width: 100%;
  padding: 4px 8px;
  border: 1px solid #409eff;
  border-radius: 4px;
  font-size: 13px;
  outline: none;
}

.col-placeholder {
  color: transparent;
}

.btn-danger {
  background-color: #f56c6c;
  color: white;
}

.btn-danger:disabled {
  background-color: #fab6b6;
  cursor: not-allowed;
}

.btn-danger:hover:not(:disabled) {
  background-color: #f78989;
}

.settings-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.settings-popup {
  background: white;
  border-radius: 8px;
  width: 400px;
  max-height: 80vh;
  overflow-y: auto;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.15);
}

.settings-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  border-bottom: 1px solid #ebeef5;
}

.settings-header h3 {
  margin: 0;
  font-size: 16px;
  color: #303133;
}

.settings-close {
  background: none;
  border: none;
  font-size: 18px;
  cursor: pointer;
  color: #909399;
}

.settings-close:hover {
  color: #303133;
}

.settings-content {
  padding: 20px;
}

.settings-section {
  margin-bottom: 20px;
}

.settings-section h4 {
  margin: 0 0 12px;
  font-size: 14px;
  color: #606266;
}

.column-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.column-item label {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  font-size: 14px;
  color: #303133;
}

.column-item input[type="checkbox"] {
  cursor: pointer;
}
</style>