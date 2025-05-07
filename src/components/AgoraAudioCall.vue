<template>
  <div class="container">
    <h1>Agora Audio Call</h1>
    <div class="controls">
      <button @click="joinChannel" :disabled="isJoined">Join Channel</button>
      <button @click="leaveChannel" :disabled="!isJoined">Leave Channel</button>
      <button @click="toggleMic" :disabled="!isJoined">
        {{ isMuted ? "Unmute Mic" : "Mute Mic" }}
      </button>
    </div>
    <div v-if="isJoined" class="remote-audio-container">
      <h3>Remote Users:</h3>
      <div ref="remoteAudioContainer"></div>
    </div>
  </div>
</template>




<script setup>
import { ref, onUnmounted, markRaw } from 'vue'
import AgoraRTC from 'agora-rtc-sdk-ng'

// All Agora objects are marked non-reactive
const client = ref(null)
const localAudioTrack = ref(null)
const isJoined = ref(false)
const activeSubscriptions = new Set() // Tracks ongoing subscriptions

const APP_ID = '9ad089e29213419cb055404480669436'
const CHANNEL = 'bismillah'

const joinChannel = async () => {
  try {
    // 1. Initialize NON-REACTIVE client
    client.value = markRaw(AgoraRTC.createClient({
      mode: 'rtc',
      codec: 'vp8'
    }))

    // 2. Setup listeners
    setupEventListeners()

    // 3. Join channel
    await client.value.join(APP_ID, CHANNEL, null, null)

    // 4. Create NON-REACTIVE audio track
    const track = await AgoraRTC.createMicrophoneAudioTrack()
    localAudioTrack.value = markRaw(track)
    await client.value.publish(localAudioTrack.value)
    
    isJoined.value = true
  } catch (error) {
    console.error('Join failed:', error)
  }
}

const setupEventListeners = () => {
  client.value.on('user-published', async (user, mediaType) => {
    if (mediaType !== 'audio' || activeSubscriptions.has(user.uid)) return
    
    activeSubscriptions.add(user.uid)
    
    try {
      // Stability delay
      await new Promise(resolve => setTimeout(resolve, 300))
      
      // Verify user presence
      if (!client.value.remoteUsers.some(u => u.uid === user.uid)) {
        throw new Error('User left before subscription')
      }

      // Subscribe to NON-REACTIVE user object
      await client.value.subscribe(markRaw(user), 'audio')
      user.audioTrack.play()
    } catch (err) {
      console.warn('Subscription skipped:', err)
    } finally {
      activeSubscriptions.delete(user.uid)
    }
  })

  // Monitor connection state
  client.value.on('connection-state-change', (state) => {
    console.log('Connection state:', state)
  })
}

const leaveChannel = async () => {
  try {
    if (localAudioTrack.value) {
      localAudioTrack.value.close()
      await client.value.unpublish(localAudioTrack.value)
    }
    await client.value.leave()
    isJoined.value = false
    activeSubscriptions.clear()
  } catch (error) {
    console.error('Leave error:', error)
  }
}

onUnmounted(() => {
  if (isJoined.value) leaveChannel()
})
</script>




<style scoped>
.container {
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
  text-align: center;
}
.controls button {
  margin: 5px;
  padding: 10px 15px;
  font-size: 16px;
}
button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
.remote-audio-container {
  margin-top: 20px;
}
</style>