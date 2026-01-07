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

    <router-link to="/home" class="back-link">Retour</router-link>
  </div>
</template>

<script>
import api from '../api'

export default {
  name: 'Join',
  data() {
    return {
      gameCode: '', // La propriété bindée sur le champ
      error: null
    }
  },
  methods: {
    async joinGame() {
      // On réinitialise l'erreur
      this.error = null;

      // Vérification basique
      if (!this.gameCode) {
        this.error = "Veuillez saisir un code.";
        return;
      }

      try {
        // Appel Ajax sur la route PATCH /api/games/:code/join
        const response = await api.patch(`/api/games/${this.gameCode}/join`);

        // Si tout se passe bien, l'API retourne les infos, dont l'ID.
        // On redirige vers la vue de la partie.
        const gameId = response.data.id;
        this.$router.push({ name: 'game', params: { id: gameId } });

      } catch (e) {
        // Gestion des erreurs retournées par le serveur
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

<style scoped>
.error-message {
  color: red;
  margin-top: 10px;
}
.form-group {
  margin-bottom: 15px;
}
input {
  padding: 5px;
  font-size: 1.1em;
}
</style>
