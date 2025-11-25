<template>
  <div id="app">
    <div id="game-wrapper">
      <!-- Header -->
      <div id="header">
        <h1>Crazy Cup Filler</h1>
        <p class="subtitle">Fill the cup!</p>
      </div>

      <!-- Stats -->
      <div id="stats">
        <div class="stat-box">
          <span class="stat-label">Score</span>
          <span class="stat-value">{{ score }}</span>
        </div>
        <div class="stat-box">
          <span class="stat-label">Round</span>
          <span class="stat-value">{{ round }}/{{ totalRounds }}</span>
        </div>
        <div class="stat-box">
          <span class="stat-label">Accuracy</span>
          <span class="stat-value">{{ Math.round(accuracy) }}%</span>
        </div>
      </div>

      <!-- Game Info -->
      <div id="info">
        <div class="info-item">
          <span>Target: <strong>{{ target }}%</strong> (±{{ tolerance }}%)</span>
          <span>Time: <strong :style="{ color: timeLeft <= 5 ? '#d9534f' : '#333' }">{{ timeLeft }}s</strong></span>
        </div>
        <div class="info-item">
          <span>Level: <strong>{{ level }}%</strong></span>
        </div>
      </div>

      <!-- Tube -->
      <div id="tube-container">
        <div id="tube">
          <div id="water" :style="{ height: level + '%' }"></div>
          <div id="target-marker" :style="{ bottom: target + '%' }"></div>
        </div>
      </div>

      <!-- Controls -->
      <div id="controls">
        <button 
          class="btn btn-start"
          @click="startGame"
          :disabled="gameRunning"
        >
          {{ gameRunning ? 'Run' : 'Start' }}
        </button>
        <button 
          class="btn btn-fill"
          @mousedown="startHoldFill"
          @mouseup="stopHoldFill"
          @mouseleave="stopHoldFill"
          @touchstart="startHoldFill"
          @touchend="stopHoldFill"
          :disabled="!gameRunning"
        >
          Fill
        </button>
        <button 
          class="btn btn-pour"
          @mousedown="startHoldPour"
          @mouseup="stopHoldPour"
          @mouseleave="stopHoldPour"
          @touchstart="startHoldPour"
          @touchend="stopHoldPour"
          :disabled="!gameRunning"
        >
          Pour
        </button>
        <button 
          class="btn btn-reset"
          @click="resetGame"
        >
          Reset
        </button>
      </div>

      <!-- Settings -->
      <div id="settings">
        <h3>Settings</h3>
        
        <div class="setting">
          <label>Duration</label>
          <div class="slider-container">
            <input 
              type="range" 
              min="5" 
              max="15" 
              v-model.number="timerDuration"
              :disabled="gameRunning"
              class="slider"
            />
            <span>{{ timerDuration }}s</span>
          </div>
        </div>

        <div class="setting">
          <label>Speed</label>
          <div class="slider-container">
            <input 
              type="range" 
              min="1" 
              max="10" 
              v-model.number="fillRate"
              class="slider"
            />
            <span>{{ fillRate }}</span>
          </div>
        </div>

        <div class="setting">
          <label>Tolerance</label>
          <div class="slider-container">
            <input 
              type="range" 
              min="1" 
              max="20" 
              v-model.number="tolerance"
              class="slider"
            />
            <span>±{{ tolerance }}%</span>
          </div>
        </div>

        <div class="setting">
          <label>Rounds</label>
          <div class="slider-container">
            <input 
              type="range" 
              min="1" 
              max="10" 
              v-model.number="totalRounds"
              :disabled="gameRunning"
              class="slider"
            />
            <span>{{ totalRounds }}</span>
          </div>
        </div>

        <div class="setting">
          <label>Difficulty</label>
          <select v-model="difficulty" :disabled="gameRunning" class="select">
            <option value="easy">Easy</option>
            <option value="medium">Medium</option>
            <option value="hard">Hard</option>
          </select>
        </div>
      </div>

      <!-- Message -->
      <div v-if="message" id="message" :class="'msg-' + messageType">
        {{ message }}
      </div>
    </div>

    <!-- Game Over Modal -->
    <div v-if="gameOver" id="modal">
      <div id="modal-box">
        <h2>Game Over!</h2>
        <div id="results">
          <p><span>Score:</span> {{ score }}</p>
          <p><span>Rounds:</span> {{ round - 1 }}/{{ totalRounds }}</p>
          <p><span>Avg Accuracy:</span> {{ Math.round(totalAccuracy / (round - 1 || 1)) }}%</p>
        </div>
        <button class="btn btn-replay" @click="restartFromBeginning">
          Play Again
        </button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: "CrazyCupFiller",
  data() {
    return {
      level: 0,
      timerDuration: 10,
      timeLeft: 0,
      gameRunning: false,
      gameOver: false,
      target: 50,
      minTarget: 30,
      maxTarget: 80,
      tolerance: 5,
      fillRate: 3,
      fillIntervalId: null,
      pourIntervalId: null,
      gameTimerId: null,
      message: "",
      messageType: "",
      score: 0,
      round: 1,
      totalRounds: 3,
      accuracy: 0,
      totalAccuracy: 0,
      difficulty: "medium",
    };
  },
  watch: {
    difficulty() {
      this.setDifficultyRange();
    },
  },
  methods: {
    setDifficultyRange() {
      const settings = {
        easy: { min: 50, max: 80, tol: 8 },
        medium: { min: 30, max: 80, tol: 5 },
        hard: { min: 10, max: 90, tol: 3 },
      };
      const s = settings[this.difficulty];
      this.minTarget = s.min;
      this.maxTarget = s.max;
      this.tolerance = s.tol;
    },

    startGame() {
      if (this.gameRunning) return;
      this.resetGame();
      this.gameOver = false;
      
      this.target = Math.floor(Math.random() * (this.maxTarget - this.minTarget + 1)) + this.minTarget;
      this.timeLeft = this.timerDuration;
      this.gameRunning = true;
      this.message = "Go! Fill now!";
      this.messageType = "info";

      this.gameTimerId = setInterval(() => {
        this.timeLeft--;
        if (this.timeLeft <= 0) {
          this.endRound();
        }
      }, 1000);
    },

    endRound() {
      this.gameRunning = false;
      this.clearIntervals();

      const lower = this.target - this.tolerance;
      const upper = this.target + this.tolerance;
      const diff = Math.abs(this.level - this.target);
      const roundAccuracy = Math.max(0, 100 - diff * 2);
      const points = Math.round(roundAccuracy);

      this.accuracy = roundAccuracy;
      this.totalAccuracy += roundAccuracy;

      if (this.level >= lower && this.level <= upper) {
        this.message = `Perfect! ${this.level}% +${points}pts`;
        this.messageType = "success";
        this.score += points;
      } else {
        this.message = `Target ${this.target}%, got ${this.level}% +${Math.round(points * 0.3)}pts`;
        this.messageType = "fail";
        this.score += Math.round(points * 0.3);
      }

      setTimeout(() => {
        if (this.round < this.totalRounds) {
          this.round++;
          this.message = `Round ${this.round} - Click Start`;
          this.messageType = "info";
        } else {
          this.gameOver = true;
        }
      }, 2000);
    },

    resetGame() {
      this.clearIntervals();
      this.level = 0;
      this.timeLeft = 0;
      this.gameRunning = false;
      this.message = "";
    },

    restartFromBeginning() {
      this.score = 0;
      this.round = 1;
      this.accuracy = 0;
      this.totalAccuracy = 0;
      this.gameOver = false;
      this.level = 0;
      this.message = "";
    },

    clearIntervals() {
      clearInterval(this.fillIntervalId);
      clearInterval(this.pourIntervalId);
      clearInterval(this.gameTimerId);
      this.fillIntervalId = null;
      this.pourIntervalId = null;
      this.gameTimerId = null;
    },

    startHoldFill() {
      if (!this.gameRunning || this.fillIntervalId) return;
      this.fillIntervalId = setInterval(() => {
        this.level = Math.min(110, this.level + this.fillRate);
      }, 100);
    },

    stopHoldFill() {
      clearInterval(this.fillIntervalId);
      this.fillIntervalId = null;
    },

    startHoldPour() {
      if (!this.gameRunning || this.pourIntervalId) return;
      this.pourIntervalId = setInterval(() => {
        this.level = Math.max(0, this.level - this.fillRate);
      }, 100);
    },

    stopHoldPour() {
      clearInterval(this.pourIntervalId);
      this.pourIntervalId = null;
    },
  },
  beforeUnmount() {
    this.clearIntervals();
  },
};
</script>

<style scoped>
* { margin: 0; padding: 0; box-sizing: border-box; }

#app {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background: linear-gradient(135deg, #5b7adb 0%, #6b8ee8 100%);
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 8px;
}

#game-wrapper {
  background: white;
  border-radius: 12px;
  box-shadow: 0 8px 25px rgba(0,0,0,0.15);
  padding: 16px;
  max-width: 500px;
  width: 100%;
}

/* Header */
#header {
  text-align: center;
  margin-bottom: 10px;
  border-bottom: 1px solid #e0e0e0;
  padding-bottom: 6px;
}

#header h1 {
  font-size: 22px;
  color: #333;
  margin-bottom: 2px;
  font-weight: 700;
}

.subtitle {
  color: #777;
  font-size: 11px;
}

/* Stats */
#stats {
  display: flex;
  gap: 8px;
  margin-bottom: 10px;
}

.stat-box {
  flex: 1;
  background: linear-gradient(135deg, #5b7adb 0%, #6b8ee8 100%);
  color: white;
  padding: 6px;
  border-radius: 8px;
  text-align: center;
}

.stat-label {
  display: block;
  font-size: 8px;
  opacity: 0.85;
  text-transform: uppercase;
  letter-spacing: 0.3px;
  margin-bottom: 2px;
}

.stat-value {
  display: block;
  font-size: 16px;
  font-weight: 700;
}

/* Info */
#info {
  background: #f8f9fa;
  padding: 6px;
  border-radius: 8px;
  margin-bottom: 10px;
  display: flex;
  flex-direction: column;
  gap: 3px;
}

.info-item {
  font-size: 10px;
  color: #444;
  display: flex;
  justify-content: space-between;
  gap: 8px;
}

.info-item strong {
  color: #333;
  font-weight: 700;
}

/* Tube */
#tube-container {
  text-align: center;
  margin: 10px 0;
}

#tube {
  position: relative;
  width: 90px;
  height: 200px;
  margin: 0 auto;
  border: 7px solid #333;
  border-bottom: none;
  border-radius: 8px 8px 0 0;
  background: white;
  overflow: hidden;
  box-shadow: inset 0 0 6px rgba(0,0,0,0.1);
}

#water {
  position: absolute;
  bottom: 0;
  width: 100%;
  background: linear-gradient(180deg, #5b7adb 0%, #4a5fbd 100%);
  transition: height 80ms linear;
}

#target-marker {
  position: absolute;
  left: 0;
  right: 0;
  height: 3px;
  background: #ffc107;
  z-index: 10;
  box-shadow: 0 0 4px rgba(255, 193, 7, 0.5);
}

/* Controls */
#controls {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr 1fr;
  gap: 6px;
  margin-bottom: 10px;
}

.btn {
  padding: 6px;
  border: none;
  border-radius: 6px;
  font-size: 10px;
  font-weight: 700;
  cursor: pointer;
  transition: all 100ms;
  text-transform: uppercase;
  letter-spacing: 0.2px;
}

.btn:hover:not(:disabled) {
  transform: translateY(-1px);
  box-shadow: 0 2px 6px rgba(0,0,0,0.1);
}

.btn:active:not(:disabled) {
  transform: translateY(0);
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-start {
  background: linear-gradient(135deg, #5b7adb 0%, #6b8ee8 100%);
  color: white;
}

.btn-fill {
  background: linear-gradient(135deg, #5b7adb 0%, #6b8ee8 100%);
  color: white;
}

.btn-pour {
  background: linear-gradient(135deg, #5b7adb 0%, #6b8ee8 100%);
  color: white;
}

.btn-reset {
  background: #e8e8e8;
  color: #333;
}

.btn-replay {
  background: linear-gradient(135deg, #5b7adb 0%, #6b8ee8 100%);
  color: white;
  padding: 8px;
  font-size: 11px;
  grid-column: 1 / -1;
}

/* Settings */
#settings {
  background: #f8f9fa;
  padding: 10px;
  border-radius: 8px;
  margin-bottom: 10px;
  border-left: 3px solid #5b7adb;
}

#settings h3 {
  margin-bottom: 6px;
  color: #333;
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
}

.setting {
  margin-bottom: 6px;
}

.setting:last-child {
  margin-bottom: 0;
}

.setting label {
  display: block;
  font-size: 9px;
  color: #555;
  font-weight: 700;
  margin-bottom: 3px;
  text-transform: uppercase;
}

.slider-container {
  display: flex;
  gap: 6px;
  align-items: center;
}

.slider {
  flex: 1;
  height: 3px;
  border-radius: 2px;
  background: #ddd;
  outline: none;
  -webkit-appearance: none;
  appearance: none;
}

.slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: linear-gradient(135deg, #5b7adb 0%, #6b8ee8 100%);
  cursor: pointer;
}

.slider::-moz-range-thumb {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: linear-gradient(135deg, #5b7adb 0%, #6b8ee8 100%);
  cursor: pointer;
  border: none;
}

.slider-container span {
  font-size: 9px;
  font-weight: 700;
  color: #5b7adb;
  min-width: 28px;
  text-align: right;
}

.select {
  width: 100%;
  padding: 4px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
  color: #333;
  font-size: 9px;
  font-weight: 600;
  cursor: pointer;
}

/* Messages */
#message {
  padding: 6px;
  border-radius: 6px;
  text-align: center;
  font-size: 10px;
  font-weight: 600;
  animation: slideDown 300ms ease;
}

@keyframes slideDown {
  from { opacity: 0; transform: translateY(-4px); }
  to { opacity: 1; transform: translateY(0); }
}

#message.msg-success {
  background: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

#message.msg-fail {
  background: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

#message.msg-info {
  background: #d1ecf1;
  color: #0c5460;
  border: 1px solid #bee5eb;
}

/* Modal */
#modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0,0,0,0.6);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: 8px;
}

#modal-box {
  background: white;
  padding: 16px;
  border-radius: 10px;
  text-align: center;
  box-shadow: 0 10px 30px rgba(0,0,0,0.2);
  max-width: 360px;
  width: 100%;
}

#modal-box h2 {
  font-size: 18px;
  color: #333;
  margin-bottom: 10px;
  font-weight: 700;
}

#results {
  text-align: left;
  background: #f8f9fa;
  padding: 8px;
  border-radius: 6px;
  margin-bottom: 10px;
}

#results p {
  font-size: 10px;
  color: #555;
  margin-bottom: 4px;
  display: flex;
  justify-content: space-between;
}

#results p:last-child {
  margin-bottom: 0;
}

#results span {
  font-weight: 700;
  color: #333;
}

/* Responsive */
@media (max-width: 480px) {
  #game-wrapper {
    padding: 12px;
  }
  
  #header h1 {
    font-size: 20px;
  }
  
  #tube {
    width: 85px;
    height: 190px;
    border-width: 6px;
  }
  
  .stat-value {
    font-size: 14px;
  }
}

@media (max-width: 360px) {
  #game-wrapper {
    padding: 10px;
  }
  
  #header h1 {
    font-size: 18px;
  }
  
  #tube {
    width: 80px;
    height: 180px;
    border-width: 5px;
  }
  
  #controls {
    grid-template-columns: 1fr 1fr;
  }
  
  .btn {
    padding: 5px;
    font-size: 9px;
  }
}
</style>
