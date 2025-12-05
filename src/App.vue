<template>
  <div id="app" class="sci-fi-container">
    <!-- 1. 摄像头原画 (隐藏显示，仅用于识别) -->
    <video ref="inputVideo" class="input-video" playsinline style="display: none;"></video>
    
    <!-- 2. 背景显示层 (用于渲染处理后的镜像画面) -->
    <canvas ref="outputCanvas" class="camera-bg"></canvas>

    <!-- 3. 手势特效层 (最上层) -->
    <canvas ref="gestureCanvas" class="gesture-canvas"></canvas>

    <!-- 视觉滤镜 -->
    <div class="camera-filter"></div>
    <div class="screen-overlay"></div>
    <div class="vignette"></div>

    <!-- UI 层 -->
    <header class="top-bar">
      <div class="sys-info">
        <span class="sys-name">LUOLUO OS <small>GESTURE-Link</small></span>
        <span class="status-badge" :class="{ active: isHandDetected }">
          {{ isHandDetected ? 'HAND TRACKING: ACTIVE' : 'SEARCHING HAND...' }}
        </span>
      </div>
      <div class="clock">{{ currentTime }}</div>
    </header>

    <main class="main-hud">
      <div class="side-panel left-panel">
        <div class="data-block" v-for="i in 5" :key="`l-${i}`">
          <div class="bar" :style="{ width: Math.random() * 100 + '%' }"></div>
          <small>TRK.DAT.0{{i}}</small>
        </div>
      </div>

      <!-- 中央反应堆 (添加 ref 以便手势判定) -->
      <div class="reactor-container" ref="reactorBtn" @click="toggleVoice">
        <div class="ring ring-1"></div>
        <div class="ring ring-2"></div>
        <div class="ring ring-3" :class="{ 'pinched': isPinching }"></div>
        <canvas ref="audioCircle" class="audio-circle" width="400" height="400"></canvas>
        <div class="core-circle" :class="{ 'speaking': isSpeaking }">
          <div class="core-text">珞珞</div>
          <div class="core-glow"></div>
        </div>
      </div>

      <div class="side-panel right-panel">
        <div class="hex-grid"><div class="hex" v-for="i in 4" :key="`h-${i}`"></div></div>
        <div class="text-block">
          GESTURE: {{ isPinching ? 'PINCH (CLICK)' : 'HOVER' }}<br>
          X: {{ handX.toFixed(0) }} | Y: {{ handY.toFixed(0) }}
        </div>
      </div>
    </main>

    <footer class="interaction-area">
      <div class="chat-display" ref="chatBox">
        <transition-group name="list" tag="div">
          <div v-for="(msg, idx) in messages" :key="idx" class="msg-row" :class="msg.type">
            <div class="msg-marker"></div>
            <div class="msg-content">
              <div class="msg-header">{{ msg.type === 'user' ? 'COMMANDER' : 'LUOLUO' }}</div>
              <div class="msg-text">{{ msg.text }}</div>
            </div>
          </div>
        </transition-group>
      </div>
      <div class="input-bar">
        <div class="deco-bracket">[</div>
        <input v-model="inputText" @keyup.enter="handleSend" type="text" placeholder="GESTURE OR VOICE..." />
        <div class="deco-bracket">]</div>
      </div>
    </footer>
  </div>
</template>

<script>
export default {
  name: 'LuoLuoGesture',
  data() {
    return {
      // --- 基础状态 ---
      messages: [{ type: 'ai', text: '视觉与手势模块已加载。请举起手进行操作。' }],
      inputText: '',
      isListening: false,
      isSpeaking: false,
      currentTime: '',
      
      // --- 手势相关状态 ---
      isHandDetected: false,
      isPinching: false,
      handX: 0,
      handY: 0,
      lastPinchTime: 0,
      
      // --- 粒子系统 ---
      particles: [],
      
      // --- 核心对象 ---
      camera: null,
      hands: null,
      gestureCtx: null,
      outputCtx: null,
      
      // --- 语音对象 (关键修复：确保定义) ---
      synth: window.speechSynthesis, 
      recognition: null,
      
      // --- 音频可视化 ---
      audioCtx: null,
      analyser: null,
      dataArray: null,
      rafId: null,
    };
  },
  async mounted() {
    this.updateTime();
    setInterval(this.updateTime, 1000);
    
    // 1. 动态加载 MediaPipe 脚本 (无需 npm install)
    await this.loadScript('https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js');
    await this.loadScript('https://cdn.jsdelivr.net/npm/@mediapipe/control_utils/control_utils.js');
    await this.loadScript('https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js');
    await this.loadScript('https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js');

    // 2. 初始化系统
    this.initGestureSystem();
    this.initParticles();
    this.initSpeech();
    
    // 3. 窗口大小调整监听
    window.addEventListener('resize', this.resizeCanvas);
  },
  methods: {
    // --- 脚本加载辅助 ---
    loadScript(src) {
      return new Promise((resolve, reject) => {
        const script = document.createElement('script');
        script.src = src;
        script.onload = resolve;
        script.onerror = reject;
        document.body.appendChild(script);
      });
    },

    // --- MediaPipe 手势初始化 ---
    initGestureSystem() {
      const videoElement = this.$refs.inputVideo;
      const outputCanvas = this.$refs.outputCanvas;
      this.outputCtx = outputCanvas.getContext('2d');
      
      this.resizeCanvas(); 

      // 配置 Hands 模型
      this.hands = new window.Hands({
        locateFile: (file) => {
          return `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`;
        }
      });

      this.hands.setOptions({
        maxNumHands: 1,
        modelComplexity: 1,
        minDetectionConfidence: 0.5,
        minTrackingConfidence: 0.5
      });

      this.hands.onResults(this.onHandsResults);

      // 配置摄像头
      this.camera = new window.Camera(videoElement, {
        onFrame: async () => {
          await this.hands.send({ image: videoElement });
        },
        width: 1280,
        height: 720
      });
      
      this.camera.start();
    },

    // --- 核心手势处理逻辑 ---
    onHandsResults(results) {
      // 1. 绘制背景视频 (处理镜像)
      const canvas = this.$refs.outputCanvas;
      const ctx = this.outputCtx;
      
      ctx.save();
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      ctx.translate(canvas.width, 0);
      ctx.scale(-1, 1); // 镜像翻转
      ctx.drawImage(results.image, 0, 0, canvas.width, canvas.height);
      ctx.restore();

      // 2. 处理手部数据
      if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        this.isHandDetected = true;
        const landmarks = results.multiHandLandmarks[0];
        
        // 获取食指指尖 (Index Tip: 8) 和 大拇指指尖 (Thumb Tip: 4)
        // 注意：因为画面被镜像了，所以 x 坐标要用 1 - x
        const indexTip = landmarks[8];
        const thumbTip = landmarks[4];
        
        const x = (1 - indexTip.x) * canvas.width;
        const y = indexTip.y * canvas.height;
        
        this.handX = x;
        this.handY = y;

        // 生成粒子特效
        this.spawnParticles(x, y);

        // 计算捏合距离 (Pinch) - 欧几里得距离
        const distance = Math.sqrt(
          Math.pow(indexTip.x - thumbTip.x, 2) + 
          Math.pow(indexTip.y - thumbTip.y, 2)
        );

        // 判定点击 (阈值可调，0.05 比较合适)
        if (distance < 0.05) {
          if (!this.isPinching) {
            // 刚开始捏合 -> 触发点击
            this.triggerClick(x, y);
          }
          this.isPinching = true;
        } else {
          this.isPinching = false;
        }

      } else {
        this.isHandDetected = false;
      }
    },

    // --- 虚拟点击逻辑 (已修复语音播放) ---
    triggerClick(x, y) {
      const now = Date.now();
      if (now - this.lastPinchTime < 500) return; // 防抖
      this.lastPinchTime = now;

      // 1. 产生点击爆炸特效
      this.spawnExplosion(x, y);
      
      // 2. 播放音效
      this.playClickSound();

      // 3. 碰撞检测 (检测是否点中反应堆)
      const reactor = this.$refs.reactorBtn;
      if (reactor) {
        const rect = reactor.getBoundingClientRect();
        
        // 简单的矩形碰撞检测
        if (x >= rect.left && x <= rect.right && y >= rect.top && y <= rect.bottom) {
          console.log("Reactor Hit!");
          
          // 语音反馈
          const reply = '手势确认。语音链路已激活。';
          this.addMessage('ai', reply);
          this.speak(reply);
          
          // 尝试切换麦克风状态
          this.toggleVoice(); 
        }
      }
    },

    playClickSound() {
      // 使用 Web Audio API 生成简单的机械点击音
      const AudioContext = window.AudioContext || window.webkitAudioContext;
      if (!AudioContext) return;
      
      const ctx = new AudioContext();
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();
      
      osc.type = 'sine';
      osc.frequency.setValueAtTime(800, ctx.currentTime);
      osc.frequency.exponentialRampToValueAtTime(100, ctx.currentTime + 0.1);
      
      gain.gain.setValueAtTime(0.3, ctx.currentTime);
      gain.gain.exponentialRampToValueAtTime(0.01, ctx.currentTime + 0.1);
      
      osc.connect(gain);
      gain.connect(ctx.destination);
      osc.start();
      osc.stop(ctx.currentTime + 0.1);
    },

    // --- 粒子特效系统 ---
    resizeCanvas() {
      // 调整所有 Canvas 大小以匹配屏幕
      [this.$refs.outputCanvas, this.$refs.gestureCanvas].forEach(el => {
        if (el) {
          el.width = window.innerWidth;
          el.height = window.innerHeight;
        }
      });
    },

    initParticles() {
      this.gestureCtx = this.$refs.gestureCanvas.getContext('2d');
      this.animateParticles();
    },

    spawnParticles(x, y) {
      // 移动时的拖尾
      for (let i = 0; i < 2; i++) {
        this.particles.push({
          x: x + (Math.random() - 0.5) * 10,
          y: y + (Math.random() - 0.5) * 10,
          vx: (Math.random() - 0.5) * 2,
          vy: (Math.random() - 0.5) * 2,
          life: 1.0,
          color: '#00f3ff',
          size: Math.random() * 3 + 1,
          type: 'trail'
        });
      }
    },

    spawnExplosion(x, y) {
      // 点击时的爆炸
      for (let i = 0; i < 20; i++) {
        const angle = Math.random() * Math.PI * 2;
        const speed = Math.random() * 5 + 2;
        this.particles.push({
          x: x,
          y: y,
          vx: Math.cos(angle) * speed,
          vy: Math.sin(angle) * speed,
          life: 1.0,
          color: '#ffaa00', // 橙色爆炸
          size: Math.random() * 5 + 2,
          type: 'boom'
        });
      }
    },

    animateParticles() {
      const ctx = this.gestureCtx;
      const canvas = this.$refs.gestureCanvas;
      
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      
      // 绘制光标 (十字准星)
      if (this.isHandDetected) {
        ctx.beginPath();
        ctx.strokeStyle = this.isPinching ? '#ff3300' : '#00f3ff';
        ctx.lineWidth = 2;
        ctx.moveTo(this.handX - 20, this.handY);
        ctx.lineTo(this.handX + 20, this.handY);
        ctx.moveTo(this.handX, this.handY - 20);
        ctx.lineTo(this.handX, this.handY + 20);
        ctx.stroke();
        
        // 绘制圆圈
        ctx.beginPath();
        ctx.arc(this.handX, this.handY, 15, 0, Math.PI * 2);
        if (this.isPinching) {
            ctx.fillStyle = 'rgba(255, 51, 0, 0.5)';
            ctx.fill();
        }
        ctx.stroke();
      }

      // 更新和绘制粒子
      for (let i = this.particles.length - 1; i >= 0; i--) {
        const p = this.particles[i];
        p.x += p.vx;
        p.y += p.vy;
        p.life -= 0.04;
        
        if (p.life <= 0) {
          this.particles.splice(i, 1);
          continue;
        }
        
        ctx.globalAlpha = p.life;
        ctx.fillStyle = p.color;
        ctx.beginPath();
        ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
        ctx.fill();
      }
      ctx.globalAlpha = 1.0;
      
      requestAnimationFrame(this.animateParticles);
    },

    // --- 语音与交互逻辑 ---
    updateTime() {
      this.currentTime = new Date().toLocaleTimeString('en-US', { hour12: false });
    },
    
    initSpeech() {
       const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
       if (!SpeechRecognition) return;
       this.recognition = new SpeechRecognition();
       this.recognition.lang = 'zh-CN';
       this.recognition.continuous = false;
       
       this.recognition.onstart = () => { 
           this.isListening = true; 
           this.startAudioVisualizer(); 
       };
       this.recognition.onend = () => { 
           this.isListening = false; 
           this.stopAudioVisualizer(); 
       };
       this.recognition.onresult = (e) => {
         this.inputText = e.results[0][0].transcript;
         this.handleSend();
       };
    },
    
    toggleVoice() {
      if (this.isListening) this.recognition.stop();
      else this.recognition.start();
    },

    handleSend() {
      if (!this.inputText.trim()) return;
      const t = this.inputText;
      this.addMessage('user', t); 
      this.inputText = '';
      setTimeout(() => this.aiReply(t), 800);
    },

    addMessage(type, text) {
      this.messages.push({ type, text });
      if (this.messages.length > 5) this.messages.shift();
      this.$nextTick(() => {
        const box = this.$refs.chatBox;
        if (box) box.scrollTop = box.scrollHeight;
      });
    },

    aiReply(t) {
      let r = "指令已接收。";
      if(t.includes("珞珞")) r = "我在，随时为您效劳。";
      if(t.includes("分析")) r = "扫描完成。目标：人类，威胁等级：低。";
      
      this.addMessage('ai', r);
      this.speak(r);
    },

    speak(t) {
      // 安全检查：防止 synth 丢失
      if (!this.synth) {
          this.synth = window.speechSynthesis;
          if (!this.synth) return;
      }
      
      if(this.synth.speaking) this.synth.cancel();
      
      const u = new SpeechSynthesisUtterance(t);
      const vs = this.synth.getVoices();
      const v = vs.find(x => x.lang.includes('zh') || x.name.includes('Google') || x.name.includes('Microsoft'));
      if(v) u.voice = v;
      
      u.rate = 1.1; // 稍微快一点，更像AI
      u.onstart = () => this.isSpeaking = true;
      u.onend = () => this.isSpeaking = false;
      this.synth.speak(u);
    },

    // --- 音频可视化 ---
    async startAudioVisualizer() {
       if (!this.audioCtx) {
           this.audioCtx = new (window.AudioContext || window.webkitAudioContext)();
           this.analyser = this.audioCtx.createAnalyser();
           this.analyser.fftSize = 512;
           this.dataArray = new Uint8Array(this.analyser.frequencyBinCount);
       }
       try {
           const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
           const source = this.audioCtx.createMediaStreamSource(stream);
           source.connect(this.analyser);
           this.drawCircleWave();
       } catch(e) {
           console.error("Mic access denied or error:", e);
       }
    },
    
    stopAudioVisualizer() { cancelAnimationFrame(this.rafId); },
    
    drawCircleWave() {
      this.rafId = requestAnimationFrame(this.drawCircleWave);
      this.analyser.getByteFrequencyData(this.dataArray);
      
      const canvas = this.$refs.audioCircle;
      const ctx = canvas.getContext('2d');
      const cx = canvas.width/2;
      const cy = canvas.height/2;
      const r = 100;
      
      ctx.clearRect(0,0,canvas.width,canvas.height);
      ctx.beginPath();
      ctx.strokeStyle = '#00f3ff';
      ctx.lineWidth = 2;
      
      for(let i=0; i<this.dataArray.length; i+=2) {
          const v = this.dataArray[i];
          const rad = (i/this.dataArray.length)*Math.PI*2;
          const h = (v/255)*60;
          ctx.moveTo(cx+Math.cos(rad)*r, cy+Math.sin(rad)*r);
          ctx.lineTo(cx+Math.cos(rad)*(r+h), cy+Math.sin(rad)*(r+h));
      }
      ctx.stroke();
    }
  },
  beforeDestroy() {
    if (this.camera) this.camera.stop();
  }
};
</script>

<style lang="scss">
/* --- 引入科幻字体 --- */
@import url('https://fonts.googleapis.com/css2?family=Share+Tech+Mono&family=Orbitron:wght@400;700;900&display=swap');

:root {
  --neon-blue: #00f3ff;
  --neon-orange: #ffaa00;
  --bg-dark: #000;
  --font-mono: 'Share Tech Mono', monospace;
  --font-head: 'Orbitron', sans-serif;
}

body { margin: 0; overflow: hidden; background: #000; color: var(--neon-blue); font-family: var(--font-mono); }

.sci-fi-container {
  width: 100vw; height: 100vh; position: relative;
  display: flex; flex-direction: column; justify-content: space-between;
}

/* --- 层级管理 --- */
/* 1. 隐藏原始视频，2. 镜像后Canvas作为背景，3. 手势特效最上层 */
.input-video { display: none; }
.camera-bg { position: fixed; top: 0; left: 0; width: 100%; height: 100%; z-index: -2; object-fit: cover; }
.camera-filter { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0, 20, 40, 0.4); mix-blend-mode: overlay; z-index: -1; }
.screen-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: linear-gradient(rgba(0,0,0,0) 50%, rgba(0,0,0,0.2) 50%); background-size: 100% 4px; pointer-events: none; z-index: 1; }
.gesture-canvas { position: fixed; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none; z-index: 100; }

/* --- UI 元素 --- */
.top-bar { display: flex; justify-content: space-between; padding: 20px 40px; z-index: 20; background: linear-gradient(to bottom, rgba(0,0,0,0.8), transparent); }
.sys-info { font-family: var(--font-head); font-weight: 700; display: flex; gap: 20px; align-items: center; }
.status-badge { border: 1px solid var(--neon-blue); padding: 2px 8px; font-size: 0.8rem; opacity: 0.5; &.active { opacity: 1; background: var(--neon-blue); color: #000; box-shadow: 0 0 10px var(--neon-blue); } }

.main-hud { flex: 1; display: flex; justify-content: center; align-items: center; z-index: 20; position: relative; gap: 50px; }
.side-panel { width: 150px; display: flex; flex-direction: column; gap: 10px; opacity: 0.8; }
.bar { height: 4px; background: var(--neon-blue); margin-bottom: 2px; box-shadow: 0 0 5px var(--neon-blue); }

.reactor-container {
  width: 400px; height: 400px; position: relative; display: flex; justify-content: center; align-items: center; cursor: pointer;
  transition: transform 0.1s;
  &:active { transform: scale(0.95); }
}
.ring { position: absolute; border-radius: 50%; border: 1px solid var(--neon-blue); }
.ring-1 { width: 380px; height: 380px; border-width: 2px; border-top-color: transparent; animation: spin 20s linear infinite; }
.ring-2 { width: 320px; height: 320px; border: 1px dashed var(--neon-blue); animation: spin 30s linear infinite reverse; }
.ring-3 { 
  width: 280px; height: 280px; border: 5px solid var(--neon-blue); border-color: var(--neon-blue) transparent transparent transparent; 
  animation: spin 4s linear infinite; 
  &.pinched { border-color: var(--neon-orange) transparent transparent transparent; box-shadow: 0 0 20px var(--neon-orange); }
}

.core-circle {
  width: 140px; height: 140px; border-radius: 50%; background: rgba(0,0,0,0.7); border: 2px solid rgba(0,243,255,0.5); display: flex; justify-content: center; align-items: center; position: relative; backdrop-filter: blur(5px);
  .core-text { font-family: var(--font-head); font-size: 2rem; font-weight: 900; z-index: 2; }
  .core-glow { position: absolute; width: 100%; height: 100%; border-radius: 50%; animation: pulse 2s infinite; }
  &.speaking .core-glow { animation: flicker 0.1s infinite; background: radial-gradient(circle, #fff, var(--neon-blue)); }
}

.audio-circle { position: absolute; pointer-events: none; }

.interaction-area { z-index: 20; padding: 0 20% 40px; display: flex; flex-direction: column; gap: 20px; background: linear-gradient(to top, rgba(0,0,0,0.9), transparent); }
.chat-display { height: 150px; overflow: hidden; display: flex; flex-direction: column; justify-content: flex-end; mask-image: linear-gradient(to top, black 80%, transparent); }
.msg-row { display: flex; margin-bottom: 10px; .msg-marker { width: 3px; background: var(--neon-blue); margin-right: 10px; } .msg-text { background: rgba(0,0,0,0.6); padding: 4px 8px; } }
.input-bar { display: flex; align-items: center; gap: 10px; input { flex: 1; background: transparent; border: none; border-bottom: 1px solid var(--neon-blue); color: #fff; padding: 10px; text-align: center; font-family: var(--font-mono); font-size: 1.2rem; &:focus { outline: none; } } .deco-bracket { font-size: 2rem; color: var(--neon-blue); transform: scaleY(1.5); } }

/* --- 动画关键帧 --- */
@keyframes spin { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }
@keyframes pulse { 0% { opacity: 0.3; } 50% { opacity: 0.6; } 100% { opacity: 0.3; } }
@keyframes flicker { 0% { opacity: 0.9; } 50% { opacity: 0.4; } 100% { opacity: 1; } }
</style>