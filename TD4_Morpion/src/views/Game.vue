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
      gameOwnerId: null // Joueur 1 = X (SOURCE UNIQUE)
    }
  },

  computed: {
    // Grille 3x3 depuis les champs r1c1...r3c3
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
    if (this.websocket) this.websocket.close()
  },

  methods: {
    // 1) Charger la partie
    loadGameData(isInitialLoad = false) {
      const id = this.$route.params.id
      api.get(`/api/games/${id}?t=${Date.now()}`)
        .then(({ data }) => {
          this.partie = data

          // IMPORTANT : source UNIQUE pour X
          if (isInitialLoad) {
            this.gameOwnerId = data.owner.id
            this.identifyUser()
          }
        })
        .catch(err => {
          console.error(err)
          this.erreurs = err.response?.data || ['Erreur de chargement']
        })
    },

    // 2) Identifier l’utilisateur courant
    identifyUser() {
      api.get('/api/profile').then(({ data }) => {
        this.myUserId = data.id
        this.connectWebSocket()
      })
    },

    // 3) WebSocket
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

      this.websocket.onmessage = (event) => {
        const msg = JSON.parse(event.data)
        if (['opponent-join', 'opponent-play', 'opponent-quit'].includes(msg.type)) {
          this.loadGameData(false)
        }
      }
    },

    // 4) Jouer une case
    async play(rowIndex, colIndex) {
      if (this.partie.state !== 1) return
      if (this.partie.next_player_id !== this.myUserId) return
      if (this.grid[rowIndex][colIndex] !== null) return

      const apiRow = rowIndex + 1
      const apiCol = colIndex + 1

      try {
        const { data } = await api.patch(
          `/api/games/${this.partie.id}/play/${apiRow}/${apiCol}`
        )
        this.partie = data
      } catch (e) {
        alert('Erreur : ' + JSON.stringify(e.response?.data || e))
      }
    },

    // 5) Contenu cellule (FIX X/O)
    getCellContent(cellValue) {
      if (cellValue === null) return ''
      return cellValue === this.gameOwnerId ? 'X' : 'O'
    },

    isMyTurn() {
      return this.partie.state === 1 && this.partie.next_player_id === this.myUserId
    }
  }
}
</script>

<template>
  <div class="game-view">
    <router-link to="/home">← Retour</router-link>

    <h1>Morpion</h1>

    <div v-if="erreurs.length">
      <ul>
        <li v-for="(e, i) in erreurs" :key="i">{{ e }}</li>
      </ul>
    </div>

    <div v-if="!partie">Chargement...</div>

    <div v-else>
      <p><strong>Code partie :</strong> {{ partie.code }}</p>

      <div v-if="!partie.opponent">
        <p>En attente d’un adversaire…</p>
      </div>

      <div v-else>
        <p>
          <strong>{{ partie.owner.name }}</strong> (X)
          VS
          <strong>{{ partie.opponent.name }}</strong> (O)
        </p>

        <p v-if="partie.state === 1">
          {{ isMyTurn() ? "C'est à vous !" : "Au tour de l'adversaire..." }}
        </p>

        <div v-if="partie.state === 2">
          <p v-if="!partie.winner_id">Match nul</p>
          <p v-else-if="partie.winner_id === myUserId">🎉 Vous avez gagné</p>
          <p v-else>❌ Vous avez perdu</p>

          <router-link to="/home">
            <button>Retour à l'accueil</button>
          </router-link>
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
.game-view { max-width: 600px; margin: 0 auto; text-align: center; }
.grid { display: inline-flex; flex-direction: column; border: 2px solid #333; margin-top: 20px; }
.row { display: flex; }
.cell {
  width: 80px; height: 80px; border: 1px solid #ccc;
  display: flex; align-items: center; justify-content: center;
  font-size: 2.5em;
}
.cell.clickable { cursor: pointer; }
</style>
