<script>
import api from '@/api'

export default {
  name: 'Game',

  data() {
    return {
      partie: null,
      erreurs: [],
      websocket: null
    }
  },

  beforeRouteEnter(to, from, next) {
    const id = to.params.id

    api.get(`/api/games/${id}`)
      .then(({ data }) => {
        next(vm => {
          vm.partie = data
          vm.attendreMouvementAdversaire()
        })
      })
      .catch(err => {
        next(vm => {
          vm.erreurs = err.response?.data || ['Erreur lors du chargement de la partie']
        })
      })
  },

  beforeUnmount() {
    if (this.websocket) {
      this.websocket.close()
    }
  },

  methods: {
    attendreMouvementAdversaire() {
      this.websocket = new WebSocket(
        'wss://morpion-api.edu.netlor.fr/websockets'
      )

      this.websocket.onopen = () => {
        const game_id = this.partie.id
        const player_id = this.partie.player_id

        this.websocket.send(JSON.stringify({
          action: 'connect',
          game_id,
          player_id
        }))
      }

      this.websocket.onmessage = (event) => {
        const message = JSON.parse(event.data)

        if (
          message.type === 'opponent-join' ||
          message.type === 'opponent-play' ||
          message.type === 'opponent-quit'
        ) {
          this.rechargerPartie()
        }
      }
    },

    rechargerPartie() {
      const id = this.$route.params.id
      api.get(`/api/games/${id}`)
        .then(({ data }) => {
          this.partie = data
        })
    }
  }
}
</script>

<template>
  <div>
    <router-link to="/home">← Retour</router-link>

    <h1>Partie</h1>

    <div v-if="erreurs.length">
      <ul>
        <li v-for="(e, i) in erreurs" :key="i">{{ e }}</li>
      </ul>
    </div>

    <div v-if="!partie">
      Chargement...
    </div>

    <div v-else>
      <p><strong>Code de la partie :</strong> {{ partie.code }}</p>

      <p v-if="!partie.opponent">
        En attente d’un adversaire…
      </p>

      <div v-else>
        <p>Adversaire connecté.</p>

        <p>
          Prochain joueur :
          {{ partie.next_player_id === partie.player_id ? 'Vous' : 'Adversaire' }}
        </p>

        <p>Grille de jeu affichée ici</p>
      </div>
    </div>
  </div>
</template>
