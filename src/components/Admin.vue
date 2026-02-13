<script setup>
import { ref, onMounted, computed } from 'vue'
import { Users, Search, RefreshCw, Trophy, Ban, CheckCircle, ArrowLeft, Download } from 'lucide-vue-next'
import { API_ENDPOINTS } from '../configs/api'

const props = defineProps(['token', 'isAdminRole'])
const emit = defineEmits(['back', 'logout'])

const users = ref([])
const loading = ref(false)
const searchQuery = ref('')

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
  <div class="min-h-screen bg-[#0f0f0f] text-white p-4 sm:p-8">
    <!-- Header -->
    <div class="max-w-7xl mx-auto space-y-8">
      <div class="flex flex-col sm:flex-row justify-between items-start sm:items-center gap-4">
        <div class="flex items-center gap-4">
          <button v-if="!props.isAdminRole" @click="emit('back')" class="p-2 hover:bg-white/10 rounded-full transition-colors">
            <ArrowLeft class="w-6 h-6 text-yellow-500" />
          </button>
          <div :class="{'pl-4 sm:pl-0': props.isAdminRole}">
            <h1 class="text-3xl font-black text-yellow-500 uppercase tracking-tighter">Hệ Thống Quản Trị</h1>
            <p class="text-white/50 text-sm italic">Chào mừng Quản trị viên, đây là toàn bộ dữ liệu người chơi</p>
          </div>
        </div>
        
        <div class="flex gap-3 w-full sm:w-auto">
          <button @click="exportCSV" class="flex-1 sm:flex-none flex items-center justify-center gap-2 px-4 py-2.5 bg-green-600/20 text-green-500 border border-green-500/30 rounded-xl font-bold hover:bg-green-600/30 transition-all">
            <Download class="w-4 h-4" /> Xuất Excel
          </button>
          <button @click="fetchAllUsers" :disabled="loading" class="flex-1 sm:flex-none flex items-center justify-center gap-2 px-4 py-2.5 bg-yellow-500/10 text-yellow-500 border border-yellow-500/30 rounded-xl font-bold hover:bg-yellow-500/20 transition-all disabled:opacity-50">
            <RefreshCw class="w-4 h-4" :class="{'animate-spin': loading}" />
          </button>
          <button @click="emit('logout')" class="flex-1 sm:flex-none flex items-center justify-center gap-2 px-4 py-2.5 bg-red-600/20 text-red-500 border border-red-500/30 rounded-xl font-bold hover:bg-red-600/30 transition-all">
            Đăng Xuất
          </button>
        </div>
      </div>

      <!-- Stats -->
      <div class="grid grid-cols-1 sm:grid-cols-3 gap-6">
        <div class="bg-red-950/20 border border-yellow-500/10 p-6 rounded-3xl backdrop-blur-sm">
          <div class="flex items-center gap-4 mb-2">
            <Users class="w-5 h-5 text-blue-400" />
            <span class="text-white/50 font-bold text-sm uppercase">Tổng người chơi</span>
          </div>
          <div class="text-3xl font-black">{{ users.length }}</div>
        </div>
        <div class="bg-red-950/20 border border-yellow-500/10 p-6 rounded-3xl backdrop-blur-sm">
          <div class="flex items-center gap-4 mb-2">
            <CheckCircle class="w-5 h-5 text-green-400" />
            <span class="text-white/50 font-bold text-sm uppercase">Đã tham gia quay</span>
          </div>
          <div class="text-3xl font-black text-green-500">{{ users.filter(u => u.spinResult).length }}</div>
        </div>
        <div class="bg-red-950/20 border border-yellow-500/10 p-6 rounded-3xl backdrop-blur-sm">
          <div class="flex items-center gap-4 mb-2">
            <Ban class="w-5 h-5 text-red-400" />
            <span class="text-white/50 font-bold text-sm uppercase">Chưa quay lộc</span>
          </div>
          <div class="text-3xl font-black text-red-500">{{ users.filter(u => !u.spinResult).length }}</div>
        </div>
      </div>

      <!-- Search & Table -->
      <div class="bg-red-950/10 border border-white/5 rounded-3xl overflow-hidden backdrop-blur-xl">
        <div class="p-6 border-b border-white/5 flex flex-col sm:flex-row gap-4 justify-between bg-white/5">
          <div class="relative flex-1">
            <Search class="absolute left-4 top-1/2 -translate-y-1/2 w-5 h-5 text-white/30" />
            <input 
              v-model="searchQuery"
              type="text" 
              placeholder="Tìm kiếm theo tên hoặc số điện thoại..." 
              class="w-full bg-[#0a0a0a] border border-white/10 rounded-2xl py-3 pl-12 pr-4 focus:border-yellow-500 focus:outline-none transition-all placeholder:text-white/20"
            />
          </div>
        </div>

        <div class="overflow-x-auto">
          <table class="w-full text-left">
            <thead>
              <tr class="text-yellow-500/50 text-xs font-black uppercase tracking-widest bg-white/5">
                <th class="px-6 py-4">Người chơi</th>
                <th class="px-6 py-4">Số điện thoại</th>
                <th class="px-6 py-4">Kết quả hái lộc</th>
                <th class="px-6 py-4">Trạng thái</th>
                <th class="px-6 py-4 text-right">Thời gian quay</th>
              </tr>
            </thead>
            <tbody class="divide-y divide-white/5">
              <tr v-for="user in filteredUsers" :key="user.id" class="hover:bg-white/5 transition-colors group">
                <td class="px-6 py-5">
                  <div class="flex items-center gap-3">
                    <div class="w-10 h-10 rounded-full bg-gradient-to-br from-yellow-500/20 to-yellow-600/10 border border-yellow-500/20 flex items-center justify-center text-yellow-500 font-black">
                      {{ user.fullName.charAt(0) }}
                    </div>
                    <span class="font-bold text-lg text-white group-hover:text-yellow-500 transition-colors">{{ user.fullName }}</span>
                  </div>
                </td>
                <td class="px-6 py-5 font-mono text-white/70">{{ user.phoneNumber }}</td>
                <td class="px-6 py-5">
                  <span v-if="user.spinResult" class="px-3 py-1.5 bg-yellow-500 text-black font-black rounded-lg text-xs uppercase shadow-lg shadow-yellow-500/10">
                    {{ user.spinResult }}
                  </span>
                  <span v-else class="text-white/20 italic text-sm">Chưa có kết quả</span>
                </td>
                <td class="px-6 py-5">
                  <div v-if="user.spinResult" class="flex items-center gap-2 text-red-500 bg-red-500/10 px-3 py-1 rounded-full w-fit border border-red-500/20">
                    <Ban class="w-3 h-3" />
                    <span class="text-[10px] font-black uppercase">Đã hết lượt</span>
                  </div>
                  <div v-else class="flex items-center gap-2 text-green-500 bg-green-500/10 px-3 py-1 rounded-full w-fit border border-green-500/20">
                    <CheckCircle class="w-3 h-3" />
                    <span class="text-[10px] font-black uppercase">Có thể quay</span>
                  </div>
                </td>
                <td class="px-6 py-5 text-right text-sm text-white/40 font-medium">
                  {{ formatDate(user.lastSpinAt) }}
                </td>
              </tr>
              <tr v-if="filteredUsers.length === 0">
                <td colspan="5" class="px-6 py-20 text-center text-white/20 italic">
                  Không tìm thấy người chơi nào khớp với yêu cầu...
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
