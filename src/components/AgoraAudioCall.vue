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
import { ref, onUnmounted } from 'vue'
import AgoraRTC from 'agora-rtc-sdk-ng'

const client = ref(null)
const localAudioTrack = ref(null)
const isJoined = ref(false)
const APP_ID = '9ad089e29213419cb055404480669436'
const CHANNEL = 'bismillah'

const joinChannel = async () => {
  try {
    // 1. Initialize client with minimal configuration
    client.value = AgoraRTC.createClient({
      mode: 'rtc',
      codec: 'vp8'
    })

    // 2. Set up event listeners first
    setupEventListeners()

    // 3. Join channel without token
    await client.value.join(APP_ID, CHANNEL, null, null)

    // 4. Create and publish local track
    localAudioTrack.value = await AgoraRTC.createMicrophoneAudioTrack()
    await client.value.publish(localAudioTrack.value)
    
    isJoined.value = true
    console.log('Successfully joined channel')
  } catch (error) {
    console.error('Join failed:', error)
  }
}

const setupEventListeners = () => {
  // Handle remote user subscription with stability checks
  client.value.on('user-published', async (user, mediaType) => {
    if (mediaType !== 'audio') return
    
    try {
      // Verify user is still in channel
      await new Promise(resolve => setTimeout(resolve, 300))
      
      const userStillExists = client.value.remoteUsers.some(u => u.uid === user.uid)
      if (!userStillExists) {
        console.warn('User left before subscription')
        return
      }

      await client.value.subscribe(user, 'audio')
      user.audioTrack.play()
    } catch (err) {
      console.log('Subscribe error (non-critical):', err)
    }
  })

  // Monitor connection state
  client.value.on('connection-state-change', (state) => {
    console.log('Connection state changed:', state)
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