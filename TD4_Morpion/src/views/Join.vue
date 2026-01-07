<template>
  <div class="join-container">
    <h1>Rejoindre une partie</h1>

    <div class="form-group">
      <label for="gameCode">Code de la partie :</label>
      <input
        type="text"
        id="gameCode"
        v-model="gameCode"
        placeholder="Ex: 5F3A2B"
      >
    </div>

    <button @click="joinGame">Rejoindre</button>

    <div v-if="error" class="error-message">
      {{ error }}
    </div>

    <br><br>
    <router-link to="/home" class="back-link">Retour</router-link>
  </div>
</template>

<script>
import api from '../api'

export default {
  name: 'Join',
  data() {
    return {
      gameCode: '',
      error: null
    }
  },
  methods: {
    async joinGame() {
      this.error = null;

      if (!this.gameCode) {
        this.error = "Veuillez saisir un code.";
        return;
      }

      try {
        const response = await api.patch(`/api/games/${this.gameCode}/join`);
        const gameId = response.data.id;
        this.$router.push({ name: 'game', params: { id: gameId } });

      } catch (e) {
        console.error(e);
        if (e.response && e.response.data && e.response.data.message) {
          this.error = e.response.data.message;
        } else {
          this.error = "Impossible de rejoindre la partie. Vérifiez le code.";
        }
      }
    }
  }
}
</script>
