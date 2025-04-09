<template>
  <div class="chat-container">
    <div class="sidebar">
      <div class="user-info">
        <h3>{{ currentUser.username }}</h3>
        <button @click="handleLogout">Logout</button>
      </div>
      <div class="online-users">
        <h4>Online Users</h4>
        <ul>
          <li
            v-for="user in onlineUsers"
            :key="user.id"
            :class="{ active: selectedUser?.id === user.id }"
            @click="selectUser(user)"
          >
            {{ user.username }}
          </li>
        </ul>
      </div>
    </div>
    <div class="chat-area">
      <div v-if="selectedUser" class="chat-messages">
        <div
          v-for="message in messages"
          :key="message.id"
          :class="['message', message.sender_id === currentUser.id ? 'sent' : 'received']"
        >
          <div class="message-content">{{ message.content }}</div>
          <div class="message-time">{{ formatTime(message.timestamp) }}</div>
        </div>
      </div>
      <div v-else class="no-chat-selected">
        <p>Select a user to start chatting</p>
      </div>
      <div v-if="selectedUser" class="message-input">
        <input
          v-model="newMessage"
          @keyup.enter="sendMessage"
          placeholder="Type a message..."
        />
        <button @click="sendMessage">Send</button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import { useRouter } from 'vue-router'
import axios from 'axios'

interface User {
  id: number
  username: string
  last_seen: string
}

interface Message {
  id: number
  content: string
  sender_id: number
  receiver_id: number
  timestamp: string
}

const router = useRouter()
const currentUser = ref<User>({ id: 0, username: '', last_seen: '' })
const onlineUsers = ref<User[]>([])
const selectedUser = ref<User | null>(null)
const messages = ref<Message[]>([])
const newMessage = ref('')
const eventSource = ref<EventSource | null>(null)

const connectSSE = () => {
  const token = localStorage.getItem('token')
  if (!token || !currentUser.value.id) {
    router.push('/login')
    return
  }

  if (eventSource.value) {
    eventSource.value.close()
  }

  eventSource.value = new EventSource(`http://localhost:8000/events/${currentUser.value.id}`)

  eventSource.value.onmessage = (event) => {
    const data = JSON.parse(event.data)
    
    if (data.type === 'online_users') {
      onlineUsers.value = data.users
    } else {
      // New message
      messages.value.push(data)
    }
  }

  eventSource.value.onerror = () => {
    eventSource.value?.close()
    setTimeout(connectSSE, 1000)
  }
}

const fetchCurrentUser = async () => {
  try {
    const token = localStorage.getItem('token')
    const response = await axios.get('http://localhost:8000/users/me', {
      headers: { Authorization: `Bearer ${token}` }
    })
    currentUser.value = response.data
    connectSSE()
  } catch (error) {
    router.push('/login')
  }
}

const selectUser = (user: User) => {
  selectedUser.value = user
  fetchMessages()
}

const fetchMessages = async () => {
  if (!selectedUser.value) return

  try {
    const token = localStorage.getItem('token')
    const response = await axios.get(
      `http://localhost:8000/messages/${selectedUser.value.id}`,
      {
        headers: { Authorization: `Bearer ${token}` }
      }
    )
    messages.value = response.data
  } catch (error) {
    console.error('Error fetching messages:', error)
  }
}

const sendMessage = async () => {
  if (!newMessage.value.trim() || !selectedUser.value) return

  try {
    const token = localStorage.getItem('token')
    await axios.post(
      'http://localhost:8000/messages',
      {
        receiver_id: selectedUser.value.id,
        content: newMessage.value
      },
      {
        headers: { Authorization: `Bearer ${token}` }
      }
    )
    newMessage.value = ''
  } catch (error) {
    console.error('Error sending message:', error)
  }
}

const handleLogout = () => {
  if (eventSource.value) {
    eventSource.value.close()
  }
  localStorage.removeItem('token')
  router.push('/login')
}

const formatTime = (timestamp: string) => {
  return new Date(timestamp).toLocaleTimeString()
}

onMounted(async () => {
  await fetchCurrentUser()
})

onUnmounted(() => {
  if (eventSource.value) {
    eventSource.value.close()
  }
})
</script>

<style scoped>
.chat-container {
  display: flex;
  height: 100vh;
}

.sidebar {
  width: 250px;
  background-color: #f8f9fa;
  border-right: 1px solid #dee2e6;
  padding: 1rem;
}

.user-info {
  margin-bottom: 1rem;
  padding-bottom: 1rem;
  border-bottom: 1px solid #dee2e6;
}

.user-info h3 {
  margin: 0 0 0.5rem 0;
}

.user-info button {
  width: 100%;
  padding: 0.5rem;
  background-color: #dc3545;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.online-users h4 {
  margin: 0 0 1rem 0;
}

.online-users ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.online-users li {
  padding: 0.5rem;
  cursor: pointer;
  border-radius: 4px;
}

.online-users li:hover {
  background-color: #e9ecef;
}

.online-users li.active {
  background-color: #007bff;
  color: white;
}

.chat-area {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.chat-messages {
  flex: 1;
  padding: 1rem;
  overflow-y: auto;
}

.message {
  margin-bottom: 1rem;
  max-width: 70%;
}

.message.sent {
  margin-left: auto;
}

.message.received {
  margin-right: auto;
}

.message-content {
  padding: 0.5rem 1rem;
  border-radius: 1rem;
  background-color: #e9ecef;
}

.message.sent .message-content {
  background-color: #007bff;
  color: white;
}

.message-time {
  font-size: 0.75rem;
  color: #6c757d;
  margin-top: 0.25rem;
}

.no-chat-selected {
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
  color: #6c757d;
}

.message-input {
  padding: 1rem;
  border-top: 1px solid #dee2e6;
  display: flex;
  gap: 0.5rem;
}

.message-input input {
  flex: 1;
  padding: 0.5rem;
  border: 1px solid #dee2e6;
  border-radius: 4px;
}

.message-input button {
  padding: 0.5rem 1rem;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}
</style> 