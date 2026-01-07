<script>
import api from '@/api'

export default {
  name: 'Game',

  data() {
    return {
      partie: null,
      erreurs: [],
      websocket: null,
      myUserId: null
    }
  },

  beforeRouteEnter(to, from, next) {
    const id = to.params.id
    api.get(`/api/games/${id}`)
      .then(({ data }) => {
        next(vm => {
          vm.partie = data
          vm.initGame()
        })
      })
      .catch(err => {
        next(vm => {
          vm.erreurs = err.response?.data || ['Erreur lors du chargement de la partie']
        })
      })
  },

  beforeUnmount() {
    if (this.websocket) this.websocket.close()
  },

  methods: {
    initGame() {
      api.get('/api/profile').then(response => {
        this.myUserId = response.data.id
        this.connectWebSocket()
      })
    },

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
        const message = JSON.parse(event.data)
        if (['opponent-join', 'opponent-play', 'opponent-quit'].includes(message.type)) {
          this.rechargerPartie()
        }
      }
    },

    rechargerPartie() {
      api.get(`/api/games/${this.partie.id}`).then(({ data }) => {
        this.partie = data
      })
    },

    async play(rowIndex, colIndex) {
      // Vérification avec le statut numérique (1 = En cours)
      if (this.partie.game_status !== 1) return
      if (this.partie.next_player_id !== this.myUserId) return
      if (this.partie.grid[rowIndex][colIndex] !== null) return

      try {
        const response = await api.patch(`/api/games/${this.partie.id}/play/${rowIndex}/${colIndex}`)
        this.partie = response.data
      } catch (e) {
        if (e.response && e.response.data) {
          alert("Erreur : " + JSON.stringify(e.response.data))
        }
      }
    },

    getCellContent(cellValue) {
      if (cellValue === null) return ''
      // On sécurise l'accès à l'ID du joueur 1
      const p1Id = this.partie.Player1 ? this.partie.Player1.id : (this.partie.player1 ? this.partie.player1.id : null)
      cellValue === p1Id ? 'X' : 'O'
    },

    isMyTurn() {
      // Vérification : Partie en cours (1) ET c'est mon tour
      return this.partie.game_status === 1 && this.partie.next_player_id === this.myUserId
    }
  }
}
</script>

<template>
  <div class="game-view">
    <router-link to="/home" class="back-link">← Retour</router-link>

    <h1>Morpion</h1>

    <div v-if="erreurs.length" class="error-box">
      <ul><li v-for="(e, i) in erreurs" :key="i">{{ e }}</li></ul>
    </div>

    <div v-if="!partie">Chargement...</div>

    <div v-else>
      <div class="info-panel">
        <p><strong>Code partie :</strong> <span class="code">{{ partie.code }}</span></p>

        <div v-if="!partie.opponent" class="waiting">
          <p>En attente d’un adversaire…</p>
        </div>

        <div v-else>
          <div class="players">
            <span :class="{ active: partie.next_player_id === (partie.Player1?.id || partie.player1?.id) }">
              {{ partie.Player1?.name || 'Joueur 1' }} (X)
            </span>
            VS
            <span :class="{ active: partie.next_player_id === partie.opponent.id }">
              {{ partie.opponent.name }} (O)
            </span>
          </div>

          <h3 v-if="partie.game_status === 1">
            {{ isMyTurn() ? "C'est à vous !" : "Au tour de l'adversaire..." }}
          </h3>

          <div v-if="partie.game_status === 2" class="result-box">
            <h2>Partie Terminée !</h2>
            <p v-if="!partie.winner_id">Match Nul ! 🤝</p>
            <p v-else-if="partie.winner_id === myUserId" class="win">🎉 VOUS AVEZ GAGNÉ ! 🎉</p>
            <p v-else class="lose">💀 Vous avez perdu...</p>

            <router-link to="/home">
              <button class="btn-home">Retour au menu principal</button>
            </router-link>
          </div>

          <div v-if="partie.game_status === 1" class="grid">
            <div v-for="(row, rIndex) in partie.grid" :key="rIndex" class="row">
              <div
                v-for="(cell, cIndex) in row"
                :key="cIndex"
                class="cell"
                :class="{
                  'clickable': isMyTurn() && cell === null,
                  'taken': cell !== null
                }"
                @click="play(rIndex, cIndex)"
              >
                {{ getCellContent(cell) }}
              </div>
            </div>
          </div>

        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Le CSS reste identique */
.game-view { max-width: 600px; margin: 0 auto; text-align: center; }
.code { font-family: monospace; font-weight: bold; background: #eee; padding: 2px 6px; }
.players { display: flex; justify-content: space-around; margin: 20px 0; font-size: 1.1em; }
.players span.active { font-weight: bold; color: #42b983; text-decoration: underline; }
.grid { display: inline-flex; flex-direction: column; border: 2px solid #333; margin-top: 20px; }
.row { display: flex; }
.cell { width: 80px; height: 80px; border: 1px solid #ccc; display: flex; align-items: center; justify-content: center; font-size: 2.5em; background-color: #fff; }
.cell.clickable { cursor: pointer; }
.cell.clickable:hover { background-color: #e6f7ff; }
.cell.taken { color: #555; background-color: #f9f9f9; }
.result-box { margin-top: 30px; padding: 20px; border: 2px dashed #ccc; background: #fafafa; }
.win { color: green; font-weight: bold; }
.lose { color: red; font-weight: bold; }
.btn-home { margin-top: 15px; padding: 10px 20px; background-color: #333; color: white; border: none; cursor: pointer; }
</style>
