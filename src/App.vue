<template>
  <div class="app">
    <header class="header">
      <h1>电子烧香 - 环保祈福平台</h1>
      <p>愿神明护佑，心愿得偿</p>
    </header>

    <main class="main-content">
      <div class="incense-container">
        <div class="incense-burner">
          <div class="burner-base"></div>
          <div
            v-for="(incense, index) in burningIncenses"
            :key="index"
            class="incense-stick"
          >
            <div
              class="incense-body"
              :style="{
                height: `${250 * (incenseStates[index] / 100)}px`,
                background: `linear-gradient(to bottom, #deb887, #a0522d ${
                  100 - incenseStates[index]
                }%)`,
              }"
            ></div>
            <div
              class="incense-flame"
              :class="{ extinguished: incenseStates[index] <= 0 }"
            ></div>
            <div
              class="incense-smoke"
              :class="{ extinguished: incenseStates[index] <= 0 }"
            ></div>
          </div>
          <!-- 飘动的心愿 -->
          <div
            v-for="(wish, index) in floatingWishes"
            :key="wish.id"
            :class="['floating-wish', wish.position]"
            :style="{ animationDelay: wish.delay + 'ms' }"
          >
            {{ wish.text }}
          </div>
        </div>

        <div class="control-panel">
          <div class="incense-selector">
            <label for="incense-count">选择香火数量:</label>
            <select
              id="incense-count"
              v-model="incenseCount"
              @change="updateIncensePreview"
            >
              <option value="1">1支</option>
              <option value="3">3支</option>
              <option value="6">6支</option>
              <option value="9">9支</option>
            </select>
          </div>

          <div class="time-selector">
            <label for="burn-time">选择燃烧时间 (分钟):</label>
            <div class="time-input-group">
              <select
                id="burn-time"
                :value="isCustomTime ? 'custom' : burnTimeMinutes"
                @change="handleTimeSelectChange($event.target.value)"
                class="time-select"
              >
                <option value="1">1分钟</option>
                <option value="5">5分钟</option>
                <option value="10">10分钟</option>
                <option value="15">15分钟</option>
                <option value="30">30分钟</option>
                <option value="60">60分钟</option>
                <option value="custom">自定义</option>
              </select>
              <input
                v-if="isCustomTime"
                type="number"
                v-model.number="burnTimeMinutes"
                @input="updateCustomTime"
                min="1"
                max="60"
                step="0.5"
                class="time-input"
                placeholder="自定义时间"
              />
            </div>
          </div>

          <button class="burn-btn" @click="burnIncense" :disabled="isBurning">
            {{ isBurning ? "上香中..." : "开始上香" }}
          </button>
        </div>
      </div>

      <div v-if="burningIncenses.length > 0" class="prayer-section">
        <h3>许下心愿</h3>
        <textarea
          v-model="prayer"
          placeholder="在这里写下您的心愿..."
        ></textarea>
        <button class="pray-btn" @click="submitPrayer">诚心祈愿</button>
      </div>
    </main>

    <footer class="footer">
      <p>环保祈福，心诚则灵</p>
    </footer>
  </div>
</template>

<script>
export default {
  name: "App",
  data() {
    return {
      incenseCount: 3,
      burningIncenses: [],
      isBurning: false,
      prayer: "身体健康；万事如意；笑口常开；幸福快乐",
      floatingWishes: [],
      allWishes: [], // 存储所有心愿
      wishInterval: null, // 心愿飘动的定时器
      currentWishIndex: 0, // 当前显示的心愿索引
      // 时间选择相关变量
      burnTimeMinutes: 5, // 默认燃烧时间（分钟）
      burnTimeSeconds: 300, // 默认燃烧时间（秒）
      burnTimers: [], // 每个香的燃烧定时器
      incenseStates: [], // 每个香的状态（剩余时间百分比）
      isCustomTime: false, // 是否为自定义时间模式
      customTimeValue: 5, // 自定义时间值（分钟）
    };
  },
  methods: {
    updateIncensePreview() {
      // 如果当前没有在上香，则更新预览
      if (!this.isBurning) {
        this.burningIncenses = Array(this.incenseCount).fill(null);
        // 同时重置每个香的状态
        this.incenseStates = Array(this.incenseCount).fill(100);
      }
    },

    // 处理时间选择变化
    handleTimeSelectChange(value) {
      // 检查是否选择了自定义选项
      if (value === 'custom') {
        this.isCustomTime = true;
        // 使用之前保存的自定义时间值
        this.burnTimeMinutes = this.customTimeValue;
      } else {
        this.isCustomTime = false;
        // 保存当前的选择值
        this.burnTimeMinutes = parseInt(value);
      }
      
      // 转换为秒
      this.burnTimeSeconds = this.burnTimeMinutes * 60;
    },
    
    // 更新自定义时间
    updateCustomTime() {
      // 确保时间值在有效范围内
      if (this.burnTimeMinutes < 0) this.burnTimeMinutes = 0;
      if (this.burnTimeMinutes > 60) this.burnTimeMinutes = 60;
      
      // 保存自定义时间值
      this.customTimeValue = this.burnTimeMinutes;
      
      // 转换为秒
      this.burnTimeSeconds = this.burnTimeMinutes * 60;
    },
    burnIncense() {
      if (this.isBurning) return;

      this.isBurning = true;
      this.burningIncenses = [];
      this.incenseStates = Array(this.incenseCount).fill(100);

      // 清除之前的所有定时器
      this.clearAllBurnTimers();

      // 逐个点燃香
      let count = 0;
      const interval = setInterval(() => {
        this.burningIncenses.push(null);

        // 为每支香设置燃烧定时器
        this.startIncenseTimer(count);

        count++;

        if (count >= this.incenseCount) {
          clearInterval(interval);
          this.isBurning = false;
        }
      }, 300);
    },

    // 为单支香设置燃烧定时器
    startIncenseTimer(index) {
      const totalSeconds = this.burnTimeSeconds;
      let remainingSeconds = totalSeconds;

      // 每秒更新一次香的状态
      const timer = setInterval(() => {
        remainingSeconds--;

        // 计算剩余时间百分比
        const remainingPercentage = (remainingSeconds / totalSeconds) * 100;

        // 更新香的状态
        this.incenseStates[index] = remainingPercentage;

        // 当香烧完时
        if (remainingSeconds <= 0) {
          clearInterval(timer);
          // 将香的状态设置为已烧完
          this.incenseStates[index] = 0;
        }
      }, 1000);

      // 保存定时器引用，以便后续清除
      this.burnTimers.push(timer);
    },

    // 清除所有燃烧定时器
    clearAllBurnTimers() {
      this.burnTimers.forEach((timer) => clearInterval(timer));
      this.burnTimers = [];
    },
    submitPrayer() {
      if (!this.prayer.trim()) {
        alert("请先写下您的心愿");
        return;
      }

      // 按逗号分隔心愿文本为独立条目
      const wishes = this.prayer
        .split("；")
        .map((wish) => wish.trim())
        .filter((wish) => wish);

      if (wishes.length === 0) {
        alert("请输入有效的心愿");
        return;
      }

      // 添加到所有心愿列表
      this.allWishes = [...this.allWishes, ...wishes];

      // 如果还没有开始持续飘动，则启动
      if (!this.wishInterval) {
        this.startContinuousWishes();
      }

      // 清空输入框
      this.prayer = "";
    },

    // 启动心愿持续飘动效果
    startContinuousWishes() {
      // 先清空现有心愿
      this.floatingWishes = [];

      // 每个2秒添加一个新的心愿
      this.wishInterval = setInterval(() => {
        if (this.allWishes.length === 0) {
          clearInterval(this.wishInterval);
          this.wishInterval = null;
          return;
        }

        // 循环显示所有心愿
        const wishText = this.allWishes[this.currentWishIndex];
        this.currentWishIndex =
          (this.currentWishIndex + 1) % this.allWishes.length;

        // 从左右两侧交替添加新的心愿
        const position =
          this.floatingWishes.length % 2 === 0 ? "left" : "right";

        // 添加新的心愿元素
        this.floatingWishes.push({
          id: Date.now(),
          text: wishText,
          position: position,
          delay: 0,
        });

        // 当心愿数量过多时，移除已经完成动画的心愿（约6秒后）
        if (this.floatingWishes.length > 20) {
          // 只保留最近的10个心愿
          this.floatingWishes = this.floatingWishes.slice(-10);
        }
      }, 2000);
    },
  },
  mounted() {
    // 初始化显示默认数量的香
    this.updateIncensePreview();
  },

  // 组件销毁时清除定时器
  beforeUnmount() {
    if (this.wishInterval) {
      clearInterval(this.wishInterval);
    }

    // 清除所有燃烧定时器
    this.clearAllBurnTimers();
  },
};
</script>

<style scoped>
.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  background: linear-gradient(to bottom, #f5f5f5, #e0e0e0);
}

.header {
  text-align: center;
  padding: 2rem 1rem;
  background: linear-gradient(to right, #8b4513, #a0522d);
  color: white;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
}

.header h1 {
  font-size: 2.5rem;
  margin-bottom: 0.5rem;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
}

.header p {
  font-size: 1.2rem;
  opacity: 0.9;
}

.main-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 2rem;
}

.incense-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2rem;
  max-width: 600px;
  width: 100%;
}

.incense-burner {
  position: relative;
  display: flex;
  justify-content: center;
  align-items: flex-end;
  height: 400px;
  width: 100%;
}

.burner-base {
  position: absolute;
  bottom: 0;
  width: 200px;
  height: 60px;
  background: radial-gradient(circle, #654321, #3e2723);
  border-radius: 50% 50% 10% 10%;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
}

.incense-stick {
  position: relative;
  margin: 0 15px;
  animation: fadeIn 0.5s ease-out;
}

.incense-body {
  width: 8px;
  height: 250px;
  background: linear-gradient(to bottom, #deb887, #a0522d);
  border-radius: 4px 4px 0 0;
  transform: translateY(-10px);
  position: relative;
}

.incense-flame {
  position: absolute;
  top: -15px;
  left: 50%;
  transform: translateX(-50%);
  width: 12px;
  height: 15px;
  background: radial-gradient(circle, #fff, #ffa500, #ff4500);
  border-radius: 50% 50% 50% 50% / 60% 60% 40% 40%;
  animation: flameFlicker 2s infinite alternate;
  box-shadow: 0 0 10px rgba(255, 69, 0, 0.6);
  transition: all 0.5s ease-out;
}

.incense-flame.extinguished {
  opacity: 0;
  width: 0;
  height: 0;
  box-shadow: none;
  animation: none;
}

.incense-smoke {
  position: absolute;
  top: -20px;
  left: 50%;
  transform: translateX(-50%);
  width: 6px;
  height: 50px;
  background: rgba(255, 255, 255, 0.4);
  border-radius: 50%;
  animation: smokeRise 3s infinite linear;
  transition: opacity 1s ease-out;
}

.incense-smoke.extinguished {
  opacity: 0;
  animation: none;
}

.control-panel {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  align-items: center;
  width: 100%;
  background: white;
  padding: 2rem;
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.incense-selector {
  display: flex;
  align-items: center;
  gap: 1rem;
  font-size: 1.1rem;
}

.incense-selector select {
  padding: 0.5rem 1rem;
  font-size: 1rem;
  border: 2px solid #a0522d;
  border-radius: 5px;
  background: white;
  cursor: pointer;
}

.time-selector {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
  font-size: 1.1rem;
}

.time-input-group {
  display: flex;
  gap: 1rem;
  align-items: center;
}

.time-select {
  padding: 0.5rem 1rem;
  font-size: 1rem;
  border: 2px solid #a0522d;
  border-radius: 5px;
  background: white;
  cursor: pointer;
}

.time-input {
  padding: 0.5rem;
  font-size: 1rem;
  border: 2px solid #a0522d;
  border-radius: 5px;
  width: 100px;
  text-align: center;
}

.burn-btn,
.pray-btn {
  padding: 0.8rem 2rem;
  font-size: 1.2rem;
  background: linear-gradient(to right, #8b4513, #a0522d);
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
}

.burn-btn:hover:not(:disabled),
.pray-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.3);
}

.burn-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  transform: none;
}

.prayer-section {
  margin-top: 2rem;
  width: 100%;
  max-width: 600px;
  background: white;
  padding: 2rem;
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.prayer-section h3 {
  text-align: center;
  margin-bottom: 1rem;
  color: #8b4513;
}

.prayer-section textarea {
  width: 100%;
  height: 100px;
  padding: 1rem;
  font-size: 1rem;
  border: 2px solid #deb887;
  border-radius: 5px;
  resize: vertical;
  margin-bottom: 1rem;
}

.prayer-section .pray-btn {
  width: 100%;
  display: block;
}

.footer {
  text-align: center;
  padding: 1rem;
  background: #3e2723;
  color: white;
  font-size: 0.9rem;
}

@keyframes flameFlicker {
  0% {
    transform: translateX(-50%) scale(1, 1);
  }
  50% {
    transform: translateX(-50%) scale(1.1, 0.9);
  }
  100% {
    transform: translateX(-50%) scale(0.9, 1.1);
  }
}

@keyframes smokeRise {
  0% {
    opacity: 0.7;
    transform: translateX(-50%) translateY(0) scale(1);
  }
  50% {
    opacity: 0.5;
    transform: translateX(-50%) translateY(-30px) scale(1.2);
  }
  100% {
    opacity: 0;
    transform: translateX(-50%) translateY(-60px) scale(1.5);
  }
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* 心愿飘动动画 - 左侧 */
.floating-wish.left {
  position: absolute;
  bottom: 100px;
  left: 20%;
  color: #fff;
  text-shadow: 1px 1px 3px rgba(0, 0, 0, 0.5);
  font-size: 16px;
  opacity: 0;
  animation: wishFloatLeft 6s forwards ease-out;
  white-space: nowrap;
  z-index: 10;
}

/* 心愿飘动动画 - 右侧 */
.floating-wish.right {
  position: absolute;
  bottom: 100px;
  right: 20%;
  color: #fff;
  text-shadow: 1px 1px 3px rgba(0, 0, 0, 0.5);
  font-size: 16px;
  opacity: 0;
  animation: wishFloatRight 6s forwards ease-out;
  white-space: nowrap;
  z-index: 10;
}

@keyframes wishFloatLeft {
  0% {
    opacity: 0;
    transform: translate(0, 0) scale(1);
    left: 20%;
  }
  20% {
    opacity: 1;
    transform: translate(20px, -20px) scale(1.1);
  }
  40% {
    opacity: 1;
    transform: translate(40px, -60px) scale(1.2);
  }
  60% {
    opacity: 0.8;
    transform: translate(60px, -120px) scale(1.3);
  }
  80% {
    opacity: 0.5;
    transform: translate(80px, -180px) scale(1.4);
  }
  100% {
    opacity: 0;
    transform: translate(100px, -300px) scale(1.5);
    left: 50%;
  }
}

@keyframes wishFloatRight {
  0% {
    opacity: 0;
    transform: translate(0, 0) scale(1);
    right: 20%;
  }
  20% {
    opacity: 1;
    transform: translate(-20px, -20px) scale(1.1);
  }
  40% {
    opacity: 1;
    transform: translate(-40px, -60px) scale(1.2);
  }
  60% {
    opacity: 0.8;
    transform: translate(-60px, -120px) scale(1.3);
  }
  80% {
    opacity: 0.5;
    transform: translate(-80px, -180px) scale(1.4);
  }
  100% {
    opacity: 0;
    transform: translate(-100px, -300px) scale(1.5);
    right: 50%;
  }
}
</style>
