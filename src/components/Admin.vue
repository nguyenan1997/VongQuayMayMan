<script setup>
import { ref, onMounted, computed } from 'vue'
import { Users, Search, RefreshCw, Trophy, Ban, CheckCircle, ArrowLeft, Download, Moon, Sun } from 'lucide-vue-next'
import { API_ENDPOINTS } from '../configs/api'

const props = defineProps(['token', 'isAdminRole'])
const emit = defineEmits(['back', 'logout'])

const users = ref([])
const loading = ref(false)
const searchQuery = ref('')
const isDark = ref(localStorage.getItem('admin_theme') === 'dark')

const toggleTheme = () => {
  isDark.value = !isDark.value
  localStorage.setItem('admin_theme', isDark.value ? 'dark' : 'light')
}

const fetchAllUsers = async () => {
  loading.value = true
  try {
    const response = await fetch(API_ENDPOINTS.GET_RESULTS, {
      headers: {
        'Authorization': `Bearer ${props.token}`
      }
    })
    const result = await response.json()
    if (result.success) {
      users.value = result.data
    }
  } catch (err) {
    console.error('Lỗi lấy danh sách user:', err)
  } finally {
    loading.value = false
  }
}

const filteredUsers = computed(() => {
  if (!searchQuery.value) return users.value
  const q = searchQuery.value.toLowerCase()
  return users.value.filter(u => 
    u.fullName.toLowerCase().includes(q) || 
    u.phoneNumber.includes(q)
  )
})

onMounted(fetchAllUsers)

const formatDate = (dateStr) => {
  if (!dateStr) return '---'
  const date = new Date(dateStr)
  return date.toLocaleString('vi-VN')
}

const exportCSV = () => {
  const headers = ['Họ Tên', 'Số Điện Thoại', 'Lượt Quay', 'Kết Quả', 'Thời Gian Quay']
  const rows = filteredUsers.value.map(u => [
    u.fullName,
    u.phoneNumber,
    u.spinCount,
    u.spinResult || 'Chưa quay',
    formatDate(u.lastSpinAt)
  ])
  
  let csvContent = "data:text/csv;charset=utf-8,\uFEFF" 
    + headers.join(",") + "\n" 
    + rows.map(e => e.join(",")).join("\n")

  const encodedUri = encodeURI(csvContent)
  const link = document.createElement("a")
  link.setAttribute("href", encodedUri)
  link.setAttribute("download", `danh_sach_trung_thuong_${new Date().getTime()}.csv`)
  document.body.appendChild(link)
  link.click()
  document.body.removeChild(link)
}
</script>

<template>
  <div :class="[isDark ? 'bg-slate-950 text-slate-100' : 'bg-white text-slate-950']" class="min-h-screen p-4 sm:p-10 font-sans tracking-tight transition-colors duration-300">
    <div class="max-w-7xl mx-auto space-y-12">
      
      <!-- Header Section -->
      <div class="flex flex-col md:flex-row justify-between items-start md:items-center gap-8">
        <div>
          <h1 class="text-3xl font-bold tracking-tight flex items-center gap-3.5" :class="isDark ? 'text-white' : 'text-slate-900'">
            <div class="w-11 h-11 bg-indigo-600 rounded-2xl flex items-center justify-center shadow-xl shadow-indigo-500/10">
              <Trophy class="w-5 h-5 text-white" />
            </div>
            Quản trị hoạt động
          </h1>
          <p class="mt-1.5 font-medium text-[15px] opacity-60">Theo dõi thông tin người chơi hái lộc Xuân Ất Tỵ</p>
        </div>

        <div class="flex flex-wrap items-center gap-3.5 w-full md:w-auto">
          <!-- Theme Toggle -->
          <button @click="toggleTheme" class="p-3 rounded-2xl border transition-all active:scale-95 duration-200" 
            :class="isDark ? 'bg-slate-900 border-slate-800 text-yellow-400 hover:bg-slate-800' : 'bg-white border-slate-200 text-slate-500 shadow-sm hover:bg-slate-50'">
            <Sun v-if="isDark" class="w-[18px] h-[18px]" />
            <Moon v-else class="w-[18px] h-[18px]" />
          </button>

          <button @click="exportCSV" class="flex-1 md:flex-none flex items-center justify-center gap-2.5 px-6 py-3 rounded-2xl font-semibold text-[14px] transition-all active:scale-95 border duration-200"
            :class="isDark ? 'bg-slate-900 border-slate-800 text-slate-300 hover:bg-slate-800' : 'bg-white border-slate-200 text-slate-700 hover:bg-slate-50'">
            <Download class="w-4 h-4" /> Xuất dữ liệu
          </button>
          
          <button @click="fetchAllUsers" :disabled="loading" class="flex-1 md:flex-none flex items-center justify-center gap-2.5 px-6 py-3 bg-indigo-600 text-white rounded-2xl font-semibold text-[14px] hover:bg-indigo-700 transition-all shadow-xl active:scale-95 disabled:opacity-50 shadow-indigo-500/15 duration-200">
            <RefreshCw class="w-4 h-4" :class="{'animate-spin': loading}" /> Làm mới
          </button>
          
          <button @click="emit('logout')" class="flex-1 md:flex-none flex items-center justify-center gap-2.5 px-6 py-3 rounded-2xl font-semibold text-[14px] transition-all active:scale-95 border duration-200"
            :class="isDark ? 'bg-red-500/5 border-red-500/20 text-red-400 hover:bg-red-500/10' : 'bg-red-50 border-red-100 text-red-600 hover:bg-red-100'">
            Đăng xuất
          </button>
        </div>
      </div>

      <!-- Stats Grid -->
      <div class="grid grid-cols-1 sm:grid-cols-3 gap-8">
        <div v-for="(stat, idx) in [
          { label: 'Tổng người chơi', value: users.length, icon: Users, color: 'text-indigo-500', bg: 'bg-indigo-500' },
          { label: 'Đã hái lộc', value: users.filter(u => u.spinResult).length, icon: CheckCircle, color: 'text-emerald-500', bg: 'bg-emerald-500' },
          { label: 'Chưa tham gia', value: users.filter(u => !u.spinResult).length, icon: Ban, color: 'text-rose-500', bg: 'bg-rose-500' }
        ]" :key="idx" 
        class="p-8 rounded-[32px] border transition-all duration-300 relative overflow-hidden group"
        :class="isDark ? 'bg-slate-900/40 border-slate-800 hover:border-slate-700' : 'bg-slate-50 border-slate-100 hover:bg-white hover:shadow-2xl hover:shadow-slate-200/50 hover:border-transparent'">
          <div class="flex items-center gap-5 mb-5 relative z-10">
            <div class="w-12 h-12 rounded-2xl flex items-center justify-center transition-all duration-300 shadow-sm"
              :class="isDark ? `bg-slate-800 ${stat.color}` : `${stat.bg}/10 ${stat.color}`">
              <component :is="stat.icon" class="w-5 h-5" />
            </div>
            <span class="font-semibold text-[13px] uppercase tracking-[0.05em] opacity-50">{{ stat.label }}</span>
          </div>
          <div class="text-4xl font-bold tracking-tight relative z-10" :class="isDark ? 'text-white' : 'text-slate-900'">{{ stat.value }}</div>
        </div>
      </div>

      <!-- Main Content Block -->
      <div class="rounded-[2rem] shadow-sm border overflow-hidden transition-all"
        :class="isDark ? 'bg-slate-900 border-slate-800' : 'bg-white border-slate-100'">
        
        <!-- Search Bar -->
        <div class="p-6 border-b flex flex-col sm:flex-row gap-4 justify-between transition-colors"
          :class="isDark ? 'bg-slate-900/50 border-slate-800' : 'bg-white border-slate-50'">
          <div class="relative flex-1 group">
            <Search class="absolute left-4 top-1/2 -translate-y-1/2 w-5 h-5 transition-colors" 
              :class="isDark ? 'text-slate-600 group-focus-within:text-indigo-400' : 'text-slate-400 group-focus-within:text-indigo-600'"/>
            <input 
              v-model="searchQuery"
              type="text" 
              placeholder="Tìm theo tên hoặc số điện thoại..." 
              class="w-full border rounded-2xl py-3.5 pl-12 pr-6 focus:ring-4 transition-all placeholder:text-slate-500 font-medium outline-none"
              :class="isDark 
                ? 'bg-slate-950 border-slate-800 focus:border-indigo-500 focus:ring-indigo-500/10 text-white' 
                : 'bg-slate-50 border-slate-100 focus:bg-white focus:border-indigo-600 focus:ring-indigo-50'"
            />
          </div>
        </div>

        <!-- Table View -->
        <div class="overflow-x-auto">
          <table class="w-full text-left">
            <thead>
              <tr class="text-xs font-bold uppercase tracking-widest border-b"
                :class="isDark ? 'text-slate-500 bg-slate-950/30 border-slate-800' : 'text-slate-400 bg-slate-50/50 border-slate-50'">
                <th class="px-8 py-5">Người chơi</th>
                <th class="px-8 py-5">Số điện thoại</th>
                <th class="px-8 py-5">Kết quả</th>
                <th class="px-8 py-5 text-center">Trạng thái</th>
                <th class="px-8 py-5 text-right">Ngày giờ quay</th>
              </tr>
            </thead>
            <tbody class="divide-y" :class="isDark ? 'divide-slate-800' : 'divide-slate-50'">
              <tr v-for="user in filteredUsers" :key="user.id" class="transition-all group"
                :class="isDark ? 'hover:bg-slate-800/50' : 'hover:bg-slate-50/80'">
                <td class="px-8 py-5">
                  <div class="flex items-center gap-3">
                    <div class="w-10 h-10 rounded-full flex items-center justify-center font-bold border"
                      :class="isDark ? 'bg-slate-800 text-indigo-400 border-slate-700' : 'bg-indigo-50 text-indigo-600 border-indigo-100'">
                      {{ user.fullName.charAt(0) }}
                    </div>
                    <span class="font-bold whitespace-nowrap" :class="isDark ? 'text-slate-200' : 'text-slate-800'">{{ user.fullName }}</span>
                  </div>
                </td>
                <td class="px-8 py-5 font-medium" :class="isDark ? 'text-slate-400' : 'text-slate-600'">{{ user.phoneNumber }}</td>
                <td class="px-8 py-5">
                  <span v-if="user.spinResult" class="px-3 py-1.5 font-bold rounded-lg text-xs uppercase border shadow-sm"
                    :class="isDark ? 'bg-indigo-900/30 text-indigo-300 border-indigo-800' : 'bg-indigo-50 text-indigo-700 border-indigo-100 shadow-indigo-100/50'">
                    {{ user.spinResult }}
                  </span>
                  <span v-else class="italic text-sm" :class="isDark ? 'text-slate-700' : 'text-slate-300'">Chưa có lộc</span>
                </td>
                <td class="px-8 py-5">
                  <div class="flex justify-center">
                    <span v-if="user.spinResult" class="flex items-center gap-1.5 font-bold text-[10px] uppercase"
                      :class="isDark ? 'text-slate-600' : 'text-slate-300'">
                      <div class="w-1.5 h-1.5 rounded-full" :class="isDark ? 'bg-slate-700' : 'bg-slate-300'"></div>
                      Hết lượt
                    </span>
                    <span v-else class="flex items-center gap-1.5 text-emerald-500 font-bold text-[10px] uppercase">
                      <div class="w-1.5 h-1.5 rounded-full bg-emerald-500 animate-pulse"></div>
                      Khả dụng
                    </span>
                  </div>
                </td>
                <td class="px-8 py-5 text-right text-sm font-medium" :class="isDark ? 'text-slate-600' : 'text-slate-400'">
                  {{ formatDate(user.lastSpinAt) }}
                </td>
              </tr>
              <tr v-if="filteredUsers.length === 0">
                <td colspan="5" class="px-8 py-24 text-center">
                  <div class="flex flex-col items-center gap-2 opacity-20">
                    <Users class="w-12 h-12" />
                    <p class="font-medium italic">Không tìm thấy ai phù hợp...</p>
                  </div>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.custom-scrollbar::-webkit-scrollbar {
  width: 6px;
}
.custom-scrollbar::-webkit-scrollbar-track {
  background: transparent;
}
.custom-scrollbar::-webkit-scrollbar-thumb {
  background: rgba(234, 179, 8, 0.2);
  border-radius: 10px;
}
</style>
