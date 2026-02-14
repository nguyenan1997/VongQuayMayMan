<script setup>
import { ref, onMounted, computed } from 'vue'
import { Users, Search, RefreshCw, Trophy, Ban, CheckCircle, ArrowLeft, Download, Moon, Sun, Ticket, Settings, Save } from 'lucide-vue-next'
import { API_ENDPOINTS } from '../configs/api'

const props = defineProps(['token', 'isAdminRole'])
const emit = defineEmits(['back', 'logout'])

const users = ref([])
const loading = ref(false)
const searchQuery = ref('')
const isDark = ref(localStorage.getItem('admin_theme') === 'dark')
const prizeConfigs = ref([])
const showPrizeModal = ref(false)
const editingPrize = ref(null)
const prizeForm = ref({
  maxWinners: 0,
  ticketsPerWinner: 1
})
const savingPrize = ref(false)

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

const fetchPrizeConfigs = async () => {
  try {
    const response = await fetch(API_ENDPOINTS.PRIZES, {
      headers: {
        'Authorization': `Bearer ${props.token}`
      }
    })
    const result = await response.json()
    if (result.success) {
      prizeConfigs.value = result.data || []
    }
  } catch (err) {
    console.error('Lỗi lấy cấu hình giải thưởng:', err)
  }
}

const openPrizeModal = (prize = null) => {
  if (prize) {
    editingPrize.value = prize.id
    prizeForm.value = {
      maxWinners: prize.maxWinners,
      ticketsPerWinner: prize.ticketsPerWinner
    }
  } else {
    editingPrize.value = null
    prizeForm.value = { maxWinners: 0, ticketsPerWinner: 1 }
  }
  showPrizeModal.value = true
}

const savePrizeConfig = async () => {
  if (prizeForm.value.maxWinners <= 0 || prizeForm.value.ticketsPerWinner <= 0) {
    alert('Vui lòng nhập giá trị hợp lệ!')
    return
  }

  savingPrize.value = true
  try {
    const isEdit = !!editingPrize.value
    const url = isEdit ? `${API_ENDPOINTS.PRIZES}/${editingPrize.value}` : API_ENDPOINTS.PRIZES
    const method = isEdit ? 'PUT' : 'POST'
    
    const response = await fetch(url, {
      method: method,
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${props.token}`
      },
      body: JSON.stringify(prizeForm.value)
    })
    const result = await response.json()
    if (result.success) {
      alert(isEdit ? 'Cập nhật thành công!' : 'Thêm mới thành công!')
      showPrizeModal.value = false
      fetchPrizeConfigs()
    } else {
      alert('Lỗi: ' + result.message)
    }
  } catch (err) {
    console.error('Lỗi lưu cấu hình:', err)
    alert('Đã có lỗi xảy ra.')
  } finally {
    savingPrize.value = false
  }
}

const deletePrize = async (id) => {
  if (!confirm('Bạn có chắc chắn muốn xóa cấu hình này?')) return
  try {
    const response = await fetch(`${API_ENDPOINTS.PRIZES}/${id}`, {
      method: 'DELETE',
      headers: {
        'Authorization': `Bearer ${props.token}`
      }
    })
    const result = await response.json()
    if (result.success) {
      fetchPrizeConfigs()
    } else {
      alert('Không thể xóa: ' + result.message)
    }
  } catch (err) {
    console.error('Lỗi xóa cấu hình:', err)
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

onMounted(() => {
  fetchAllUsers()
  fetchPrizeConfigs()
})

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
const refreshAllData = async () => {
  loading.value = true
  try {
    await Promise.all([
      fetchAllUsers(),
      fetchPrizeConfigs()
    ])
  } finally {
    loading.value = false
  }
}
</script>

<template>
  <div :class="[isDark ? 'bg-red-950 text-yellow-50' : 'bg-[#fffdfa] text-red-950']" class="min-h-screen p-4 sm:p-10 font-sans tracking-tight transition-colors duration-300">
    <div class="max-w-7xl mx-auto space-y-12">
      
      <!-- Header Section -->
      <div class="flex flex-col md:flex-row justify-between items-start md:items-center gap-8">
        <div>
          <h1 class="text-3xl font-bold tracking-tight flex items-center gap-3.5" :class="isDark ? 'text-yellow-500' : 'text-red-900'">
            <div class="w-11 h-11 bg-yellow-500 rounded-2xl flex items-center justify-center shadow-xl shadow-yellow-500/20">
              <Trophy class="w-5 h-5 text-red-950" />
            </div>
            Quản trị hoạt động
          </h1>
          <p class="mt-1.5 font-medium text-[15px] opacity-60">Theo dõi thông tin người chơi hái lộc Xuân Ất Tỵ</p>
        </div>

        <div class="flex flex-wrap items-center gap-3.5 w-full md:w-auto">
          <!-- Theme Toggle -->
          <button @click="toggleTheme" class="p-3 rounded-2xl border transition-all active:scale-95 duration-200" 
            :class="isDark ? 'bg-red-900 border-yellow-500/20 text-yellow-500 hover:bg-red-800' : 'bg-white border-red-200 text-red-500 shadow-sm hover:bg-red-50'">
            <Sun v-if="isDark" class="w-[18px] h-[18px]" />
            <Moon v-else class="w-[18px] h-[18px]" />
          </button>

          <button @click="exportCSV" class="flex-1 md:flex-none flex items-center justify-center gap-2.5 px-6 py-3 rounded-2xl font-semibold text-[14px] transition-all active:scale-95 border duration-200"
            :class="isDark ? 'bg-red-900 border-yellow-500/20 text-yellow-100 hover:bg-red-800' : 'bg-white border-red-200 text-red-700 hover:bg-red-50'">
            <Download class="w-4 h-4" /> Xuất dữ liệu
          </button>
          
          <button @click="refreshAllData" :disabled="loading" class="flex-1 md:flex-none flex items-center justify-center gap-2.5 px-6 py-3 bg-yellow-500 text-red-950 rounded-2xl font-semibold text-[14px] hover:bg-yellow-400 transition-all shadow-xl active:scale-95 disabled:opacity-50 shadow-yellow-500/15 duration-200">
            <RefreshCw class="w-4 h-4" :class="{'animate-spin': loading}" /> Làm mới
          </button>
          
          <button @click="emit('logout')" class="flex-1 md:flex-none flex items-center justify-center gap-2.5 px-6 py-3 rounded-2xl font-semibold text-[14px] transition-all active:scale-95 border duration-200"
            :class="isDark ? 'bg-red-500/5 border-red-500/20 text-red-400 hover:bg-red-500/10' : 'bg-red-50 border-red-100 text-red-600 hover:bg-red-100'">
            Đăng xuất
          </button>
        </div>
      </div>

      <!-- Prize Pool Configuration -->
      <div class="rounded-[2rem] border overflow-hidden p-8 transition-all duration-300"
        :class="isDark ? 'bg-red-900/60 border-yellow-500/20' : 'bg-red-50/50 border-red-100 shadow-sm'">
        <div class="flex items-center justify-between mb-8">
          <div class="flex items-center gap-3">
            <div class="w-10 h-10 rounded-xl bg-yellow-500/10 flex items-center justify-center">
              <Settings class="w-5 h-5 text-yellow-600" />
            </div>
            <div>
              <h2 class="text-xl font-bold tracking-tight" :class="isDark ? 'text-white' : 'text-red-900'">Quản lý kho giải thưởng</h2>
              <p class="text-sm opacity-60">Thiết lập nhiều đợt hoặc loại giải thưởng</p>
            </div>
          </div>
          <button @click="openPrizeModal()" 
            class="flex items-center gap-2 px-5 py-2.5 bg-yellow-500 text-red-950 rounded-xl font-bold text-sm hover:bg-yellow-400 transition-all active:scale-95">
            <Trophy class="w-4 h-4" /> Thêm giải mới
          </button>
        </div>

        <div class="overflow-x-auto">
          <table class="w-full text-left">
            <thead>
              <tr class="text-[10px] font-bold uppercase tracking-widest opacity-40 border-b" :class="isDark ? 'border-yellow-500/20' : 'border-red-200'">
                <th class="pb-4 px-2">Cấu hình</th>
                <th class="pb-4 px-2">Số người tối đa</th>
                <th class="pb-4 px-2">Vé mỗi người</th>
                <th class="pb-4 px-2">Đã trúng</th>
                <th class="pb-4 px-2 text-right">Thao tác</th>
              </tr>
            </thead>
            <tbody class="divide-y" :class="isDark ? 'divide-yellow-500/10' : 'divide-red-100'">
              <tr v-for="(prize, index) in prizeConfigs" :key="prize.id" class="group">
                <td class="py-4 px-2">
                  <span class="font-bold whitespace-nowrap" :class="isDark ? 'text-yellow-100' : 'text-red-900'">{{ prize.ticketsPerWinner }} vé</span>
                </td>
                <td class="py-4 px-2 font-mono">{{ prize.maxWinners }}</td>
                <td class="py-4 px-2 font-mono">{{ prize.ticketsPerWinner }}</td>
                <td class="py-4 px-2">
                  <span class="px-2 py-1 rounded-md text-[11px] font-bold"
                    :class="prize.currentWinners >= prize.maxWinners ? 'bg-red-500/10 text-red-500' : 'bg-emerald-500/10 text-emerald-500'">
                    {{ prize.currentWinners }} / {{ prize.maxWinners }}
                  </span>
                </td>
                <td class="py-4 px-2 text-right space-x-2">
                  <button @click="openPrizeModal(prize)" class="p-2 rounded-lg hover:bg-yellow-500/10 text-yellow-500 transition-colors">
                    <Settings class="w-4 h-4" />
                  </button>
                  <button @click="deletePrize(prize.id)" class="p-2 rounded-lg hover:bg-red-500/10 text-red-500 transition-colors">
                    <Ban class="w-4 h-4" />
                  </button>
                </td>
              </tr>
              <tr v-if="prizeConfigs.length === 0">
                <td colspan="4" class="py-12 text-center opacity-30 italic text-sm">Chưa có cấu hình giải thưởng nào</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Add/Edit Prize Modal -->
      <div v-if="showPrizeModal" class="fixed inset-0 z-[100] flex items-center justify-center p-4">
        <div @click="showPrizeModal = false" class="absolute inset-0 bg-red-950/60 backdrop-blur-sm"></div>
        <div class="relative w-full max-w-md rounded-[2rem] border p-8 shadow-2xl transition-all"
          :class="isDark ? 'bg-red-900 border-red-800' : 'bg-white border-red-100'">
          
          <h3 class="text-xl font-bold mb-6 flex items-center gap-3">
            <Trophy class="w-6 h-6 text-yellow-500" />
            {{ editingPrize ? 'Cập nhật giải thưởng' : 'Thêm giải thưởng mới' }}
          </h3>

          <div class="space-y-6">
            <div class="grid grid-cols-2 gap-4">
              <div class="space-y-2">
                <label class="text-xs font-bold uppercase tracking-widest opacity-50 px-1">Số người tối đa</label>
                <input v-model.number="prizeForm.maxWinners" type="number" 
                  class="w-full border rounded-2xl py-3.5 px-5 transition-all outline-none font-mono font-bold"
                  :class="isDark ? 'bg-red-950 border-red-800 focus:border-yellow-500 text-white' : 'bg-red-50 border-red-200 focus:border-red-600'" />
              </div>
              <div class="space-y-2">
                <label class="text-xs font-bold uppercase tracking-widest opacity-50 px-1">Vé/Người</label>
                <input v-model.number="prizeForm.ticketsPerWinner" type="number"
                  class="w-full border rounded-2xl py-3.5 px-5 transition-all outline-none font-mono font-bold"
                  :class="isDark ? 'bg-red-950 border-red-800 focus:border-yellow-500 text-white' : 'bg-red-50 border-red-200 focus:border-red-600'" />
              </div>
            </div>
          </div>

          <div class="mt-10 flex gap-3">
            <button @click="showPrizeModal = false" class="flex-1 py-3.5 rounded-2xl font-bold opacity-60 hover:opacity-100 transition-opacity">Hủy</button>
            <button @click="savePrizeConfig" :disabled="savingPrize"
              class="flex-1 py-3.5 bg-yellow-500 text-red-950 rounded-2xl font-bold shadow-xl shadow-yellow-500/20 hover:bg-yellow-400 active:scale-95 disabled:opacity-50 transition-all flex items-center justify-center gap-2">
              <RefreshCw v-if="savingPrize" class="w-4 h-4 animate-spin" />
              {{ editingPrize ? 'Cập nhật' : 'Xác nhận' }}
            </button>
          </div>
        </div>
      </div>

      <!-- Stats Grid -->
      <div class="grid grid-cols-1 sm:grid-cols-3 gap-8">
        <div v-for="(stat, idx) in [
          { label: 'Tổng người chơi', value: users.length, icon: Users, color: 'text-yellow-600', bg: 'bg-yellow-500' },
          { label: 'Đã hái lộc', value: users.filter(u => u.spinResult).length, icon: CheckCircle, color: 'text-emerald-500', bg: 'bg-emerald-500' },
          { label: 'Chưa tham gia', value: users.filter(u => !u.spinResult).length, icon: Ban, color: 'text-red-500', bg: 'bg-red-500' }
        ]" :key="idx" 
        class="p-8 rounded-[32px] border transition-all duration-300 relative overflow-hidden group"
        :class="isDark ? 'bg-red-900/40 border-red-800 hover:border-red-700' : 'bg-red-50 border-red-100 hover:bg-white hover:shadow-2xl hover:shadow-red-200/50 hover:border-transparent'">
          <div class="flex items-center gap-5 mb-5 relative z-10">
            <div class="w-12 h-12 rounded-2xl flex items-center justify-center transition-all duration-300 shadow-sm"
              :class="isDark ? `bg-red-800 ${stat.color}` : `${stat.bg}/10 ${stat.color}`">
              <component :is="stat.icon" class="w-5 h-5" />
            </div>
            <span class="font-semibold text-[13px] uppercase tracking-[0.05em] opacity-50">{{ stat.label }}</span>
          </div>
          <div class="text-4xl font-bold tracking-tight relative z-10" :class="isDark ? 'text-white' : 'text-red-900'">{{ stat.value }}</div>
        </div>
      </div>

      <!-- Main Content Block -->
      <div class="rounded-[2rem] shadow-sm border overflow-hidden transition-all"
        :class="isDark ? 'bg-red-900 border-red-800' : 'bg-white border-red-100'">
        
        <!-- Search Bar -->
        <div class="p-6 border-b flex flex-col sm:flex-row gap-4 justify-between transition-colors"
          :class="isDark ? 'bg-red-900/50 border-red-800' : 'bg-white border-red-50'">
          <div class="relative flex-1 group">
            <Search class="absolute left-4 top-1/2 -translate-y-1/2 w-5 h-5 transition-colors" 
              :class="isDark ? 'text-red-600 group-focus-within:text-yellow-400' : 'text-red-400 group-focus-within:text-red-600'"/>
            <input 
              v-model="searchQuery"
              type="text" 
              placeholder="Tìm theo tên hoặc số điện thoại..." 
              class="w-full border rounded-2xl py-3.5 pl-12 pr-6 focus:ring-4 transition-all placeholder:text-red-800/30 font-medium outline-none"
              :class="isDark 
                ? 'bg-red-950 border-red-800 focus:border-yellow-500 focus:ring-yellow-500/10 text-white' 
                : 'bg-red-50 border-red-100 focus:bg-white focus:border-red-600 focus:ring-red-50'"
            />
          </div>
        </div>

        <!-- Table View -->
        <div class="overflow-x-auto">
          <table class="w-full text-left">
            <thead>
              <tr class="text-xs font-bold uppercase tracking-widest border-b"
                :class="isDark ? 'text-yellow-500/60 bg-red-950/30 border-red-800' : 'text-red-400 bg-red-50/50 border-red-50'">
                <th class="px-8 py-5">Người chơi</th>
                <th class="px-8 py-5">Số điện thoại</th>
                <th class="px-8 py-5">Kết quả</th>
                <th class="px-8 py-5 text-center">Trạng thái</th>
                <th class="px-8 py-5 text-right">Ngày giờ quay</th>
              </tr>
            </thead>
            <tbody class="divide-y" :class="isDark ? 'divide-red-800' : 'divide-red-50'">
              <tr v-for="user in filteredUsers" :key="user.id" class="transition-all group"
                :class="isDark ? 'hover:bg-red-800/50' : 'hover:bg-red-50/80'">
                <td class="px-8 py-5">
                  <div class="flex items-center gap-3">
                    <div class="w-10 h-10 rounded-full flex items-center justify-center font-bold border"
                      :class="isDark ? 'bg-red-800 text-yellow-400 border-red-700' : 'bg-red-100 text-red-600 border-red-100'">
                      {{ user.fullName.charAt(0) }}
                    </div>
                    <span class="font-bold whitespace-nowrap" :class="isDark ? 'text-yellow-100' : 'text-red-900'">{{ user.fullName }}</span>
                  </div>
                </td>
                <td class="px-8 py-5 font-medium" :class="isDark ? 'text-red-300' : 'text-red-700/60'">{{ user.phoneNumber }}</td>
                <td class="px-8 py-5">
                  <span v-if="user.spinResult" class="px-3 py-1.5 font-bold rounded-lg text-xs uppercase border shadow-sm"
                    :class="isDark ? 'bg-yellow-500/10 text-yellow-500 border-yellow-500/20' : 'bg-red-900 text-white border-red-800 shadow-red-100/50'">
                    {{ user.spinResult }}
                  </span>
                  <span v-else class="italic text-sm" :class="isDark ? 'text-red-800' : 'text-red-200'">Chưa có lộc</span>
                </td>
                <td class="px-8 py-5">
                  <div class="flex justify-center">
                    <span v-if="user.spinResult" class="flex items-center gap-1.5 font-bold text-[10px] uppercase"
                      :class="isDark ? 'text-red-700' : 'text-red-200'">
                      <div class="w-1.5 h-1.5 rounded-full" :class="isDark ? 'bg-red-700' : 'bg-red-200'"></div>
                      Hết lượt
                    </span>
                    <span v-else class="flex items-center gap-1.5 text-emerald-500 font-bold text-[10px] uppercase">
                      <div class="w-1.5 h-1.5 rounded-full bg-emerald-500 animate-pulse"></div>
                      Khả dụng
                    </span>
                  </div>
                </td>
                <td class="px-8 py-5 text-right text-sm font-medium" :class="isDark ? 'text-red-400' : 'text-red-400'">
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
