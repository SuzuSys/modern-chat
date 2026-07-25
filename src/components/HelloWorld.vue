<template>
  <v-container class="py-4">
    <v-card class="mx-auto" max-width="640" rounded="lg">
      <v-card-text>
        <v-text-field
          v-model="inputtedMyName"
          counter
          density="compact"
          hide-details="auto"
          label="Your name"
          maxlength="20"
          :rules="[nameRules.required, nameRules.counter]"
          variant="outlined"
        >
          <template #append>
            <v-btn
              block
              color="primary"
              :disabled="!inputtedMyName.trim()"
              min-height="40"
              variant="outlined"
              @click="registerName"
            >
              Submit
            </v-btn>
          </template>
        </v-text-field>

        <div v-if="!!myName">
          Your name is:
          <code> {{ myName }} </code>
        </div>

        <div v-if="!!myName">
          Your peer ID is:
          <code> {{ myId }} </code>
          <CopyClipboard :text="myId" />
        </div>
      </v-card-text>
    </v-card>
  </v-container>

  <v-container v-if="!!myName" class="py-4">
    <v-card class="mx-auto" max-width="640" rounded="lg">
      <v-card-text>
        <v-text-field
          v-model="destId"
          density="compact"
          :disabled="isConnecting"
          hide-details="auto"
          label="Dest peer ID"
          variant="outlined"
        >
          <template #append>
            <v-btn
              block
              color="primary"
              :disabled="isConnecting || !destId.trim()"
              :loading="isConnecting"
              variant="outlined"
              @click="handleConnect"
            >
              {{ isConnecting ? "Connecting..." : "Connect" }}
            </v-btn>
          </template>
        </v-text-field>
      </v-card-text>
    </v-card>
  </v-container>

  <v-container class="py-4">
    <v-table>
      <thead>
        <tr>
          <th class="text-left">Name</th>
          <th class="text-left">ID</th>
        </tr>
      </thead>

      <tbody>
        <tr v-for="[id, con] in connectedDestId" :key="id">
          <td>{{ con.name }}</td>
          <td>{{ con.conn.peer }}</td>
        </tr>
      </tbody>
    </v-table>
  </v-container>

  <v-container class="py-4">
    <v-card class="mx-auto" max-width="640" rounded="lg">
      <v-container>
        <v-row align="center" density="compact">
          <v-col cols="12" sm>
            <v-text-field
              v-model="sendtoId"
              density="compact"
              hide-details="auto"
              label="Target peer ID"
              variant="outlined"
            />
          </v-col>

          <v-col cols="12" sm>
            <v-text-field
              v-model="messageToSend"
              density="compact"
              hide-details="auto"
              label="Message"
              variant="outlined"
            />
          </v-col>

          <v-col cols="12" sm="auto">
            <v-btn
              block
              color="primary"
              :disabled="!messageToSend.trim() && !sendtoId.trim()"
              :loading="isConnecting"
              variant="outlined"
              @click="SendMessage"
            >
              Send
            </v-btn>
          </v-col>
        </v-row>
      </v-container>
    </v-card>
  </v-container>

  <v-snackbar-queue v-model="messages" :timeout="2000" />
</template>

<script setup lang="ts">
  import type { DataConnection } from 'peerjs'
  import { Peer } from 'peerjs'
  import { ref, type Ref } from 'vue'

  import CopyClipboard from './CopyClipboard.vue'

  const nameRules = {
    required: (value: string) => !!value || 'Required.',
    counter: (value: string) => value.length <= 20 || 'Max 20 characters',
  }

  const peer = new Peer()

  const sendtoId = ref('')
  const messageToSend = ref('')

  const myId = ref('')
  const inputtedMyName = ref('')
  const myName = ref('')
  const destId = ref('')
  const isConnecting = ref(false)
  type PeerConn = {
    conn: DataConnection
    name: string
  }
  const connectedDestId = ref(new Map<string, PeerConn>())
  const messages: Ref<
    Array<{
      text: string
      color: string
      variant: 'text' | 'flat' | 'elevated' | 'outlined' | 'plain' | 'tonal'
    }>
  > = ref([])

  type PeerData = {
    type: 'name' | 'message'
    content: string
    handshakeFirst: boolean
  }

  peer.on('open', id => {
    myId.value = id
  })

  function isPeerData (data: unknown): data is PeerData {
    // 1. オブジェクトであり、nullではないことをチェック
    if (typeof data !== 'object' || data === null) {
      return false
    }

    // プロパティに安全にアクセスするために型をキャスト
    const obj = data as Record<string, unknown>

    // 2. type プロパティが "name" または "message" かチェック
    const hasValidType = obj.type === 'name' || obj.type === 'message'

    // 3. content プロパティが文字列かチェック
    const hasValidContent = typeof obj.content === 'string'

    // 4. handshakeFirst プロパティが真偽値がチェック
    const hasValidHandshakeFirst = typeof obj.handshakeFirst === 'boolean'

    return hasValidType && hasValidContent && hasValidHandshakeFirst
  }

  async function handleConnect () {
    if (isConnecting.value) {
      return
    }

    isConnecting.value = true
    try {
      await new Promise((resolve, reject) => {
        const conn = peer.connect(destId.value.trim())
        connectedDestId.value.set(conn.peer, { conn, name: '' })
        conn.on('open', () => {
          isConnecting.value = false
          conn.on('data', receivePeerMessageWrapper(conn))
          destId.value = ''
          const peerData: PeerData = {
            type: 'name',
            content: myName.value,
            handshakeFirst: true,
          }
          conn.send(peerData)
          console.log('sended!')
          resolve(conn)
        })

        conn.on('error', err => reject(err))
      })
    } catch (error) {
      console.error('Connection failed:', error)
      isConnecting.value = false
      messages.value.push({
        text: 'Connection failed',
        color: 'error',
        variant: 'tonal',
      })
    }
  }

  function receivePeerMessageWrapper (conn: DataConnection) {
    return (data: unknown) => {
      console.log(data)
      if (!isPeerData(data)) return
      if (data.type == 'name') {
        connectedDestId.value.set(conn.peer, { conn, name: data.content })
        messages.value.push({
          text: data.content + ' has connected!',
          color: 'success',
          variant: 'tonal',
        })
        if (data.handshakeFirst) {
          const peerData: PeerData = {
            type: 'name',
            content: myName.value,
            handshakeFirst: false,
          }
          conn.send(peerData)
        }
      } else if (data.type == 'message') {
        const peerConn = connectedDestId.value.get(conn.peer)
        const n = peerConn ? peerConn.name : 'anonymous'
        messages.value.push({
          text: n + ' < ' + data.content,
          color: 'info',
          variant: 'tonal',
        })
      }
    }
  }

  function SendMessage () {
    if ([...connectedDestId.value.keys()].includes(sendtoId.value)) {
      console.log('in')
      const peerConn = connectedDestId.value.get(sendtoId.value)
      console.log(peerConn)
      if (peerConn) {
        const peerData: PeerData = {
          type: 'message',
          content: messageToSend.value,
          handshakeFirst: false,
        }
        peerConn.conn.send(peerData)
      }
    }
  }

  peer.on('error', error => {
    console.log(error)
  })

  function registerName () {
    myName.value = inputtedMyName.value

    peer.on('connection', conn => {
      connectedDestId.value.set(conn.peer, { conn, name: '' })
      conn.on('data', receivePeerMessageWrapper(conn))
    })
  }
</script>
