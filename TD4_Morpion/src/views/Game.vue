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
      gameOwnerId: null // VARIABLE IMPORTANTE : On stocke ici l'ID du créateur définitivement
    }
  },

  computed: {
    // Reconstruction de la grille (3x3) à partir des données plates (r1c1...)
    grid() {
      if (!this.partie) return []
      return [
        [this.partie.r1c1, this.partie.r1c2, this.partie.r1c3],
        [this.partie.r2c1, this.partie.r2c2, this.partie.r2c3],
        [this.partie.r3c1, this.partie.r3c2, this.partie.r3c3]
      ]
    }
  },

  beforeRouteEnter(to, from, next) {
    const id = to.params.id
    // Ajout d'un timestamp pour éviter le cache dès le début
    api.get(`/api/games/${id}?t=${Date.now()}`)
      .then(({ data }) => {
        next(vm => {
          vm.partie = data
          // On sauvegarde l'ID du créateur tout de suite !
          // On regarde 'owner_id' ou 'owner.id' selon ce qui est dispo
          vm.gameOwnerId = data.owner_id || (data.owner ? data.owner.id : null)

          vm.initGame()
        })
      })
      .catch(err => {
        next(vm => {
          vm.erreurs = err.response?.data || ['Erreur chargement partie']
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
      }).catch(err => {
        console.error("Erreur profil", err)
      })
    },

    connectWebSocket() {
      if (this.websocket) return

      this.websocket = new WebSocket('wss://morpion-api.edu.netlor.fr/websockets')

      this.websocket.onopen = () => {
        console.log("WS Connecté")
        this.websocket.send(JSON.stringify({
          action: 'connect',
          game_id: this.partie.id,
          player_id: this.myUserId
        }))
      }

      this.websocket.onmessage = (event) => {
        const message = JSON.parse(event.data)
        console.log("WS Message:", message) // Regarde ta console (F12) si ça ne marche pas

        // Si n'importe quel événement de jeu arrive, on recharge
        if (['opponent-join', 'opponent-play', 'opponent-quit'].includes(message.type)) {
          this.rechargerPartie()
        }
      }
    },

    rechargerPartie() {
      // ASTUCE ANTI-CACHE : On ajoute Date.now() pour forcer une vraie requête au serveur
      api.get(`/api/games/${this.partie.id}?t=${Date.now()}`).then(({ data }) => {
        this.partie = data
        // Note: on ne touche PAS à gameOwnerId ici, pour garder la référence stable
      })
    },

    async play(rowIndex, colIndex) {
      if (this.partie.state !== 1) return
      if (this.partie.next_player_id !== this.myUserId) return
      if (this.grid[rowIndex][colIndex] !== null) return

      // +1 car l'API compte à partir de 1
      const apiRow = rowIndex + 1
      const apiCol = colIndex + 1

      try {
        const response = await api.patch(`/api/games/${this.partie.id}/play/${apiRow}/${apiCol}`)
        this.partie = response.data
      } catch (e) {
        console.error(e)
        if (e.response && e.response.data) alert("Erreur: " + JSON.stringify(e.response.data))
      }
    },

    getCellContent(cellValue) {
      if (cellValue === null) return ''

      // LOGIQUE RÉPARÉE :
      // On compare la valeur de la case avec notre variable stable gameOwnerId
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
            <span :class="{ active: partie.next_player_id === gameOwnerId }">
              {{ partie.owner ? partie.owner.name : 'Joueur 1' }} (X)
            </span>
            VS
            <span :class="{ active: partie.next_player_id === partie.opponent.id }">
              {{ partie.opponent ? partie.opponent.name : 'Joueur 2' }} (O)
            </span>
          </div>

          <h3 v-if="partie.state === 1">
            {{ isMyTurn() ? "C'est à vous !" : "Au tour de l'adversaire..." }}
          </h3>

          <div v-if="partie.state === 2" class="result-box">
            <h2>Partie Terminée !</h2>
            <p v-if="!partie.winner_id">Match Nul ! 🤝</p>
            <p v-else-if="partie.winner_id === myUserId" class="win">🎉 VOUS AVEZ GAGNÉ ! 🎉</p>
            <p v-else class="lose">💀 Vous avez perdu...</p>

            <router-link to="/home">
              <button class="btn-home">Retour au menu principal</button>
            </router-link>
          </div>

          <div v-if="partie.state !== 2" class="grid">
            <div v-for="(row, rIndex) in grid" :key="rIndex" class="row">
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
.game-view { max-width: 600px; margin: 0 auto; text-align: center; }
.code { font-family: monospace; font-weight: bold; background: #eee; padding: 2px 6px; }
.players { display: flex; justify-content: space-around; margin: 20px 0; font-size: 1.1em; }
.players span.active { font-weight: bold; color: #42b983; text-decoration: underline; }
.grid { display: inline-flex; flex-direction: column; border: 2px solid #333; margin-top: 20px; }
.row { display: flex; }
.cell { width: 80px; height: 80px; border: 1px solid #ccc; display: flex; align-items: center; justify-content: center; font-size: 2.5em; background-color: #fff; cursor: default; }
.cell.clickable { cursor: pointer; }
.cell.clickable:hover { background-color: #e6f7ff; }
.cell.taken { color: #555; background-color: #f9f9f9; }
.result-box { margin-top: 30px; padding: 20px; border: 2px dashed #ccc; background: #fafafa; }
.win { color: green; font-weight: bold; }
.lose { color: red; font-weight: bold; }
.btn-home { margin-top: 15px; padding: 10px 20px; background-color: #333; color: white; border: none; cursor: pointer; }
.waiting { margin-top: 20px; font-style: italic; color: #666; }
</style>
