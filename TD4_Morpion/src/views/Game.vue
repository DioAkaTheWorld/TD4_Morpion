<script>
import api from '@/api'

export default {
  name: 'Game',

  data() {
    return {
      partie: null,
      erreurs: [],
      websocket: null,
      myUserId: null,
      gameOwnerId: null // ID du créateur = X (SOURCE UNIQUE)
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
    }
  },

  mounted() {
    this.loadGameData(true)
  },

  beforeUnmount() {
    if (this.websocket) {
      this.websocket.close()
      this.websocket = null
    }
  },

  methods: {
    /* =======================
       CHARGEMENT DE LA PARTIE
       ======================= */
    async loadGameData(isInitialLoad = false) {
      try {
        const id = this.$route.params.id
        const { data } = await api.get(`/api/games/${id}?t=${Date.now()}`)
        this.partie = data

        // SOURCE UNIQUE ET STABLE POUR X
        if (isInitialLoad) {
          this.gameOwnerId = data.owner.id
          await this.identifyUser()
        }
      } catch (e) {
        console.error(e)
        this.erreurs = ['Erreur de chargement de la partie']
      }
    },

    /* =======================
       IDENTIFICATION UTILISATEUR
       ======================= */
    async identifyUser() {
      const { data } = await api.get('/api/profile')
      this.myUserId = data.id
      this.connectWebSocket()
    },

    /* =======================
       WEBSOCKET (SYNCHRO)
       ======================= */
    connectWebSocket() {
      if (this.websocket) return

      this.websocket = new WebSocket('wss://morpion-api.edu.netlor.fr/websockets')

      this.websocket.onopen = () => {
        this.websocket.send(JSON.stringify({
          action: 'connect',
          game_id: this.partie.id,
          player_id: this.myUserId
        }))
      }

      this.websocket.onmessage = async (event) => {
        const msg = JSON.parse(event.data)

        if (['opponent-join', 'opponent-play', 'opponent-quit'].includes(msg.type)) {
          await this.loadGameData(false)
        }
      }
    },

    /* =======================
       JOUER UN COUP
       ======================= */
    async play(rowIndex, colIndex) {
      if (this.partie.state !== 1) return
      if (this.partie.next_player_id !== this.myUserId) return
      if (this.grid[rowIndex][colIndex] !== null) return

      const apiRow = rowIndex + 1
      const apiCol = colIndex + 1

      try {
        // PATCH = action uniquement
        await api.patch(`/api/games/${this.partie.id}/play/${apiRow}/${apiCol}`)

        // GET = vérité (OBLIGATOIRE)
        await this.loadGameData(false)
      } catch (e) {
        alert('Erreur lors du coup')
      }
    },

    /* =======================
       AFFICHAGE X / O (FIX)
       ======================= */
    getCellContent(cellValue) {
      if (cellValue === null) return ''
      return cellValue === this.gameOwnerId ? 'X' : 'O'
    },

    isMyTurn() {
      return this.partie.state === 1 &&
             this.partie.next_player_id === this.myUserId
    }
  }
}
</script>

<template>
  <div class="game-view">
    <router-link to="/home">← Retour</router-link>

    <h1>Morpion</h1>

    <div v-if="erreurs.length">
      <p v-for="(e, i) in erreurs" :key="i">{{ e }}</p>
    </div>

    <div v-if="!partie">Chargement...</div>

    <div v-else>
      <p><strong>Code :</strong> {{ partie.code }}</p>

      <div v-if="!partie.opponent">
        <p>En attente d’un adversaire…</p>
      </div>

      <div v-else>
        <p>
          {{ partie.owner.name }} (X)
          VS
          {{ partie.opponent.name }} (O)
        </p>

        <p v-if="partie.state === 1">
          {{ isMyTurn() ? "C'est à vous !" : "Au tour de l'adversaire..." }}
        </p>

        <div v-if="partie.state === 2">
          <p v-if="!partie.winner_id">Match nul</p>
          <p v-else-if="partie.winner_id === myUserId">🎉 Vous avez gagné</p>
          <p v-else>❌ Vous avez perdu</p>
        </div>

        <div v-if="partie.state !== 2" class="grid">
          <div v-for="(row, r) in grid" :key="r" class="row">
            <div
              v-for="(cell, c) in row"
              :key="c"
              class="cell"
              :class="{ clickable: isMyTurn() && cell === null }"
              @click="play(r, c)"
            >
              {{ getCellContent(cell) }}
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.game-view { max-width: 600px; margin: auto; text-align: center; }
.grid { display: inline-flex; flex-direction: column; border: 2px solid #333; margin-top: 20px; }
.row { display: flex; }
.cell {
  width: 80px;
  height: 80px;
  border: 1px solid #ccc;
  font-size: 2.5em;
  display: flex;
  align-items: center;
  justify-content: center;
}
.cell.clickable { cursor: pointer; }
</style>
