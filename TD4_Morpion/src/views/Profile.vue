<script>
import api from '@/api'

export default {
  name: 'Profile',

  data() {
    return {
      user: null,
      erreurs: [],
      succes: false
    }
  },

  // demandé par l’énoncé
  beforeRouteEnter(to, from, next) {
    api.get('/api/profile').then(({ data }) => {
      next(vm => {
        vm.user = data
      })
    })
  },

  methods: {
    enregistrer() {
      this.erreurs = []
      this.succes = false

      api.put('/api/profile', this.user)
        .then(({ data }) => {
          this.user = data
          this.succes = true
        })
        .catch(err => {
          this.erreurs = err.response?.data || ['Erreur lors de la sauvegarde']
        })
    }
  }
}
</script>

<template>
  <div>
    <router-link to="/home">← Retour à l’accueil</router-link>

    <h1>Mon profil</h1>

    <!-- Message de succès -->
    <p v-if="succes">
      Profil enregistré avec succès.
    </p>

    <!-- Messages d’erreur -->
    <div v-if="erreurs.length">
      <p><strong>Erreurs :</strong></p>
      <ul>
        <li v-for="(e, i) in erreurs" :key="i">{{ e }}</li>
      </ul>
    </div>

    <div v-if="!user">
      Chargement...
    </div>

    <form v-else @submit.prevent="enregistrer">
      <div>
        <label>
          Nom
          <input type="text" v-model="user.name" />
        </label>
      </div>

      <div>
        <label>
          Email
          <input type="email" v-model="user.email" />
        </label>
      </div>

      <button type="submit">
        Enregistrer
      </button>
    </form>
  </div>
</template>
