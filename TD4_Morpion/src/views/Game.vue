<script>
import api from '@/api'

export default {
  name: 'Game',

  data() {
    return {
      partie: null,      // L'objet partie qui change à chaque coup
      erreurs: [],
      websocket: null,
      myUserId: null,    // Mon ID
      gameOwnerId: null  // L'ID du créateur (Joueur 1 = X), stocké une bonne fois pour toutes
    }
  },

  computed: {
    // On transforme les données "plates" de l'API (r1c1...) en tableau 2D pour l'affichage
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
    // 1. Au chargement de la page, on récupère la partie
    this.loadGameData(true)
  },

  beforeUnmount() {
    if (this.websocket) this.websocket.close()
  },

  methods: {
    // Charge les données de la partie
    loadGameData(isInitialLoad = false) {
      const id = this.$route.params.id
      // Le paramètre t=Date.now() force le navigateur à ne pas utiliser le cache
      api.get(`/api/games/${id}?t=${Date.now()}`)
        .then(({ data }) => {
          this.partie = data

          // Si c'est le tout premier chargement, on sauvegarde QUI est le joueur 1 (X)
          if (isInitialLoad) {
            // On cherche l'ID partout où il peut être (owner_id ou owner.id)
            this.gameOwnerId = data.owner_id || (data.owner ? data.owner.id : null)
            console.log("Joueur 1 (X) ID fixé à :", this.gameOwnerId)

            // Et on lance l'identification utilisateur + websocket
            this.identifyUser()
          }
        })
        .catch(err => {
          console.error(err)
          this.erreurs = err.response?.data || ['Erreur de chargement']
        })
    },

    // Récupère mon propre ID
    identifyUser() {
      api.get('/api/profile').then(response => {
        this.myUserId = response.data.id
        console.log("Je suis :", this.myUserId)
        this.connectWebSocket()
      })
    },

    // Connexion au temps réel
    connectWebSocket() {
      if (this.websocket) return

      this.websocket = new WebSocket('wss://morpion-api.edu.netlor.fr/websockets')

      this.websocket.onopen = () => {
        console.log("WebSocket connecté !")
        this.websocket.send(JSON.stringify({
          action: 'connect',
          game_id: this.partie.id,
          player_id: this.myUserId
        }))
      }

      this.websocket.onmessage = (event) => {
        const message = JSON.parse(event.data)
        console.log("Notification reçue :", message)

        // Si quelqu'un joue, rejoint ou quitte, on recharge la partie
        if (['opponent-join', 'opponent-play', 'opponent-quit'].includes(message.type)) {
          this.loadGameData(false) // false car ce n'est pas le chargement initial
        }
      }
    },

    // Action de jouer
    async play(rowIndex, colIndex) {
      if (this.partie.state !== 1) return // Pas en cours
      if (this.partie.next_player_id !== this.myUserId) return // Pas mon tour
      if (this.grid[rowIndex][colIndex] !== null) return // Case prise

      // L'API compte les lignes/colonnes à partir de 1
      const apiRow = rowIndex + 1
      const apiCol = colIndex + 1

      try {
        const response = await api.patch(`/api/games/${this.partie.id}/play/${apiRow}/${apiCol}`)
        // On met à jour la partie immédiatement
        this.partie = response.data
      } catch (e) {
        if (e.response && e.response.data) alert("Erreur : " + JSON.stringify(e.response.data))
      }
    },

    // Détermine si on affiche X ou O
    getCellContent(cellValue) {
      if (cellValue === null) return ''
      // Si l'ID dans la case est celui du gameOwnerId sauvegardé au début => X
      // Sinon => O
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
              {{ (partie.owner && partie.owner.name) ? partie.owner.name : 'Joueur 1' }} (X)
            </span>
            VS
            <span :class="{ active: partie.next_player_id === partie.opponent.id }">
              {{ (partie.opponent && partie.opponent.name) ? partie.opponent.name : 'Joueur 2' }} (O)
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
