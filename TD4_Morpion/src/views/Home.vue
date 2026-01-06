<script>
import api from '@/api'

export default {
  name: 'Home',

  methods: {
    createGame() {
      api.post('/api/games')
        .then(({ data }) => {
          const id = data.id
          this.$router.push({ name: 'game', params: { id } })
        })
    }
  }
}
</script>

<template>
  <div>
    <h1>Page d’accueil</h1>

    <ul>
      <li>
        <router-link to="/profile">
          Mon profil
        </router-link>
      </li>

      <li>
        <button @click="createGame">
          Nouvelle partie
        </button>
      </li>

      <li>
        <router-link to="/join">
          Rejoindre une partie
        </router-link>
      </li>
    </ul>
  </div>
</template>
