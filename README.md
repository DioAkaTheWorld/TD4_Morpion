## Membres du groupe
- **Burak Ozen**
- **Baptiste Delaborde**

---

## Installation du projet

### Étapes d’installation

1. Installer les dépendances :
```bash
npm install
```

2. Installer Axios :
```bash
npm install axios
```

3. Lancer le serveur de développement :
```bash
npm run dev
```

---

## Lancement
Une fois le serveur lancé, l’application est accessible à l’adresse :
```
http://localhost:5173
```

---

## Remarques
- Chaque joueur doit utiliser une **clé API différente**
Pour cela on a créé des fichiers nommé .env.local avec notre propre clé api à l'interieur

- Contenu du fichier .env.local :
VITE_API_KEY=clé_api

## Problèmes
- Affichage du jeu, le jeu fonctionne mais les 2 joueurs jouent avec le symbole O.
