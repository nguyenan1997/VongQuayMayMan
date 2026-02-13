<script setup>
import { HelpCircle, Trophy, Play, Star, Sparkles, Coins } from 'lucide-vue-next'
import { ref, onMounted } from 'vue'
import Auth from './components/Auth.vue'
import { API_ENDPOINTS } from './configs/api'

onMounted(() => {
  // 1. Chặn chuột phải (Context Menu)
  document.addEventListener('contextmenu', (e) => {
    e.preventDefault()
  })

  // 2. Chặn các phím tắt phổ biến để mở DevTools
  document.addEventListener('keydown', (e) => {
    // F12
    if (e.key === 'F12') {
      e.preventDefault()
    }
    // Ctrl + Shift + I/C/J
    if (e.ctrlKey && e.shiftKey && (e.key === 'I' || e.key === 'C' || e.key === 'J')) {
      e.preventDefault()
    }
    // Ctrl + U (Xem source code)
    if (e.ctrlKey && e.key === 'u') {
      e.preventDefault()
    }
  })

  checkAuth()
})

const isSpinning = ref(false)
const rewards = ['4 vé', '5 vé', '6 vé', '4 vé', '5 vé', '6 vé', '4 vé', '5 vé', '6 vé']
const currentRotation = ref(0)
const winMessage = ref('')
const userName = ref('')
const userToken = ref('')
const spinCount = ref(0)
const hasStarted = ref(false)

const checkAuth = async () => {
  const token = localStorage.getItem('lucky_token')
  if (!token) {
    hasStarted.value = false
    return
  }

  try {
    const response = await fetch(API_ENDPOINTS.GET_ME, {
      headers: {
        'Authorization': `Bearer ${token}`
      }
    })
    const result = await response.json()

    if (result.success) {
      const user = result.data
      userName.value = user.fullName
      userToken.value = token
      spinCount.value = user.spinCount || 0
      hasStarted.value = true
      
      // Cập nhật lại localStorage với thông tin mới nhất
      localStorage.setItem('lucky_user', JSON.stringify(user))
      
      // Đồng bộ lượt quay từ server
      if (user.spinResult) {
        spinsLeft.value = 0
      } else {
        spinsLeft.value = MAX_SPINS
      }
    } else {
      // Token không hợp lệ hoặc hết hạn
      logout()
    }
  } catch (err) {
    console.error('Lỗi kiểm tra xác thực:', err)
    // Nếu lỗi mạng, vẫn cho dùng tạm dữ liệu cũ từ localStorage nếu có
    const userStr = localStorage.getItem('lucky_user')
    if (userStr) {
      const user = JSON.parse(userStr)
      userName.value = user.fullName
      userToken.value = token
      hasStarted.value = true
    }
  }
}

const onAuthSuccess = (user) => {
  userName.value = user.fullName
  userToken.value = localStorage.getItem('lucky_token')
  spinCount.value = user.spinCount || 0
  hasStarted.value = true
  
  if (user.spinResult) {
    spinsLeft.value = 0
  }
}

const logout = () => {
  localStorage.removeItem('lucky_token')
  localStorage.removeItem('lucky_user')
  userName.value = ''
  userToken.value = ''
  hasStarted.value = false
  window.location.reload()
}

// Anti-cheat: Giới hạn lượt quay
const MAX_SPINS = 1
const spinsLeft = ref(localStorage.getItem('lucky_spins_left') !== null ? parseInt(localStorage.getItem('lucky_spins_left')) : MAX_SPINS)
const lastSpinDate = ref(localStorage.getItem('last_spin_date') ?? '')

// Kiểm tra và reset lượt quay theo ngày mới
const today = new Date().toDateString()
if (lastSpinDate.value !== today) {
  spinsLeft.value = MAX_SPINS
  localStorage.setItem('lucky_spins_left', MAX_SPINS)
  localStorage.setItem('last_spin_date', today)
}

const fetchResultFromServer = () => {
  // Giả lập gọi API lên Server để lấy kết quả
  // Server sẽ quyết định kết quả dựa trên tỉ lệ % thật
  const randomIndex = Math.floor(Math.random() * rewards.length)
  return {
    index: randomIndex,
    reward: rewards[randomIndex]
  }
}

const spin = async () => {
  if (isSpinning.value || spinsLeft.value <= 0) return
  
  isSpinning.value = true
  winMessage.value = ''
  
  // 1. Lấy kết quả từ "Server" ngay lập tức
  const result = fetchResultFromServer()
  
  // 2. Tính toán góc quay để dừng đúng ô đó
  const segmentDegree = 360 / rewards.length
  const targetDegree = 360 - (result.index * segmentDegree) - (segmentDegree / 2)
  const totalRotation = currentRotation.value + (360 * 10) + (targetDegree - (currentRotation.value % 360))
  
  currentRotation.value = totalRotation

  // 3. Trừ lượt quay ngay khi bắt đầu (đề phòng F5)
  spinsLeft.value--
  
  // Call backend to save result
  try {
    const response = await fetch(API_ENDPOINTS.UPDATE_SPIN, {
      method: 'PUT',
      headers: { 
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${userToken.value}`
      },
      body: JSON.stringify({ result: result.reward })
    })
    
    const resultData = await response.json()
    if (!resultData.success) {
      throw new Error(resultData.message || 'Không thể lưu kết quả')
    }

    // Cập nhật lại thông tin user trong localStorage nếu cần
    const currentUser = JSON.parse(localStorage.getItem('lucky_user') || '{}')
    currentUser.spinResult = resultData.data.spinResult
    currentUser.lastSpinAt = resultData.data.lastSpinAt
    localStorage.setItem('lucky_user', JSON.stringify(currentUser))

  } catch (err) {
    console.error('Error saving spin result:', err)
    winMessage.value = 'Lỗi: ' + err.message
    isSpinning.value = false
    // Hoàn lại lượt quay nếu lỗi server
    spinsLeft.value++
    return
  }
  
  setTimeout(() => {
    isSpinning.value = false
    winMessage.value = `Chúc mừng ${userName.value}! Bạn đã hái lộc được ${result.reward}`
    showFireworks()
  }, 3000)
}


const resetForTesting = () => {
  spinsLeft.value = MAX_SPINS
  localStorage.setItem('lucky_spins_left', MAX_SPINS)
  winMessage.value = 'Đã khôi phục lượt quay (Chỉ dùng cho Testing)'
}
const resetSystem = () => {
  localStorage.removeItem('lucky_token')
  localStorage.removeItem('lucky_user')
  window.location.reload()
}

// Firework Logic
const showFireworks = () => {
  const canvas = document.getElementById('fireworks-canvas')
  const ctx = canvas.getContext('2d')
  canvas.width = window.innerWidth
  canvas.height = window.innerHeight

  let particles = []
  const colors = ['#ff0000', '#ffd700', '#ff4500', '#ffffff', '#ff6347', '#ffce00']

  class Particle {
    constructor(x, y, color) {
      this.x = x
      this.y = y
      this.color = color
      this.velocity = {
        x: (Math.random() - 0.5) * 12,
        y: (Math.random() - 0.5) * 12
      }
      this.alpha = 1
      this.friction = 0.96
      this.gravity = 0.15
    }

    draw() {
      ctx.save()
      ctx.globalAlpha = this.alpha
      ctx.beginPath()
      ctx.arc(this.x, this.y, 3, 0, Math.PI * 2)
      ctx.fillStyle = this.color
      ctx.shadowBlur = 10
      ctx.shadowColor = this.color
      ctx.fill()
      ctx.restore()
    }

    update() {
      this.velocity.x *= this.friction
      this.velocity.y *= this.friction
      this.velocity.y += this.gravity
      this.x += this.velocity.x
      this.y += this.velocity.y
      this.alpha -= 0.012
    }
  }

  const createFirework = (x, y) => {
    for (let i = 0; i < 80; i++) {
      particles.push(new Particle(x, y, colors[Math.floor(Math.random() * colors.length)]))
    }
  }

  const animate = () => {
    const requestID = requestAnimationFrame(animate)
    ctx.clearRect(0, 0, canvas.width, canvas.height)

    particles.forEach((particle, index) => {
      if (particle.alpha > 0) {
        particle.update()
        particle.draw()
      } else {
        setTimeout(() => {
          particles.splice(index, 1)
        }, 0)
      }
    })

    // Tự động dừng sau 6 giây để tiết kiệm tài nguyên
  }

  // Khởi chạy vòng lặp animation
  animate()

  // Bắn pháo hoa liên tục trong 3 giây
  let count = 0
  const interval = setInterval(() => {
    createFirework(
      Math.random() * canvas.width,
      Math.random() * canvas.height * 0.5
    )
    count++
    if (count > 12) clearInterval(interval)
  }, 400)

  // Dọn dẹp sau 8 giây
  setTimeout(() => {
    particles = []
    ctx.clearRect(0, 0, canvas.width, canvas.height)
  }, 8000)
}
</script>

<template>
  <div class="selection:bg-yellow-500/30">
    <canvas id="fireworks-canvas" class="fixed inset-0 pointer-events-none z-[150]"></canvas>
    <!-- Auth Screen (Login/Register) -->
    <Auth v-if="!hasStarted" @auth-success="onAuthSuccess" />

    <!-- Main Game Content -->
    <template v-if="hasStarted">
      <!-- Navigation -->
      <nav class="flex items-center justify-between px-4 sm:px-12 py-6 backdrop-blur-md bg-red-950/10 sticky top-0 z-50">
        <div class="flex items-center gap-3 group cursor-pointer">
          <div class="w-12 h-12 bg-gradient-to-br from-yellow-400 to-yellow-600 rounded-full flex items-center justify-center shadow-lg shadow-yellow-500/30 group-hover:scale-110 transition-transform border-2 border-red-800">
            <Coins class="w-7 h-7 text-red-900" />
          </div>
          <div>
            <span class="text-2xl font-black tracking-tighter block leading-none text-yellow-500 shadow-yellow-900 drop-shadow-sm uppercase">Tết May Mắn</span>
          </div>
        </div>
        
        <div class="flex items-center gap-8">
          <div class="hidden sm:flex items-center gap-2 px-4 py-2 bg-yellow-500/10 rounded-full border border-yellow-500/20">
            <span class="text-xs font-black text-yellow-500/60 uppercase">Lượt quay:</span>
            <span class="text-sm font-black text-yellow-400 uppercase tracking-wider">{{ spinCount }}</span>
          </div>
          <div class="hidden sm:flex items-center gap-2 px-4 py-2 bg-yellow-500/10 rounded-full border border-yellow-500/20">
            <span class="text-xs font-black text-yellow-500/60 uppercase">Người chơi:</span>
            <span class="text-sm font-black text-yellow-400 uppercase tracking-wider">{{ userName }}</span>
          </div>
          <button 
            @click="logout"
            class="bg-red-900/50 hover:bg-red-900 px-6 py-2.5 rounded-lg text-sm font-black text-yellow-500 border border-yellow-500/30 transition-all active:scale-95 uppercase"
          >
            Đăng Xuất
          </button>
        </div>
      </nav>

      <main class="w-full px-4 sm:px-12 pb-24 flex flex-col items-center pt-0 mt-0">
        <!-- Header Banner Style -->
        <div class="text-center space-y-2 mb-6 lg:mb-10 animate-in fade-in slide-in-from-top duration-1000 flex-shrink-0">
          <div class="inline-flex items-center gap-2 px-3 lg:px-4 py-1.5 rounded-full bg-yellow-400 text-red-900 text-[10px] lg:text-xs font-black uppercase tracking-[0.2em] border-2 border-yellow-200">
            <Sparkles class="w-3 h-3 lg:w-4 lg:h-4 fill-red-900" />
            Xuân Ất Tỵ 2025
            <Sparkles class="w-3 h-3 lg:w-4 lg:h-4 fill-red-900" />
          </div>
          
          <h1 class="text-4xl sm:text-6xl lg:text-8xl font-black tracking-tighter leading-none">
            <span class="text-white block">KHAI XUÂN</span>
            <span class="bg-clip-text text-transparent bg-gradient-to-b from-yellow-200 via-yellow-400 to-yellow-600 drop-shadow-2xl">NHẬN QUÀ KHỦNG</span>
          </h1>
          
          <p class="text-base lg:text-xl text-yellow-100/70 max-w-2xl mx-auto font-medium leading-relaxed italic px-4">
            "Chào mừng <span class="text-yellow-400 font-bold">{{ userName }}</span> hái lộc đầu năm"
          </p>
        </div>

        <div class="grid lg:grid-cols-2 gap-10 lg:gap-32 items-center w-full px-4 lg:px-20">
          <!-- Left Side: Prizes & Info (Order 2 on mobile, 1 on desktop) -->
          <div class="space-y-6 lg:space-y-8 order-2 lg:order-1 px-2">
            <div class="bg-red-900/40 p-6 lg:p-8 rounded-3xl border-2 border-yellow-500/20 backdrop-blur-sm relative overflow-hidden group">
              <div class="absolute -top-10 -right-10 w-32 h-32 bg-yellow-500/10 blur-3xl rounded-full"></div>
              <h3 class="text-xl lg:text-2xl font-black text-yellow-500 mb-4 lg:mb-6 flex items-center gap-3">
                <Trophy class="w-5 h-5 lg:w-6 lg:h-6" />
                DANH SÁCH GIẢI THƯỞNG
              </h3>
              <ul class="space-y-3 lg:space-y-4">
                <li v-for="item in ['6 vé - Đại Cát', '5 vé - Như Ý', '4 vé - Tài Lộc']" :key="item" 
                    class="flex items-center justify-between p-3 lg:p-4 bg-white/5 rounded-xl border border-white/5 hover:bg-yellow-500/10 hover:border-yellow-500/30 transition-all cursor-default">
                  <span class="text-base lg:text-lg font-bold">{{ item.split(' - ')[0] }}</span>
                  <span class="text-[10px] lg:text-sm font-black text-yellow-500 bg-yellow-500/10 px-2 lg:px-3 py-1 rounded-full uppercase tracking-widest border border-yellow-500/20">
                    {{ item.split(' - ')[1] }}
                  </span>
                </li>
              </ul>
            </div>

            <div class="flex flex-col gap-4">
            <div class="flex items-center justify-between px-5 py-3 bg-red-950/40 rounded-xl border border-yellow-500/20">
              <span class="text-sm font-bold text-yellow-100/50 uppercase tracking-wider">Lượt quay hôm nay:</span>
              <span class="text-xl lg:text-2xl font-black text-yellow-400">{{ spinsLeft }} / {{ MAX_SPINS }}</span>
            </div>

            <button 
              @click="spin"
              :disabled="isSpinning || spinsLeft <= 0"
              class="group relative w-full flex items-center justify-center gap-3 bg-gradient-to-r from-yellow-400 via-yellow-500 to-yellow-400 text-red-950 py-4 lg:py-5 rounded-2xl font-black text-xl lg:text-2xl transition-all hover:scale-[1.02] active:scale-95 disabled:opacity-50 disabled:grayscale disabled:cursor-not-allowed shadow-[0_0_40px_rgba(234,179,8,0.3)] hover:shadow-[0_0_60px_rgba(234,179,8,0.5)]"
            >
              <Play class="w-6 h-6 lg:w-7 lg:h-7 fill-red-900" />
              <span v-if="spinsLeft > 0">{{ isSpinning ? 'HÁI LỘC XUÂN...' : 'QUAY NGAY' }}</span>
              <span v-else>HẾT LƯỢT QUAY</span>
              <div class="absolute -inset-1 bg-yellow-400/30 rounded-2xl blur opacity-0 group-hover:opacity-100 transition-opacity"></div>
            </button>
            
            <p v-if="winMessage" class="text-center text-xl lg:text-2xl font-black text-yellow-400 animate-bounce py-2 lg:py-4 drop-shadow-lg">
              🎊 {{ winMessage }} 🎊
            </p>

            <button 
              @click="resetSystem"
              class="text-[10px] text-yellow-500/20 hover:text-yellow-500/50 transition-colors uppercase font-bold tracking-widest mt-4"
            >
              [ Reset Toàn Bộ Hệ Thống ]
            </button>
          </div>
          </div>

          <!-- Right Side: The Wheel (Order 1 on mobile, 2 on desktop) -->
          <div class="relative flex items-center justify-center order-1 lg:order-2 sm:scale-95 lg:scale-100 my-8 lg:my-0">
            <!-- Decorative Background Elements -->
            <div class="absolute w-[320px] lg:w-[500px] h-[320px] lg:h-[500px] bg-red-600/20 blur-[60px] lg:blur-[100px] rounded-full"></div>
            
            <!-- Lucky Wheel Frame -->
            <div class="relative p-2 sm:p-4 rounded-full bg-gradient-to-b from-yellow-300 via-yellow-600 to-yellow-800 shadow-[0_0_50px_rgba(0,0,0,0.5)] lg:shadow-[0_0_100px_rgba(0,0,0,0.5)]">
              <div class="absolute inset-0 bg-[radial-gradient(circle_at_center,_transparent_40%,_black_100%)] opacity-30 rounded-full"></div>
              
              <!-- The Actual Wheel -->
              <div 
                class="relative w-[320px] h-[320px] sm:w-[380px] sm:h-[380px] lg:w-[450px] lg:h-[450px] rounded-full border-4 border-yellow-100/50 transition-transform duration-[3000ms] ease-[cubic-bezier(0.25, 0.1, 0.25, 1)] overflow-hidden shadow-inner flex items-center justify-center"
                :style="{ transform: `rotate(${currentRotation}deg)` }"
              >
                <!-- Colored Segments -->
                <div v-for="(reward, i) in rewards" :key="i" 
                    class="absolute top-0 right-0 w-1/2 h-1/2 origin-bottom-left"
                    :style="{ 
                      transform: `rotate(${i * (360 / rewards.length)}deg) skewY(-${90 - (360 / rewards.length)}deg)`,
                      backgroundColor: i % 2 === 0 ? '#cc0000' : '#e60000'
                    }">
                </div>

                <!-- Reward Text in Segments -->
                <div v-for="(reward, i) in rewards" :key="'text-'+i" 
                    class="absolute w-full h-full flex items-start justify-center pt-8"
                    :style="{ transform: `rotate(${i * (360 / rewards.length) + (180 / rewards.length)}deg)` }">
                  <div class="flex flex-col items-center gap-1">
                    <Star class="w-4 h-4 text-yellow-400 fill-yellow-400" />
                    <span class="text-xl font-black text-yellow-400 drop-shadow-md whitespace-nowrap uppercase tracking-tighter">
                      {{ reward }}
                    </span>
                  </div>
                </div>

                <!-- Center Hub -->
                <div class="absolute w-32 h-32 bg-gradient-to-br from-yellow-200 via-yellow-500 to-yellow-700 rounded-full z-20 flex items-center justify-center border-8 border-red-900 shadow-2xl">
                  <div class="w-full h-full rounded-full border-4 border-yellow-300 opacity-50 absolute"></div>
                  <div class="text-3xl font-black text-red-950 tracking-tighter text-center leading-none">
                    TẾT<br/>2025
                  </div>
                </div>
              </div>

              <!-- Pointer/Indicator -->
              <div class="absolute -top-4 left-1/2 -translate-x-1/2 z-30 drop-shadow-2xl">
                <div class="w-12 h-12 bg-gradient-to-b from-yellow-200 to-yellow-500 scale-x-75 rotate-45 rounded-sm border-2 border-red-900 shadow-xl"></div>
              </div>

              <!-- Outer Frame Lights -->
              <div v-for="i in 24" :key="'light-'+i"
                  class="absolute w-2 h-2 rounded-full z-10 shadow-[0_0_5px_white]"
                  :class="i % 2 === 0 ? 'bg-yellow-200 animate-pulse' : 'bg-white'"
                  :style="{ transform: `rotate(${i * 15}deg) translateY(-210px)` }">
              </div>
            </div>
          </div>
        </div>
      </main>

      <!-- Bottom Decoration: Apricot Blossoms placeholder -->
      <div class="fixed bottom-0 left-0 p-8 flex gap-4 pointer-events-none opacity-40">
        <div v-for="i in 3" :key="i" class="w-10 h-10 bg-yellow-400 rounded-full animate-pulse" :style="{ animationDelay: `${i * 0.5}s` }"></div>
      </div>
      <div class="fixed bottom-0 right-0 p-8 flex gap-4 pointer-events-none opacity-40">
        <div v-for="i in 3" :key="i" class="w-10 h-10 bg-yellow-400 rounded-full animate-pulse" :style="{ animationDelay: `${i * 0.5}s` }"></div>
      </div>
    </template>
  </div>
</template>

<style>
/* Custom shadow for the wheel text */
.drop-shadow-gold {
  filter: drop-shadow(0 2px 2px rgba(234, 179, 8, 0.8));
}

@keyframes firework {
  0% { transform: translate(-50%, 100vh); width: 4px; opacity: 1; }
  50% { width: 4px; opacity: 1; }
  100% { transform: translate(-50%, 50vh); width: 400px; opacity: 0; }
}
</style>
