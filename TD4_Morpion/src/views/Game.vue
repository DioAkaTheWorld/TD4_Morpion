<script>
import api from '@/api'

export default {
  name: 'Game',

  data() {
    return {
      partie: null,
      websocket: null,
      erreurs: [],
      myUserId: null,
      polling: false
    }
  },

  computed: {
    grid() {
      if (!this.partie) return []
      return [
        [this.partie.r1c1, this.partie.r1c2, this.partie.r1c3],
        [this.partie.r2c1, this.partie.r2c2, this.partie.r2c3],
        [this.partie.r3c1, this.partie.r3c2, this.partie.r3c3]
      ]
    },

    isMyTurn() {
      return (
        this.partie &&
        this.partie.state === 1 &&
        this.partie.next_player_id === this.myUserId
      )
    }
  },

  async mounted() {
    await this.identifyUser()
    await this.fetchGame(true)
    this.startLongPolling()
  },

  beforeUnmount() {
    this.polling = false
    if (this.websocket) this.websocket.close()
  },

  methods: {
    async identifyUser() {
      const { data } = await api.get('/api/profile')
      this.myUserId = data.id
    },

    async fetchGame(initial = false) {
      const id = this.$route.params.id
      const { data } = await api.get(`/api/games/${id}`)
      this.partie = data

      if (initial) this.connectWebSocket()
    },

    /* 🔥 LONG POLLING = CLÉ DU COURS 🔥 */
    async startLongPolling() {
      this.polling = true
      while (this.polling) {
        await this.fetchGame(false)
      }
    },

    connectWebSocket() {
      this.websocket = new WebSocket('wss://morpion-api.edu.netlor.fr/websockets')

      this.websocket.onopen = () => {
        this.websocket.send(JSON.stringify({
          action: 'connect',
          game_id: this.partie.id,
          player_id: this.myUserId
        }))
      }
    },

    async play(row, col) {
      if (!this.isMyTurn) return
      if (this.grid[row][col] !== null) return

      await api.patch(
        `/api/games/${this.partie.id}/play/${row + 1}/${col + 1}`
      )
    },

    /* 🔥 LOGIQUE X / O CORRECTE 🔥 */
    getCellContent(cellValue) {
      if (cellValue === null) return ''
      if (cellValue === this.partie.owner.id) return 'X'
      if (this.partie.opponent && cellValue === this.partie.opponent.id) return 'O'
      return ''
    }
  }
}
</script>

<template>
  <div class="game-view">
    <router-link to="/home">← Retour</router-link>

    <h1>Morpion</h1>

    <div v-if="!partie">Chargement…</div>

    <div v-else>
      <p><strong>Code :</strong> {{ partie.code }}</p>

      <p v-if="!partie.opponent">En attente d’un adversaire…</p>

      <div v-else>
        <p>
          {{ partie.owner.name }} (X)
          VS
          {{ partie.opponent.name }} (O)
        </p>

        <p>
          {{ isMyTurn ? "C’est à vous !" : "Au tour de l’adversaire…" }}
        </p>

        <div class="grid">
          <div v-for="(row, r) in grid" :key="r" class="row">
            <div
              v-for="(cell, c) in row"
              :key="c"
              class="cell"
              @click="play(r, c)"
            >
              {{ getCellContent(cell) }}
            </div>
          </div>
        </div>

        <div v-if="partie.state === 2">
          <p v-if="!partie.winner_id">Match nul</p>
          <p v-else-if="partie.winner_id === myUserId">🎉 Gagné</p>
          <p v-else>❌ Perdu</p>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.grid { display: inline-flex; flex-direction: column; border: 2px solid #333; }
.row { display: flex; }
.cell {
  width: 80px;
  height: 80px;
  border: 1px solid #ccc;
  font-size: 2.5em;
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>
