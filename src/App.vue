<script setup>
import { HelpCircle, Trophy, Play, Star, Sparkles, Coins } from 'lucide-vue-next'
import { ref, onMounted, computed } from 'vue'
import Auth from './components/Auth.vue'
import Admin from './components/Admin.vue'
import { API_ENDPOINTS } from './configs/api'

onMounted(() => {
  checkAuth()
})

const isSpinning = ref(false)
const rewards = ref(['4 vé', '5 vé', '6 vé', '4 vé', '5 vé', '6 vé']) 
const prizesList = ref([])
const currentRotation = ref(0)
const winMessage = ref('')
const userName = ref('')
const userToken = ref('')
const spinCount = ref(0)
const hasStarted = ref(false)
const spinsLeft = ref(0)
const leaderboard = ref([])
const isAdmin = ref(false)
const isAdminView = ref(false)

// Xử lý dữ liệu bảng xếp hạng: Lọc người chưa quay, trim khoảng trắng và sắp xếp mới nhất lên đầu
const sortedLeaderboard = computed(() => {
  return leaderboard.value
    .filter(item => item.spinResult && item.spinResult.trim() !== "" && item.spinResult !== "null")
    .map(item => ({
      ...item,
      spinResult: item.spinResult.trim()
    }))
    .sort((a, b) => new Date(b.lastSpinAt) - new Date(a.lastSpinAt))
})

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
      isAdmin.value = user.role === 'admin'
      if (isAdmin.value) {
        isAdminView.value = true
      }
      console.log("CheckAuth - User data:", user)
      
      // Logic chuẩn theo API của bạn:
      // - Nếu CHƯA CÓ kết quả (spinResult là null/rỗng) -> Được quay (spinsLeft = 1)
      // - Nếu ĐÀ CÓ kết quả (spinResult có giá trị) -> Hết lượt (spinsLeft = 0)
      if (!user.spinResult || user.spinResult === "null" || user.spinResult.trim() === "") {
        spinsLeft.value = 1
        winMessage.value = "" // Xóa thông báo nếu có
        console.log("-> Chưa quay (null/rỗng), cho phép quay.")
      } else {
        spinsLeft.value = 0
        winMessage.value = "Bạn đã hết lượt hái lộc hôm nay!"
        console.log("-> Đã quay rồi (có spinResult), khóa nút quay.")
      }
      
      spinCount.value = user.spinCount || 0
      hasStarted.value = true
      fetchLeaderboard(); // Lấy BXH sau khi xác thực thành công
      await fetchPrizes(); // Lấy danh sách giải thưởng cho vòng quay
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
      spinCount.value = user.spinCount || 0
      
      // Đồng bộ spinsLeft từ cache nếu lỗi mạng
      if (user.spinResult && user.spinResult !== "null") {
        spinsLeft.value = 0
      } else {
        spinsLeft.value = user.spinCount || 0
      }

      hasStarted.value = true
    }
  }
}

const onAuthSuccess = async (user) => {
  console.log("Đăng nhập/Đăng ký thành công, bắt đầu đồng bộ dữ liệu từ Server...");
  await checkAuth();
  await fetchLeaderboard();
}

const fetchLeaderboard = async () => {
  try {
    const token = localStorage.getItem('lucky_token');
    const response = await fetch(API_ENDPOINTS.GET_RESULTS, {
      headers: {
        'Authorization': `Bearer ${token}`
      }
    });
    const result = await response.json();
    if (result.success) {
      leaderboard.value = result.data;
    }
  } catch (err) {
    console.error('Lỗi lấy bảng xếp hạng:', err);
  }
}

const fetchPrizes = async () => {
  try {
    const token = localStorage.getItem('lucky_token');
    const response = await fetch(API_ENDPOINTS.PRIZES, {
      headers: {
        'Authorization': `Bearer ${token}`
      }
    })
    const result = await response.json()
    if (result.success && result.data) {
      // Chỉ lấy các giải thưởng CÒN XUẤT (currentWinners < maxWinners)
      prizesList.value = result.data.filter(p => p.currentWinners < p.maxWinners)
      
      if (prizesList.value.length > 0) {
        // Tạo danh sách rewards từ prize configs
        let newRewards = prizesList.value.map(p => `${p.ticketsPerWinner} vé`)
        
        // Đảm bảo số ô là số chẵn và >= 6 để màu sắc đan xen đẹp
        let finalRewards = [...newRewards]
        while (finalRewards.length < 6 || finalRewards.length % 2 !== 0) {
          finalRewards = [...finalRewards, ...newRewards]
          if (finalRewards.length > 12) break // Không quá nhiều ô
        }
        
        rewards.value = finalRewards
      } else {
        rewards.value = ['4 vé', '5 vé', '6 vé', '4 vé', '5 vé', '6 vé']
      }
    }
  } catch (err) {
    console.error('Lỗi lấy danh sách giải thưởng:', err)
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

// Giới hạn lượt quay (Dùng để hiển thị hoặc fallback) - Đã đưa lên top

const fetchResultFromServer = () => {
  // TODO: Trong thực tế, server nên trả về kết quả dựa trên kho giải thưởng thực tế
  // Hiện tại ta lấy ngẫu nhiên từ danh sách rewards đang hiển thị
  const randomIndex = Math.floor(Math.random() * rewards.value.length)
  return {
    index: randomIndex,
    reward: rewards.value[randomIndex]
  }
}

const spin = async () => {
  if (isSpinning.value || spinsLeft.value <= 0) return
  
  isSpinning.value = true
  winMessage.value = ''
  
  // 1. Lấy kết quả từ "Server" ngay lập tức
  const result = fetchResultFromServer()
  
  // 2. Tính toán góc quay để dừng đúng ô đó
  const segmentDegree = 360 / rewards.value.length
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
    // Chỉ cập nhật Bảng Vàng SAU KHI bánh xe dừng hẳn
    fetchLeaderboard();
  }, 3000)
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
    
    <!-- Admin Screen -->
    <Admin v-if="isAdminView" :token="userToken" :is-admin-role="isAdmin" @back="isAdminView = false" @logout="logout" />

    <!-- Auth Screen (Login/Register) -->
    <Auth v-else-if="!hasStarted" @auth-success="onAuthSuccess" />

    <!-- Main Game Content (Chỉ dành cho người chơi, không phải Admin) -->
    <template v-else-if="hasStarted && !isAdminView && !isAdmin">
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
            <span class="text-xs font-black text-yellow-500/60 uppercase">Người chơi:</span>
            <span class="text-sm font-black text-yellow-400 uppercase tracking-wider">{{ userName }}</span>
          </div>
          <button 
            v-if="isAdmin"
            @click="isAdminView = true"
            class="bg-yellow-500/10 hover:bg-yellow-500/20 px-6 py-2.5 rounded-lg text-sm font-black text-yellow-500 border border-yellow-500/30 transition-all active:scale-95 uppercase"
          >
            Quản Trị
          </button>
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
                BẢNG VÀNG HÁI LỘC
              </h3>
              <div class="max-h-[300px] overflow-y-auto pr-2 custom-scrollbar">
                <ul class="space-y-3 lg:space-y-4">
                  <li v-for="(item, index) in sortedLeaderboard" :key="index" 
                      class="flex items-center justify-between p-3 lg:p-4 bg-white/5 rounded-xl border border-white/5 hover:bg-yellow-500/10 hover:border-yellow-500/30 transition-all cursor-default animate-in fade-in slide-in-from-right duration-500"
                      :style="{ animationDelay: `${index * 100}ms` }"
                  >
                    <div class="flex items-center gap-3">
                      <div class="w-8 h-8 rounded-full bg-yellow-500/20 flex items-center justify-center text-yellow-500 font-bold text-sm border border-yellow-500/30">
                        {{ index + 1 }}
                      </div>
                      <span class="text-base font-bold text-yellow-100">{{ item.fullName }}</span>
                    </div>
                    <span class="text-[10px] lg:text-sm font-black text-yellow-500 bg-yellow-500/10 px-2 lg:px-3 py-1 rounded-full uppercase tracking-widest border border-yellow-500/20 shadow-sm shadow-yellow-500/10">
                      {{ item.spinResult || 'Đang chờ...' }}
                    </span>
                  </li>
                  <li v-if="sortedLeaderboard.length === 0" class="text-center py-10 text-yellow-500/40 italic text-sm">
                    Chưa có lượt hái lộc nào hôm nay...
                  </li>
                </ul>
              </div>
            </div>

            <div class="flex flex-col gap-4">


            <button 
              @click="spin"
              :disabled="isSpinning || spinsLeft <= 0"
              class="group relative w-full flex items-center justify-center gap-3 bg-gradient-to-r from-yellow-400 via-yellow-500 to-yellow-400 text-red-950 py-4 lg:py-5 rounded-2xl font-black text-xl lg:text-2xl transition-all hover:scale-[1.02] active:scale-95 disabled:opacity-50 disabled:grayscale disabled:cursor-not-allowed shadow-[0_0_40px_rgba(234,179,8,0.3)] hover:shadow-[0_0_60px_rgba(234,179,8,0.5)]"
            >
              <Play class="w-6 h-6 lg:w-7 lg:h-7 fill-red-900" />
              <span v-if="isSpinning">ĐANG HÁI LỘC...</span>
              <span v-else-if="spinsLeft > 0">QUAY NGAY</span>
              <span v-else>BẠN ĐÃ HẾT LƯỢT QUAY</span>
              <div class="absolute -inset-1 bg-yellow-400/30 rounded-2xl blur opacity-0 group-hover:opacity-100 transition-opacity"></div>
            </button>
            
            <p v-if="winMessage" class="text-center text-xl lg:text-2xl font-black text-yellow-400 animate-bounce py-2 lg:py-4 drop-shadow-lg">
              🎊 {{ winMessage }} 🎊
            </p>


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


    </template>
  </div>
</template>

<style>
/* Custom shadow for the wheel text */
.drop-shadow-gold {
  filter: drop-shadow(0 2px 2px rgba(234, 179, 8, 0.8));
}

.custom-scrollbar::-webkit-scrollbar {
  width: 4px;
}
.custom-scrollbar::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.05);
  border-radius: 10px;
}
.custom-scrollbar::-webkit-scrollbar-thumb {
  background: rgba(234, 179, 8, 0.3);
  border-radius: 10px;
}
.custom-scrollbar::-webkit-scrollbar-thumb:hover {
  background: rgba(234, 179, 8, 0.5);
}

@keyframes firework {
  0% { transform: translate(-50%, 100vh); width: 4px; opacity: 1; }
  50% { width: 4px; opacity: 1; }
  100% { transform: translate(-50%, 50vh); width: 400px; opacity: 0; }
}
</style>
