<script setup lang="ts">
import { ref } from 'vue'

const totalTimeMinutes = ref(1) // Temps total en minuts
const numAlarms = ref(5) // Número d'alarmes
const isRunning = ref(false)
const isPaused = ref(false)
const showAlarmModal = ref(false)
const timeRemaining = ref(0)
const alarmsTriggered = ref(0)
const bufferTime = 5 // Temps de buffer al principi i final (en segons)

let alarmTimeouts: number[] = []
let countdownInterval: number | null = null
let wakeLock: any = null
let totalPausedTime = 0
let pauseStartTime = 0
let startTime = 0
let nextAlarmIndex = 0
let scheduledAlarms: number[] = []

const playAlarmSound = () => {
  // Crea un so d'alarma més llarg i fort amb Web Audio API
  const audioContext = new (window.AudioContext || (window as any).webkitAudioContext)()
  const duration = 2 // Durada de 2 segons

  // Crea múltiples oscil·ladors per un so més intens
  for (let i = 0; i < 3; i++) {
    const oscillator = audioContext.createOscillator()
    const gainNode = audioContext.createGain()

    oscillator.connect(gainNode)
    gainNode.connect(audioContext.destination)

    // Freqüències alternades per fer-ho més sonor
    oscillator.frequency.value = i === 0 ? 800 : i === 1 ? 1000 : 600
    oscillator.type = 'square'

    // Volum més alt
    gainNode.gain.setValueAtTime(0.5, audioContext.currentTime + i * 0.2)
    gainNode.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + duration)

    oscillator.start(audioContext.currentTime + i * 0.2)
    oscillator.stop(audioContext.currentTime + duration)
  }
}

const generateRandomAlarmTimes = () => {
  const totalTimeSeconds = totalTimeMinutes.value * 60
  const availableTime = totalTimeSeconds - 2 * bufferTime
  const minGapBetweenAlarms = 30 // Mínim 30 segons entre alarmes
  const times: number[] = []

  // Genera temps aleatoris amb una distància mínima entre ells
  let attempts = 0
  const maxAttempts = 1000

  while (times.length < numAlarms.value && attempts < maxAttempts) {
    attempts++
    const randomTime = bufferTime + Math.random() * availableTime

    // Comprova que el nou temps tingui almenys 5 segons de distància amb els altres
    const tooClose = times.some((existingTime) => {
      return Math.abs(randomTime - existingTime) < minGapBetweenAlarms
    })

    if (!tooClose) {
      times.push(randomTime)
    }
  }

  // Converteix a mil·lisegons i ordena
  return times.map((t) => t * 1000).sort((a, b) => a - b)
}

const startAlarms = async () => {
  if (isRunning.value) return

  // Reset
  clearAllTimeouts()
  alarmsTriggered.value = 0
  timeRemaining.value = totalTimeMinutes.value * 60
  isRunning.value = true
  isPaused.value = false
  nextAlarmIndex = 0
  totalPausedTime = 0
  pauseStartTime = 0

  // Intenta mantenir la pantalla encesa (Wake Lock API)
  try {
    if ('wakeLock' in navigator) {
      wakeLock = await (navigator as any).wakeLock.request('screen')
      console.log('Wake Lock activat')
    }
  } catch (err) {
    console.log('Wake Lock no disponible:', err)
  }

  // Genera temps aleatoris per les alarmes
  scheduledAlarms = generateRandomAlarmTimes()

  // Inicia el countdown
  startTime = Date.now()
  startCountdown()
  scheduleNextAlarm()
}

const scheduleNextAlarm = () => {
  if (nextAlarmIndex >= scheduledAlarms.length) return

  const elapsedTime = Date.now() - startTime - totalPausedTime
  const timeUntilAlarm = scheduledAlarms[nextAlarmIndex] - elapsedTime

  if (timeUntilAlarm > 0) {
    const timeout = setTimeout(() => {
      triggerAlarm()
    }, timeUntilAlarm)
    alarmTimeouts.push(timeout)
  }
}

const triggerAlarm = () => {
  playAlarmSound()
  pauseTimer()
  showAlarmModal.value = true
  alarmsTriggered.value++
  nextAlarmIndex++
}

const continueAfterAlarm = () => {
  showAlarmModal.value = false
  resumeTimer()
  scheduleNextAlarm()
}

const startCountdown = () => {
  countdownInterval = setInterval(() => {
    if (!isPaused.value) {
      const elapsed = (Date.now() - startTime - totalPausedTime) / 1000
      timeRemaining.value = Math.max(0, totalTimeMinutes.value * 60 - elapsed)

      if (timeRemaining.value <= 0) {
        stopAlarms()
      }
    }
  }, 100)
}

const pauseTimer = () => {
  if (!isPaused.value) {
    isPaused.value = true
    pauseStartTime = Date.now()
  }
}

const resumeTimer = () => {
  if (isPaused.value) {
    isPaused.value = false
    totalPausedTime += Date.now() - pauseStartTime
  }
}

const togglePause = () => {
  if (isPaused.value) {
    resumeTimer()
  } else {
    pauseTimer()
  }
}

const stopAlarms = () => {
  clearAllTimeouts()
  isRunning.value = false
  isPaused.value = false
  showAlarmModal.value = false
  timeRemaining.value = 0
  totalPausedTime = 0
  pauseStartTime = 0
  nextAlarmIndex = 0

  // Allibera el Wake Lock
  if (wakeLock) {
    wakeLock.release()
    wakeLock = null
    console.log('Wake Lock alliberat')
  }
}

const clearAllTimeouts = () => {
  alarmTimeouts.forEach((timeout) => clearTimeout(timeout))
  alarmTimeouts = []
  if (countdownInterval) {
    clearInterval(countdownInterval)
    countdownInterval = null
  }
}

const formatTime = (seconds: number) => {
  const mins = Math.floor(seconds / 60)
  const secs = Math.floor(seconds % 60)
  return `${mins}:${secs.toString().padStart(2, '0')}`
}
</script>

<template>
  <div class="container">
    <h1>⏰ Alarmes Aleatòries</h1>

    <div class="controls">
      <div class="input-group">
        <label for="time">Temps total (minuts):</label>
        <input
          id="time"
          v-model.number="totalTimeMinutes"
          type="number"
          min="1"
          :disabled="isRunning"
        />
      </div>

      <div class="input-group">
        <label for="alarms">Número d'alarmes:</label>
        <input id="alarms" v-model.number="numAlarms" type="number" min="1" :disabled="isRunning" />
      </div>

      <button @click="startAlarms" :disabled="isRunning" class="play-button">▶️ Començar</button>

      <button @click="togglePause" :disabled="!isRunning || showAlarmModal" class="pause-button">
        {{ isPaused ? '▶️ Continuar' : '⏸️ Pausa' }}
      </button>

      <button @click="stopAlarms" :disabled="!isRunning" class="stop-button">⏹️ Aturar</button>
    </div>

    <div v-if="isRunning" class="status">
      <div class="time-display" :class="{ paused: isPaused }">
        <h2>{{ formatTime(timeRemaining) }}</h2>
        <p>{{ isPaused ? 'EN PAUSA' : 'Temps restant' }}</p>
      </div>

      <div class="alarms-display">
        <h3>{{ alarmsTriggered }} / {{ numAlarms }}</h3>
        <p>Alarmes sonades</p>
      </div>
    </div>

    <!-- Modal d'alarma -->
    <div v-if="showAlarmModal" class="modal-overlay">
      <div class="modal">
        <h2>🔔 Alarma!</h2>
        <p>Prem el botó per continuar</p>
        <button @click="continueAfterAlarm" class="continue-button">✓ Continuar</button>
      </div>
    </div>

    <div class="info">
      <p>
        ℹ️ Les alarmes sonaran de forma aleatòria amb un marge de {{ bufferTime }} segons al
        principi i al final.
      </p>
      <p>
        📱 <strong>Mòbils:</strong> Mantén la pantalla encesa i l'app en primer pla per millor
        funcionament.
      </p>
    </div>
  </div>
</template>

<style scoped>
.container {
  max-width: 600px;
  margin: 0 auto;
  padding: 2rem;
  text-align: center;
  font-family:
    -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
}

h1 {
  color: #2c3e50;
  margin-bottom: 2rem;
}

.controls {
  background: #f5f5f5;
  padding: 2rem;
  border-radius: 12px;
  margin-bottom: 2rem;
}

.input-group {
  margin-bottom: 1.5rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
}

label {
  font-weight: 600;
  color: #555;
}

input[type='number'] {
  width: 120px;
  padding: 0.75rem;
  font-size: 1.1rem;
  border: 2px solid #ddd;
  border-radius: 8px;
  text-align: center;
  transition: border-color 0.3s;
}

input[type='number']:focus {
  outline: none;
  border-color: #42b983;
}

input[type='number']:disabled {
  background: #e9ecef;
  cursor: not-allowed;
}

button {
  padding: 1rem 2rem;
  font-size: 1.1rem;
  font-weight: 600;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s;
  margin: 0.5rem;
}

.play-button {
  background: #42b983;
  color: white;
}

.play-button:hover:not(:disabled) {
  background: #369970;
  transform: translateY(-2px);
}

.stop-button {
  background: #e74c3c;
  color: white;
}

.stop-button:hover:not(:disabled) {
  background: #c0392b;
  transform: translateY(-2px);
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  transform: none;
}

.status {
  display: flex;
  justify-content: space-around;
  gap: 2rem;
  margin-bottom: 2rem;
}

.time-display,
.alarms-display {
  flex: 1;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 1.5rem;
  border-radius: 12px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
}

.time-display h2 {
  font-size: 3rem;
  margin: 0;
  font-weight: 700;
}

.alarms-display h3 {
  font-size: 2.5rem;
  margin: 0;
  font-weight: 700;
}

.time-display p,
.alarms-display p {
  margin: 0.5rem 0 0 0;
  opacity: 0.9;
  font-size: 0.9rem;
}

.info {
  background: #e7f3ff;
  padding: 1rem;
  border-radius: 8px;
  color: #0066cc;
  font-size: 0.9rem;
}

.pause-button {
  background: #f39c12;
  color: white;
}

.pause-button:hover:not(:disabled) {
  background: #e67e22;
  transform: translateY(-2px);
}

.time-display.paused {
  background: linear-gradient(135deg, #f39c12 0%, #e67e22 100%);
  animation: pulse 1.5s ease-in-out infinite;
}

@keyframes pulse {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.7;
  }
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  animation: fadeIn 0.3s ease-in;
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

.modal {
  background: white;
  padding: 3rem 2rem;
  border-radius: 16px;
  text-align: center;
  max-width: 400px;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.3);
  animation: slideUp 0.3s ease-out;
}

@keyframes slideUp {
  from {
    transform: translateY(50px);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}

.modal h2 {
  font-size: 2.5rem;
  margin: 0 0 1rem 0;
  color: #2c3e50;
}

.modal p {
  font-size: 1.2rem;
  color: #666;
  margin-bottom: 2rem;
}

.continue-button {
  background: #27ae60;
  color: white;
  padding: 1.2rem 3rem;
  font-size: 1.3rem;
  font-weight: 700;
}

.continue-button:hover {
  background: #229954;
  transform: translateY(-2px);
}

@media (max-width: 600px) {
  .status {
    flex-direction: column;
  }

  .container {
    padding: 1rem;
  }

  .modal {
    margin: 1rem;
    padding: 2rem 1.5rem;
  }
}
</style>
