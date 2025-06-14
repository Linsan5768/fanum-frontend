<template>
  <div class="admin-dashboard-container">
    <div class="main-card">
      <div class="back-arrow" @click="goBack">
        <span class="arrow-icon">←</span>
        <span class="back-text">Back</span>
      </div>

      <div class="header-info">
        <h1>Admin Audit Logs</h1>
        <div class="timezone-display">
          <span>Time Zone: {{ currentTimeZone }}</span>
          <span class="time-display">Current Time: {{ currentTime }}</span>
        </div>
      </div>
      
      <div class="dashboard-controls">
        <div class="search-filters">
          <button class="btn-filter" @click="toggleFilterOverlay">
            <span class="filter-icon">🔍</span> Filters
          </button>
          
          <div class="active-filters" v-if="hasActiveFilters">
            <div class="filter-tag" v-if="filters.action">
              Action: {{ formatAction(filters.action) }}
              <span class="remove-filter" @click="removeFilter('action')">✕</span>
            </div>
            <div class="filter-tag" v-if="filters.role">
              Role: {{ filters.role }}
              <span class="remove-filter" @click="removeFilter('role')">✕</span>
            </div>
            <div class="filter-tag" v-if="filters.fromDate">
              From: {{ filters.fromDate }}
              <span class="remove-filter" @click="removeFilter('fromDate')">✕</span>
            </div>
            <div class="filter-tag" v-if="filters.toDate">
              To: {{ filters.toDate }}
              <span class="remove-filter" @click="removeFilter('toDate')">✕</span>
            </div>
            <div class="filter-tag clear-all" @click="resetFilters">
              Clear All
            </div>
          </div>
        </div>
        
        <div class="action-controls">
          <button class="btn-export" @click="toggleExportOverlay">
            <span class="export-icon">📤</span> Export
          </button>
          
          <button class="btn-refresh" @click="loadAuditLogs">
            <span class="refresh-icon">↻</span> Refresh
          </button>
          
          <label class="auto-refresh-label">
            <input type="checkbox" v-model="autoRefresh" />
            Auto-refresh (30s)
          </label>
        </div>
      </div>
      
      <!-- Loading and Error Messages -->
      <div v-if="loading" class="loading-indicator">
        Loading audit logs...
      </div>
      
      <div v-if="error" class="error-message">
        {{ error }}
      </div>
      
      <!-- Audit Logs Table -->
      <div class="table-container">
        <table class="audit-logs-table">
          <thead>
            <tr>
              <th>ID</th>
              <th>Timestamp</th>
              <th>User</th>
              <th>Role</th>
              <th>Action</th>
              <th>IP Address</th>
              <th>Details</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="log in auditLogs" :key="log.id" :class="getActionClass(log.action)">
              <td>{{ log.id }}</td>
              <td>{{ log.timestamp }}</td>
              <td>{{ log.username }}</td>
              <td>{{ log.user_role }}</td>
              <td>{{ formatAction(log.action) }}</td>
              <td>{{ log.ip_address || '-' }}</td>
              <td class="details-cell">{{ log.details || '-' }}</td>
            </tr>
            
            <tr v-if="auditLogs.length === 0 && !loading">
              <td colspan="7" class="no-data">No audit logs found</td>
            </tr>
          </tbody>
        </table>
      </div>
      
      <!-- Pagination Controls -->
      <div class="pagination-controls" v-if="totalPages > 0">
        <button 
          class="btn-page" 
          :disabled="currentPage === 1"
          @click="changePage(currentPage - 1)"
        >
          &lt; Prev
        </button>
        
        <span class="page-info">
          Page {{ currentPage }} of {{ totalPages }}
        </span>
        
        <button 
          class="btn-page" 
          :disabled="currentPage === totalPages"
          @click="changePage(currentPage + 1)"
        >
          Next &gt;
        </button>
      </div>
    </div>
    
    <!-- Filter Overlay -->
    <div class="overlay-backdrop" v-if="showFilterOverlay" @click="toggleFilterOverlay"></div>
    <div class="filter-overlay" v-if="showFilterOverlay">
      <div class="overlay-header">
        <h3>Filter Audit Logs</h3>
        <button class="close-overlay" @click="toggleFilterOverlay">✕</button>
      </div>
      
      <div class="overlay-content">
        <div class="filter-section">
          <h4>User Role</h4>
          <select v-model="tempFilters.role" class="filter-control">
            <option value="">All Roles</option>
            <option value="admin">Admin</option>
            <option value="individual">Individual</option>
            <option value="business">Business</option>
          </select>
        </div>
        
        <div class="filter-section">
          <h4>Action Type</h4>
          <select v-model="tempFilters.action" class="filter-control">
            <option value="">All Actions</option>
            <option value="login">Login</option>
            <option value="logout">Logout</option>
            <option value="FORM_SUBMIT">Form Submit</option>
            <option value="DRAFT_SAVE">Draft Save</option>
            <option value="DRAFT_UPDATE">Draft Update</option>
            <option value="VIEW_FORMS">View Forms</option>
            <option value="view_audit_logs">View Audit Logs</option>
            <option value="view_users">View Users</option>
            <option value="update_user">Update User</option>
          </select>
        </div>
        
        <div class="filter-section">
          <h4>Date Range</h4>
          <div class="date-range-inputs">
            <div class="date-input-group">
              <label>From</label>
              <input type="date" v-model="tempFilters.fromDate" class="filter-control" />
            </div>
            <div class="date-input-group">
              <label>To</label>
              <input type="date" v-model="tempFilters.toDate" class="filter-control" />
            </div>
          </div>
        </div>
      </div>
      
      <div class="overlay-footer">
        <button class="btn-cancel" @click="toggleFilterOverlay">Cancel</button>
        <button class="btn-reset" @click="clearTempFilters">Reset</button>
        <button class="btn-apply" @click="applyFilters">Apply Filters</button>
      </div>
    </div>
    
    <!-- Export Overlay -->
    <div class="overlay-backdrop" v-if="showExportOverlay" @click="toggleExportOverlay"></div>
    <div class="export-overlay" v-if="showExportOverlay">
      <div class="overlay-header">
        <h3>Export Audit Logs</h3>
        <button class="close-overlay" @click="toggleExportOverlay">✕</button>
      </div>
      
      <div class="overlay-content">
        <p class="export-info">Export current filtered logs ({{ auditLogs.length }} records)</p>
        
        <div class="export-options">
          <button class="btn-export-option" @click="exportAsPDF">
            <span class="export-icon">📑</span>
            Export as PDF
          </button>
          
          <button class="btn-export-option" @click="exportAsCSV">
            <span class="export-icon">📊</span>
            Export as CSV
          </button>
        </div>
      </div>
      
      <div class="overlay-footer">
        <button class="btn-cancel" @click="toggleExportOverlay">Cancel</button>
      </div>
    </div>
  </div>
</template>

<script>
import { getAuditLogs, formatAction, getActionClass, exportAuditLogsToPDF, exportAuditLogsToCSV, convertToUTCTime } from '../services/auditService';

export default {
  name: 'AdminDashboard',
  data() {
    return {
      auditLogs: [],
      loading: false,
      error: null,
      currentPage: 1,
      totalPages: 0,
      perPage: 15,
      autoRefresh: false,
      refreshInterval: null,
      filters: {
        action: '',
        fromDate: '',
        toDate: '',
        userId: '',
        role: ''
      },
      tempFilters: {
        action: '',
        fromDate: '',
        toDate: '',
        userId: '',
        role: ''
      },
      showFilterOverlay: false,
      showExportOverlay: false,
      timeDisplayInterval: null,
      currentTimeDisplay: new Date().toLocaleString()
    };
  },
  computed: {
    hasActiveFilters() {
      return this.filters.action || this.filters.fromDate || this.filters.toDate || this.filters.role;
    },
    currentTimeZone() {
      return Intl.DateTimeFormat().resolvedOptions().timeZone;
    },
    currentTime() {
      return this.currentTimeDisplay;
    }
  },
  methods: {
    async loadAuditLogs() {
      this.loading = true;
      this.error = null;
      
      try {
        const params = {
          page: this.currentPage,
          per_page: 15
        };
        
        // Add filters if they exist
        if (this.filters.action) params.action = this.filters.action;
        
        // 日期需要转换为UTC格式后再传给后端
        if (this.filters.fromDate) {
          // 将日期转换为该日期的00:00:00 (UTC)
          const fromDate = new Date(this.filters.fromDate);
          params.from_date = convertToUTCTime(fromDate); // 转换为UTC时间
        }
        
        if (this.filters.toDate) {
          // 将日期转换为该日期的23:59:59 (UTC)
          const toDate = new Date(this.filters.toDate);
          params.to_date = convertToUTCTime(toDate); // 转换为UTC时间
        }
        
        if (this.filters.userId) params.user_id = this.filters.userId;
        if (this.filters.role) params.user_role = this.filters.role;
        
        const response = await getAuditLogs(params);
        
        if (response.success) {
          this.auditLogs = response.logs;
          this.totalPages = response.total_pages;
          
          // Adjust currentPage if it's out of bounds
          if (this.currentPage > this.totalPages && this.totalPages > 0) {
            this.currentPage = this.totalPages;
            this.loadAuditLogs();
          }
        } else {
          this.error = response.message || 'Failed to load audit logs';
        }
      } catch (err) {
        console.error('Error loading audit logs:', err);
        if (err.response) {
          if (err.response.status === 401) {
            // 未授权，重定向到登录页面
            this.$router.push('/login');
            return;
          } else if (err.response.status === 403) {
            // 用户没有权限访问
            this.error = '您没有访问管理员仪表盘的权限。只有管理员用户才能访问此页面。';
            setTimeout(() => {
              this.$router.push('/home');
            }, 3000); // 3秒后重定向到首页
            return;
          }
        }
        this.error = 'Error loading audit logs. Please try again later.';
      } finally {
        this.loading = false;
      }
    },
    changePage(page) {
      this.currentPage = page;
      this.loadAuditLogs();
    },
    resetFilters() {
      this.filters = {
        action: '',
        fromDate: '',
        toDate: '',
        userId: '',
        role: ''
      };
      this.currentPage = 1;
      this.loadAuditLogs();
    },
    removeFilter(filterName) {
      this.filters[filterName] = '';
      this.currentPage = 1;
      this.loadAuditLogs();
    },
    toggleFilterOverlay() {
      if (!this.showFilterOverlay) {
        // 复制当前筛选条件到临时筛选条件
        this.tempFilters = { ...this.filters };
      }
      this.showFilterOverlay = !this.showFilterOverlay;
    },
    toggleExportOverlay() {
      this.showExportOverlay = !this.showExportOverlay;
    },
    applyFilters() {
      this.filters = { ...this.tempFilters };
      this.currentPage = 1;
      this.loadAuditLogs();
      this.toggleFilterOverlay();
    },
    clearTempFilters() {
      this.tempFilters = {
        action: '',
        fromDate: '',
        toDate: '',
        userId: '',
        role: ''
      };
    },
    async exportAsPDF() {
      try {
        this.loading = true;
        
        // 准备查询参数，使用当前筛选条件
        const params = {};
        if (this.filters.action) params.action = this.filters.action;
        if (this.filters.fromDate) params.from_date = this.filters.fromDate;
        if (this.filters.toDate) params.to_date = this.filters.toDate;
        if (this.filters.userId) params.user_id = this.filters.userId;
        if (this.filters.role) params.user_role = this.filters.role;
        
        // 调用API获取数据
        const response = await exportAuditLogsToPDF(params);
        
        if (response.success) {
          const logs = response.logs;
          
          // 导入html2pdf库
          const html2pdf = require('html2pdf.js');
          
          // 创建HTML内容
          const element = document.createElement('div');
          element.innerHTML = `
            <html>
              <head>
                <title>Audit Logs - ${new Date().toLocaleDateString()}</title>
                <style>
                  body { font-family: Arial, sans-serif; }
                  table { width: 100%; border-collapse: collapse; }
                  th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
                  th { background-color: #f2f2f2; }
                  h1 { text-align: center; }
                  .filters { margin-bottom: 20px; font-size: 14px; }
                </style>
              </head>
              <body>
                <h1>Audit Logs</h1>
                <div class="filters">
                  <p><strong>Generated:</strong> ${new Date().toLocaleString()} (Local Time)</p>
                  <p><strong>Time Zone:</strong> ${Intl.DateTimeFormat().resolvedOptions().timeZone}</p>
                  ${this.filters.action ? `<p><strong>Action:</strong> ${formatAction(this.filters.action)}</p>` : ''}
                  ${this.filters.role ? `<p><strong>Role:</strong> ${this.filters.role}</p>` : ''}
                  ${this.filters.fromDate ? `<p><strong>From Date:</strong> ${this.filters.fromDate}</p>` : ''}
                  ${this.filters.toDate ? `<p><strong>To Date:</strong> ${this.filters.toDate}</p>` : ''}
                </div>
                <table>
                  <thead>
                    <tr>
                      <th>ID</th>
                      <th>Timestamp</th>
                      <th>User</th>
                      <th>Role</th>
                      <th>Action</th>
                      <th>IP Address</th>
                      <th>Details</th>
                    </tr>
                  </thead>
                  <tbody>
                    ${logs.map(log => `
                      <tr>
                        <td>${log.id}</td>
                        <td>${log.timestamp}</td>
                        <td>${log.username}</td>
                        <td>${log.user_role}</td>
                        <td>${formatAction(log.action)}</td>
                        <td>${log.ip_address || '-'}</td>
                        <td>${log.details || '-'}</td>
                      </tr>
                    `).join('')}
                  </tbody>
                </table>
              </body>
            </html>
          `;
          
          // 配置PDF选项
          const opt = {
            margin: 10,
            filename: `audit_logs_${new Date().toISOString().slice(0, 10)}.pdf`,
            image: { type: 'jpeg', quality: 0.98 },
            html2canvas: { scale: 2 },
            jsPDF: { unit: 'mm', format: 'a4', orientation: 'landscape' }
          };
          
          // 生成并下载PDF
          html2pdf().from(element).set(opt).save();
        } else {
          alert('Failed to export logs: ' + (response.message || 'Unknown error'));
        }
      } catch (error) {
        console.error('Error exporting as PDF:', error);
        alert('Error exporting logs. Please try again later.');
      } finally {
        this.loading = false;
        this.toggleExportOverlay();
      }
    },
    async exportAsCSV() {
      try {
        this.loading = true;
        
        // 准备查询参数
        const params = {};
        if (this.filters.action) params.action = this.filters.action;
        if (this.filters.fromDate) params.from_date = this.filters.fromDate;
        if (this.filters.toDate) params.to_date = this.filters.toDate;
        if (this.filters.userId) params.user_id = this.filters.userId;
        if (this.filters.role) params.user_role = this.filters.role;
        
        // 调用API
        const response = await exportAuditLogsToCSV(params);
        
        if (response.success) {
          const logs = response.logs;
          
          // 准备数据
          const headers = ['ID', 'Timestamp', 'User', 'Role', 'Action', 'IP Address', 'Details'];
          const csvData = logs.map(log => [
            log.id,
            log.timestamp,
            log.username,
            log.user_role,
            formatAction(log.action),
            log.ip_address || '',
            log.details || ''
          ]);
          
          // 添加标题行
          csvData.unshift(headers);
          
          // 转为CSV格式
          const csvContent = csvData.map(row => 
            row.map(cell => `"${String(cell).replace(/"/g, '""')}"`).join(',')
          ).join('\n');
          
          // 创建下载链接
          const encodedUri = encodeURI('data:text/csv;charset=utf-8,' + csvContent);
          const link = document.createElement('a');
          link.setAttribute('href', encodedUri);
          link.setAttribute('download', `audit_logs_${new Date().toISOString().slice(0, 10)}.csv`);
          document.body.appendChild(link);
          
          // 触发下载
          link.click();
          document.body.removeChild(link);
        } else {
          alert('Failed to export logs: ' + (response.message || 'Unknown error'));
        }
      } catch (error) {
        console.error('Error exporting as CSV:', error);
        alert('Error exporting logs. Please try again later.');
      } finally {
        this.loading = false;
        this.toggleExportOverlay();
      }
    },
    formatAction(action) {
      return formatAction(action);
    },
    getActionClass(action) {
      return getActionClass(action);
    },
    goBack() {
      this.$router.go(-1);
    },
    startAutoRefresh() {
      // Clear any existing interval
      if (this.refreshInterval) {
        clearInterval(this.refreshInterval);
      }
      
      // Set new interval if autoRefresh is enabled
      if (this.autoRefresh) {
        this.refreshInterval = setInterval(() => {
          this.loadAuditLogs();
        }, 30000); // 30 seconds
      }
    }
  },
  watch: {
    autoRefresh() {
      this.startAutoRefresh();
    }
  },
  mounted() {
    // 加载审计日志
    this.loadAuditLogs();
    
    // 更新当前时间
    this.timeDisplayInterval = setInterval(() => {
      this.currentTimeDisplay = new Date().toLocaleString();
    }, 1000);
  },
  beforeUnmount() {
    // 清除自动刷新间隔
    if (this.refreshInterval) {
      clearInterval(this.refreshInterval);
    }
    
    // 清除时间显示间隔
    if (this.timeDisplayInterval) {
      clearInterval(this.timeDisplayInterval);
    }
  }
};
</script>

<style scoped>
.admin-dashboard-container {
  padding: 18px;
  display: flex;
  justify-content: center;
  align-items: flex-start;
  min-height: 100vh;
  background-color: #E5E5E5;
}

.main-card {
  position: relative;
  background-color: #E1E1E1;
  border-radius: 10px;
  padding: 24px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
  width: 100%;
  max-width: 1320px;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  min-height: 80vh;
  max-height: 90vh;
  overflow-y: auto;
}

.back-arrow {
  position: absolute;
  top: 15px;
  left: 15px;
  font-size: 16px;
  color: #333;
  cursor: pointer;
  z-index: 10;
  display: flex;
  align-items: center;
  justify-content: flex-start;
  gap: 4px;
  background-color: rgba(255, 255, 255, 0.3);
  border-radius: 20px;
  padding: 5px 12px;
  transition: background-color 0.2s, transform 0.2s;
}

.back-arrow:hover {
  background-color: rgba(0, 0, 0, 0.1);
  transform: scale(1.05);
}

.arrow-icon {
  font-size: 18px;
  font-weight: bold;
}

.back-text {
  font-weight: 500;
}

.dashboard-title {
  text-align: center;
  margin-top: 0;
  margin-bottom: 20px;
  font-size: 24px;
  color: #333;
  font-weight: bold;
}

.dashboard-controls {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
  align-items: flex-start;
  margin-top: 20px;
  margin-bottom: 20px;
  gap: 10px;
}

.search-filters {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  align-items: center;
}

.active-filters {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  align-items: center;
}

.filter-tag {
  background-color: #E8F4FF;
  color: #0e5ba6;
  border-radius: 20px;
  padding: 5px 10px;
  font-size: 13px;
  display: flex;
  align-items: center;
  gap: 5px;
}

.clear-all {
  background-color: #f0f0f0;
  color: #555;
  cursor: pointer;
}

.clear-all:hover {
  background-color: #e0e0e0;
}

.remove-filter {
  cursor: pointer;
  font-weight: bold;
  margin-left: 2px;
}

.remove-filter:hover {
  color: #e74c3c;
}

.action-controls {
  display: flex;
  align-items: center;
  gap: 10px;
}

.refresh-controls {
  display: flex;
  align-items: center;
  gap: 10px;
}

.filter-group {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

label {
  font-size: 12px;
  font-weight: bold;
  color: #555;
}

.filter-control {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 5px;
  background-color: white;
  font-size: 14px;
  min-width: 120px;
}

.btn-filter, .btn-export, .btn-refresh, .btn-apply-filter, .btn-reset-filter {
  padding: 8px 12px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 5px;
  transition: all 0.2s;
}

.btn-filter {
  background-color: #1F3A93;
  color: white;
}

.btn-export {
  background-color: #2980b9;
  color: white;
}

.btn-refresh {
  background-color: #4caf50;
  color: white;
}

.btn-filter:hover {
  background-color: #142c70;
}

.btn-export:hover {
  background-color: #1c638e;
}

.btn-refresh:hover {
  background-color: #388e3c;
}

.auto-refresh-label {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 14px;
  color: #666;
  cursor: pointer;
}

.table-container {
  overflow-x: auto;
  margin-bottom: 20px;
  background-color: white;
  border-radius: 5px;
  box-shadow: 0 2px 5px rgba(0,0,0,0.05);
  flex: 1;
  max-height: calc(70vh - 200px);
  overflow-y: auto;
}

.audit-logs-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 14px;
}

.audit-logs-table th {
  background-color: #f5f5f5;
  padding: 12px 8px;
  text-align: left;
  font-weight: 600;
  color: #333;
  border-bottom: 1px solid #ddd;
  position: sticky;
  top: 0;
}

.audit-logs-table td {
  padding: 10px 8px;
  border-bottom: 1px solid #eee;
  color: #555;
}

.audit-logs-table tr:last-child td {
  border-bottom: none;
}

.audit-logs-table tr:hover td {
  background-color: #f9f9f9;
}

.details-cell {
  max-width: 300px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.details-cell:hover {
  white-space: normal;
  overflow: visible;
  text-overflow: unset;
  word-break: break-word;
}

.action-login {
  background-color: rgba(76, 175, 80, 0.1);
}

.action-logout {
  background-color: rgba(33, 150, 243, 0.1);
}

.action-submit {
  background-color: rgba(255, 152, 0, 0.1);
}

.action-draft {
  background-color: rgba(156, 39, 176, 0.1);
}

.action-view {
  background-color: rgba(139, 195, 74, 0.1);
}

.action-admin {
  background-color: rgba(156, 39, 176, 0.15);
}

.loading-indicator, .error-message, .no-data {
  text-align: center;
  padding: 20px;
  color: #666;
}

.error-message {
  color: #f44336;
}

.pagination-controls {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 10px;
  margin-top: 20px;
  margin-bottom: 10px;
  position: sticky;
  bottom: 0;
  background-color: rgba(225, 225, 225, 0.9);
  padding: 10px 0;
  z-index: 10;
}

.btn-page {
  padding: 5px 10px;
  border: 1px solid #ddd;
  background-color: white;
  border-radius: 5px;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-page:hover:not(:disabled) {
  background-color: #f0f0f0;
}

.btn-page:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}

.page-info {
  font-size: 14px;
  color: #666;
}

/* Overlay Styles */
.overlay-backdrop {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  z-index: 100;
}

.filter-overlay, .export-overlay {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background-color: white;
  border-radius: 10px;
  width: 90%;
  max-width: 500px;
  max-height: 90vh;
  overflow-y: auto;
  z-index: 101;
  display: flex;
  flex-direction: column;
  box-shadow: 0 5px 20px rgba(0, 0, 0, 0.2);
}

.overlay-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  border-bottom: 1px solid #eee;
}

.overlay-header h3 {
  margin: 0;
  font-size: 18px;
  color: #333;
}

.close-overlay {
  background: none;
  border: none;
  font-size: 20px;
  cursor: pointer;
  color: #888;
}

.close-overlay:hover {
  color: #333;
}

.overlay-content {
  padding: 20px;
  flex-grow: 1;
  overflow-y: auto;
}

.filter-section {
  margin-bottom: 20px;
}

.filter-section h4 {
  margin: 0 0 10px 0;
  font-size: 16px;
  color: #555;
}

.date-range-inputs {
  display: flex;
  gap: 10px;
}

.date-input-group {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.overlay-footer {
  padding: 15px 20px;
  border-top: 1px solid #eee;
  display: flex;
  justify-content: flex-end;
  gap: 10px;
}

.btn-cancel, .btn-reset, .btn-apply {
  padding: 8px 16px;
  border-radius: 5px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-cancel {
  background-color: #f0f0f0;
  border: 1px solid #ddd;
  color: #555;
}

.btn-reset {
  background-color: #f5f5f5;
  border: 1px solid #ddd;
  color: #666;
}

.btn-apply {
  background-color: #1F3A93;
  border: 1px solid #1F3A93;
  color: white;
}

.btn-cancel:hover {
  background-color: #e0e0e0;
}

.btn-reset:hover {
  background-color: #e5e5e5;
}

.btn-apply:hover {
  background-color: #142c70;
}

.export-info {
  margin-bottom: 20px;
  color: #666;
}

.export-options {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.btn-export-option {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 15px;
  border: 1px solid #ddd;
  border-radius: 8px;
  background-color: #f9f9f9;
  cursor: pointer;
  transition: all 0.2s;
  font-weight: 600;
  font-size: 16px;
  color: #333;
}

.btn-export-option:hover {
  background-color: #f0f0f0;
  border-color: #ccc;
}

.export-icon {
  font-size: 20px;
}

@media (max-width: 768px) {
  .search-filters, .refresh-controls {
    flex-direction: column;
    width: 100%;
  }
  
  .filter-control {
    width: 100%;
  }
  
  .table-container {
    font-size: 12px;
  }
  
  .audit-logs-table th, .audit-logs-table td {
    padding: 8px 4px;
  }
  
  .details-cell {
    max-width: 150px;
  }
  
  .filter-overlay, .export-overlay {
    width: 95%;
  }
  
  .date-range-inputs {
    flex-direction: column;
  }
}

.header-info {
  margin-bottom: 20px;
}

.header-info h1 {
  margin-bottom: 10px;
  color: var(--text-main);
}

.timezone-display {
  display: flex;
  justify-content: space-between;
  font-size: 0.9rem;
  color: var(--text-subtle);
  background-color: rgba(31, 58, 147, 0.05);
  padding: 8px 12px;
  border-radius: 4px;
  margin-bottom: 15px;
}

.time-display {
  font-weight: 500;
}
</style> 