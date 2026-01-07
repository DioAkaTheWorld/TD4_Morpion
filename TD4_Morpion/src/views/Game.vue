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
      polling: false,
      gameOwnerId: null
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
    // On charge la partie et on initialise les IDs fixes
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

      // AU DÉBUT SEULEMENT : On sauvegarde qui est le créateur (X)
      // On regarde data.owner.id OU data.owner_id pour être sûr de l'avoir
      if (initial) {
        this.gameOwnerId = data.owner ? data.owner.id : data.owner_id
        this.connectWebSocket()
      }
    },

    /* 🔥 LONG POLLING SÉCURISÉ 🔥 */
    async startLongPolling() {
      this.polling = true
      while (this.polling) {
        // On attend 1 seconde entre chaque appel pour ne pas tuer le navigateur
        await new Promise(resolve => setTimeout(resolve, 1000))
        if (!this.polling) break
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
      // Pas besoin de fetchGame ici, le polling le fera tout seul
    },

    /* 🔥 LOGIQUE X / O RÉPARÉE 🔥 */
    getCellContent(cellValue) {
      if (cellValue === null) return ''

      // On compare avec la variable stable gameOwnerId
      // Si la valeur dans la case est l'ID du créateur => X
      // Sinon (c'est l'adversaire) => O
      return cellValue === this.gameOwnerId ? 'X' : 'O'
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
          {{ partie.owner ? partie.owner.name : 'Joueur 1' }} (X)
          VS
          {{ partie.opponent ? partie.opponent.name : 'Joueur 2' }} (O)
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
          <p v-else-if="partie.winner_id === myUserId" class="error-message" style="color: green;">🎉 Gagné</p>
          <p v-else class="error-message">❌ Perdu</p>
        </div>
      </div>
    </div>
  </div>
</template>
