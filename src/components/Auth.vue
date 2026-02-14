<script setup>
import { ref } from 'vue'
import { User, Phone, Lock, ArrowRight, UserPlus, LogIn } from 'lucide-vue-next'
import { API_ENDPOINTS } from '../configs/api'
import logo from '../assets/logo.png'

const emit = defineEmits(['auth-success'])

const isLogin = ref(true)
const loading = ref(false)
const error = ref('')

const loginData = ref({
  phoneNumber: '',
  password: ''
})

const registerData = ref({
  phoneNumber: '',
  fullName: '',
  password: ''
})

const toggleAuth = () => {
  isLogin.value = !isLogin.value
  error.value = ''
}

const handleLogin = async () => {
  loading.value = true
  error.value = ''
  try {
    const response = await fetch(API_ENDPOINTS.LOGIN, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(loginData.value)
    })
    const result = await response.json()
    
    // Kiểm tra success tương tự như API đăng ký
    if (!result.success) throw new Error(result.message || 'Đăng nhập thất bại')
    
    const userData = result.data
    localStorage.setItem('lucky_token', userData.token)
    localStorage.setItem('lucky_user', JSON.stringify(userData))
    emit('auth-success', userData)
  } catch (err) {
    error.value = err.message
  } finally {
    loading.value = false
  }
}

const handleRegister = async () => {
  loading.value = true
  error.value = ''
  try {
    const response = await fetch(API_ENDPOINTS.REGISTER, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(registerData.value)
    })
    const result = await response.json()
    
    // Kiểm tra success dựa trên response của bạn
    if (!result.success) throw new Error(result.message || 'Đăng ký thất bại')
    
    const userData = result.data
    // Lưu thông tin vào localStorage để tự động đăng nhập
    localStorage.setItem('lucky_token', userData.token)
    localStorage.setItem('lucky_user', JSON.stringify(userData))
    
    alert('Đăng ký thành công! Chào mừng bạn.')
    emit('auth-success', userData)
  } catch (err) {
    error.value = err.message
  } finally {
    loading.value = false
  }
}
</script>

<template>
  <div class="fixed inset-0 z-[100] flex items-center justify-center p-6 bg-red-950/95 backdrop-blur-xl transition-all duration-500">
    <div class="w-full max-w-md space-y-8 animate-in fade-in zoom-in duration-500">
      <!-- Logo Section -->
      <div class="text-center space-y-4">
        <div class="flex justify-center">
          <div class="w-20 h-20 bg-white rounded-full flex items-center justify-center shadow-2xl shadow-yellow-500/50 border-4 border-red-800 animate-bounce overflow-hidden">
            <img :src="logo" alt="Logo" class="w-full h-full object-cover rounded-full" />
          </div>
        </div>
        <div class="space-y-1">
          <h2 class="text-3xl font-black text-yellow-500 uppercase tracking-tighter">
            {{ isLogin ? 'Đăng Nhập' : 'Đăng Ký' }}
          </h2>
          <p class="text-yellow-100/60 font-medium italic text-sm">
            {{ isLogin ? 'Chào mừng bạn quay trở lại hái lộc' : 'Tham gia cùng chúng tôi để nhận quà khủng' }}
          </p>
        </div>
      </div>

      <!-- Form Section -->
      <div class="bg-red-900/20 backdrop-blur-md rounded-3xl p-8 border border-yellow-500/20 shadow-2xl">
        <form @submit.prevent="isLogin ? handleLogin() : handleRegister()" class="space-y-5">
          
          <!-- Full Name (Only for Register) -->
          <div v-if="!isLogin" class="space-y-2">
            <label class="text-xs font-black text-yellow-500 uppercase tracking-widest px-1">Họ và Tên</label>
            <div class="relative group">
              <User class="absolute left-4 top-1/2 -translate-y-1/2 w-5 h-5 text-yellow-500/50 group-focus-within:text-yellow-500 transition-colors" />
              <input 
                v-model="registerData.fullName"
                type="text" 
                required
                placeholder="Nguyễn Văn A"
                class="w-full bg-red-950/40 border border-yellow-500/20 rounded-2xl pl-12 pr-6 py-4 text-yellow-100 placeholder:text-yellow-500/20 focus:outline-none focus:border-yellow-500 transition-all"
              />
            </div>
          </div>

          <!-- Phone Number -->
          <div class="space-y-2">
            <label class="text-xs font-black text-yellow-500 uppercase tracking-widest px-1">Số điện thoại</label>
            <div class="relative group">
              <Phone class="absolute left-4 top-1/2 -translate-y-1/2 w-5 h-5 text-yellow-500/50 group-focus-within:text-yellow-500 transition-colors" />
              <input 
                v-if="isLogin"
                v-model="loginData.phoneNumber"
                type="tel" 
                required
                placeholder="0912345678"
                class="w-full bg-red-950/40 border border-yellow-500/20 rounded-2xl pl-12 pr-6 py-4 text-yellow-100 placeholder:text-yellow-500/20 focus:outline-none focus:border-yellow-500 transition-all font-mono"
              />
              <input 
                v-else
                v-model="registerData.phoneNumber"
                type="tel" 
                required
                placeholder="0912345678"
                class="w-full bg-red-950/40 border border-yellow-500/20 rounded-2xl pl-12 pr-6 py-4 text-yellow-100 placeholder:text-yellow-500/20 focus:outline-none focus:border-yellow-500 transition-all font-mono"
              />
            </div>
          </div>

          <!-- Password -->
          <div class="space-y-2">
            <label class="text-xs font-black text-yellow-500 uppercase tracking-widest px-1">Mật khẩu</label>
            <div class="relative group">
              <Lock class="absolute left-4 top-1/2 -translate-y-1/2 w-5 h-5 text-yellow-500/50 group-focus-within:text-yellow-500 transition-colors" />
              <input 
                v-if="isLogin"
                v-model="loginData.password"
                type="password" 
                required
                placeholder="••••••••"
                class="w-full bg-red-950/40 border border-yellow-500/20 rounded-2xl pl-12 pr-6 py-4 text-yellow-100 placeholder:text-yellow-500/20 focus:outline-none focus:border-yellow-500 transition-all"
              />
              <input 
                v-else
                v-model="registerData.password"
                type="password" 
                required
                placeholder="••••••••"
                class="w-full bg-red-950/40 border border-yellow-500/20 rounded-2xl pl-12 pr-6 py-4 text-yellow-100 placeholder:text-yellow-500/20 focus:outline-none focus:border-yellow-500 transition-all"
              />
            </div>
          </div>

          <!-- Error Message -->
          <p v-if="error" class="text-red-400 text-center text-sm font-bold animate-pulse">{{ error }}</p>

          <!-- Submit Button -->
          <button 
            type="submit"
            :disabled="loading"
            class="w-full bg-gradient-to-r from-yellow-500 to-yellow-600 hover:from-yellow-400 hover:to-yellow-500 text-red-950 py-4 rounded-2xl font-black text-xl uppercase tracking-widest transition-all hover:scale-[1.02] active:scale-95 shadow-lg shadow-yellow-500/20 flex items-center justify-center gap-3"
          >
            <template v-if="loading">
              <div class="w-6 h-6 border-2 border-red-950 border-t-transparent rounded-full animate-spin"></div>
            </template>
            <template v-else>
              <LogIn v-if="isLogin" class="w-6 h-6" />
              <UserPlus v-else class="w-6 h-6" />
              {{ isLogin ? 'Vào Game' : 'Đăng Ký Ngay' }}
            </template>
          </button>
        </form>

        <!-- Toggle Auth Mode -->
        <div class="mt-8 text-center">
          <p class="text-yellow-100/40 text-sm font-medium">
            {{ isLogin ? 'Chưa có tài khoản?' : 'Đã có tài khoản?' }}
            <button 
              @click="toggleAuth"
              class="text-yellow-500 font-black uppercase tracking-wider hover:text-yellow-400 transition-colors ml-2 inline-flex items-center gap-1"
            >
              {{ isLogin ? 'Đăng ký ngay' : 'Đăng nhập' }}
              <ArrowRight class="w-3 h-3" />
            </button>
          </p>
        </div>
      </div>
    </div>
  </div>
</template>
