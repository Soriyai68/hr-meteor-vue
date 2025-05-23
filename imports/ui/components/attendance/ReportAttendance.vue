<template>
  <div class="relative">
    <!-- Controls -->
    <div class="flex items-center mb-4 sticky top-0 z-10 bg-white p-2 rounded-lg">
      <el-select
        v-model="searchType"
        placeholder="Select search type"
        class="m-2"
        style="width: 150px"
        @change="onSearchTypeChange"
      >
        <el-option label="All Dates" value="all" />
        <el-option label="Today" value="today" />
        <el-option label="Yesterday" value="yesterday" />
        <el-option label="Last Week" value="lastWeek" />
        <el-option label="Last Month" value="lastMonth" />
        <el-option label="Custom Date" value="custom" />
      </el-select>
      <el-date-picker
        v-if="searchType === 'custom'"
        v-model="searchDate"
        type="date"
        placeholder="Search by date"
        clearable
        class="m-2"
        style="width: 150px"
      />
      <el-select
        v-if="searchType === 'custom'"
        v-model="searchSession"
        placeholder="Select session"
        class="m-2"
        style="width: 150px"
      >
        <el-option label="All Sessions" value="" />
        <el-option label="Morning" value="Morning" />
        <el-option label="Afternoon" value="Afternoon" />
      </el-select>
      <el-button type="primary" @click="onSearch" class="m-2">
        Search
      </el-button>
      <el-button type="success" @click="printAttendance" class="m-2">
        Print
      </el-button>
    </div>

    <!-- Printable Invoice Area -->
    <div id="printableArea" class="invoice-container">
      <div class="invoice-header">
        <div class="logo">
          <img src="/images/lov.png" alt="Logo" />
        </div>
        <div class="header-text">
          <h1>Attendance Report</h1>
          <p>{{ currentDate }}</p>
        </div>
      </div>

      <div class="invoice-body">
        <el-table
          :data="filteredTableData"
          style="width: 100%"
          border
          stripe
          class="modern-table"
          v-loading="loading"
        >
          <el-table-column type="index" label="#" width="60" />
          <el-table-column label="Employee">
            <template #default="scope">
              {{ getEmployeeName(scope.row.employee_id) }}
            </template>
          </el-table-column>
          <el-table-column label="Attendance Date">
            <template #default="scope">
              {{ formatDateOnly(scope.row.attendance_date) }}
            </template>
          </el-table-column>
          <el-table-column label="In Time">
            <template #default="scope">
              {{ formatDate(scope.row.in_time) }}
            </template>
          </el-table-column>
          <el-table-column label="Out Time">
            <template #default="scope">
              {{ formatDate(scope.row.out_time) }}
            </template>
          </el-table-column>
          <el-table-column label="Status" prop="status">
            <template #default="scope">
              <span
                :class="{
                  'bg-green-200 text-green-800 px-2 py-1 rounded-md':
                    scope.row.status === 'Present',
                  'bg-red-200 text-red-800 px-2 py-1 rounded-md':
                    scope.row.status === 'Absent',
                  'bg-yellow-200 text-yellow-800 px-2 py-1 rounded-md':
                    scope.row.status === 'Late',
                }"
              >
                {{ scope.row.status }}
              </span>
            </template>
          </el-table-column>
          <el-table-column label="Session" prop="session">
            <template #default="scope">
              <span
                :class="{
                  'bg-blue-200 text-blue-800 px-2 py-1 rounded-md':
                    scope.row.session === 'Morning',
                  'bg-purple-200 text-purple-800 px-2 py-1 rounded-md':
                    scope.row.session === 'Afternoon',
                }"
              >
                {{ scope.row.session }}
              </span>
            </template>
          </el-table-column>
        </el-table>
      </div>

      <div class="invoice-footer">
        <p>Generated by Sao Soriya</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from "vue";
import { ElNotification } from "element-plus";
import { Meteor } from "meteor/meteor";
import printJS from "print-js";

const loading = ref(false);
const searchDate = ref(null);
const searchType = ref("all");
const searchSession = ref("");
const dataTable = ref([]);
const employees = ref([]);
const filteredTableData = ref([]);

const currentDate = computed(() => {
  const d = new Date();
  return d.toLocaleDateString(undefined, {
    year: "numeric",
    month: "long",
    day: "numeric",
  });
});

const notify = (message, type) => {
  ElNotification({ message, type, duration: 2000 });
};

const getData = () => {
  loading.value = true;
  Meteor.call("fetchAttendance", (err, res) => {
    loading.value = false;
    if (err) {
      notify(`Failed to fetch data: ${err.message}`, "error");
    } else {
      dataTable.value = res;
      filteredTableData.value = res;
    }
  });
};

const getEmployees = () => {
  Meteor.call("fetchEmployees", (err, res) => {
    if (err) {
      notify(`Failed to fetch employees: ${err.message}`, "error");
    } else {
      employees.value = res;
    }
  });
};

const getEmployeeName = (employeeId) => {
  const emp = employees.value.find((employee) => employee._id === employeeId);
  return emp ? emp.name : employeeId;
};

const formatDate = (date) => {
  if (!date) return "";
  const d = new Date(date);
  const day = String(d.getDate()).padStart(2, "0");
  const month = String(d.getMonth() + 1).padStart(2, "0");
  const year = d.getFullYear();
  let hours = d.getHours();
  const minutes = String(d.getMinutes()).padStart(2, "0");
  const seconds = String(d.getSeconds()).padStart(2, "0");
  const ampm = hours >= 12 ? "PM" : "AM";
  hours = hours % 12;
  hours = hours ? hours : 12; // the hour '0' should be '12'
  return `${month}/${day}/${year}, ${hours}:${minutes}:${seconds} ${ampm}`;
};

const formatDateOnly = (date) => {
  if (!date) return "";
  const d = new Date(date);
  const day = String(d.getDate()).padStart(2, "0");
  const month = String(d.getMonth() + 1).padStart(2, "0");
  const year = d.getFullYear();
  return `${month}/${day}/${year}`;
};

const onSearchTypeChange = () => {
  if (searchType.value !== "custom") {
    searchDate.value = null; // Clear date when not custom
    searchSession.value = ""; // Clear session when not custom
  }
};

const onSearch = () => {
  loading.value = true;
  let methodName;
  const params = {};

  switch (searchType.value) {
    case "today":
      methodName = "searchAttendanceToday";
      break;
    case "yesterday":
      methodName = "searchAttendanceYesterday";
      break;
    case "lastWeek":
      methodName = "searchAttendanceLastWeek";
      break;
    case "lastMonth":
      methodName = "searchAttendanceLastMonth";
      break;
    case "custom":
      if (!searchDate.value) {
        notify("Please select a date for custom search", "warning");
        loading.value = false;
        return;
      }
      methodName = "searchAttendanceByDay";
      params.date = searchDate.value;
      params.session = searchSession.value;
      break;
    case "all":
      methodName = "fetchAttendance";
      break;
    default:
      methodName = "fetchAttendance";
  }

  Meteor.call(methodName, params, (err, res) => {
    loading.value = false;
    if (err) {
      notify(`Failed to search data: ${err.message}`, "error");
    } else {
      filteredTableData.value = res;
      if (res.length === 0) {
        notify("No attendance records found", "info");
      }
    }
  });
};

const printAttendance = () => {
  printJS({
    printable: "printableArea",
    type: "html",
    style: `
      @page { margin: 20mm; }
      .invoice-container { 
        font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
        color: #333; 
        padding: 20px;
      }
      .invoice-header { 
        display: flex; 
        justify-content: space-between; 
        align-items: center; 
        border-bottom: 2px solid #eee; 
        margin-bottom: 20px;
        padding-bottom: 10px;
      }
      .invoice-header .logo img {
        max-width: 80px;
      }
      .invoice-header .header-text h1 {
        margin: 0;
        font-size: 24px;
        color: #4CAF50;
      }
      .invoice-header .header-text p {
        margin: 0;
        font-size: 14px;
        color: #777;
      }
      .invoice-body { margin-bottom: 20px; }
      .modern-table th, .modern-table td {
        border: 1px solid #ddd;
        padding: 8px;
      }
      .modern-table th {
        background-color: #f7f7f7;
        color: #333;
      }
      .invoice-footer {
        text-align: center;
        font-size: 12px;
        color: #aaa;
        border-top: 1px solid #eee;
        padding-top: 10px;
      }
    `,
    scanStyles: false,
    documentTitle: "Attendance Report",
  });
};

onMounted(() => {
  getData();
  getEmployees();
});
</script>

<style>
.mb-4 {
  margin-bottom: 1rem;
}
.ml-4 {
  margin-left: 1rem;
}

.invoice-container {
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  padding: 20px;
  margin-top: 20px;
}

.invoice-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 2px solid #eee;
  padding-bottom: 10px;
  margin-bottom: 20px;
}
.invoice-header .logo img {
  max-width: 80px;
}
.invoice-header .header-text h1 {
  margin: 0;
  font-size: 24px;
  color: #4caf50;
}
.invoice-header .header-text p {
  margin: 0;
  font-size: 14px;
  color: #777;
}

.invoice-body {
  margin-bottom: 20px;
}
.modern-table th,
.modern-table td {
  border: 1px solid #ddd;
  padding: 8px;
}
.modern-table th {
  background-color: #f7f7f7;
  color: #333;
}

.invoice-footer {
  text-align: center;
  font-size: 12px;
  color: #aaa;
  border-top: 1px solid #eee;
  padding-top: 10px;
}
</style>