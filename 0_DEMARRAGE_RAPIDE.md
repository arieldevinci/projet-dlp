# 🚀 Démarrage rapide — UPSILON Helpdesk & SAV
## Voir l'application dans ton navigateur en 5 minutes

---

## Ce dont tu as besoin

| Outil | Version minimale | Vérifier |
|-------|-----------------|---------|
| Node.js | 18+ | `node --version` |
| npm | 9+ | `npm --version` |

> Si Node.js n'est pas installé → télécharger sur **https://nodejs.org** (prendre la version LTS)

---

## Étape 1 — Ouvrir le dossier du projet

```bash
# Aller dans le dossier du projet front (là où se trouve package.json)
cd chemin/vers/le/projet
```

---

## Étape 2 — Installer les dépendances

```bash
npm install
```

> Cette commande télécharge tous les packages nécessaires (React, Tailwind, Recharts...).
> Elle crée un dossier `node_modules/`. À faire **une seule fois**.

---

## Étape 3 — Lancer l'application

```bash
npm run dev
```

> Le terminal affiche quelque chose comme :
> ```
>   VITE v5.4.19  ready in 800ms
>
>   ➜  Local:   http://localhost:5173/
>   ➜  Network: http://192.168.X.X:5173/
> ```

---

## Étape 4 — Ouvrir dans le navigateur

Ouvrir **http://localhost:5173** dans Chrome, Firefox ou Edge.

La page de connexion s'affiche.

---

## Connexion — Comptes de démo disponibles

L'application tourne en mode démo avec des données simulées.
Tu peux te connecter avec n'importe quel de ces comptes :

| Rôle | Email | Mot de passe |
|------|-------|-------------|
| **Client** | amine.belhaj@biat.tn | n'importe quoi |
| **Ingénieur** | karim.belhaj@upsilon.tn | n'importe quoi |
| **Commercial** | rania.zouari@upsilon.tn | n'importe quoi |
| **Super Admin** | sophie.aboud@upsilon.tn | n'importe quoi |

> En mode démo, le mot de passe n'est **pas vérifié**.
> Il suffit que l'email corresponde à un compte existant dans `src/data/seed.ts`.

---

## Commandes utiles

```bash
# Lancer en mode développement (avec rechargement automatique)
npm run dev

# Compiler pour la production
npm run build

# Prévisualiser la version compilée
npm run preview

# Vérifier les erreurs de code
npm run lint

# Lancer les tests
npm run test
```

---

## Si ça ne fonctionne pas

**Erreur `node_modules not found` ou `Cannot find module`**
```bash
# Supprimer et réinstaller les dépendances
rm -rf node_modules
npm install
```

**Port 5173 déjà utilisé**
```bash
# Vite choisit automatiquement le prochain port disponible (5174, 5175...)
# L'URL correcte est affichée dans le terminal
```

**Erreur TypeScript au démarrage**
```bash
# Ces erreurs n'empêchent pas l'app de tourner en mode dev
# Vite compile quand même et affiche l'app
```

---

## Structure visible dans le navigateur

Une fois connecté, voici ce que tu vois selon le rôle :

```
Super Admin  →  /admin/...        Dashboard global, utilisateurs, tickets, stats
Ingénieur    →  /ing/...          Tickets, maintenances, contrats (lecture)
Commercial   →  /co/...           Contrats, clients
Client       →  /mes-tickets/...  Tickets, contrats, maintenances, équipements
```

Tous les rôles ont accès à `/profil` pour modifier leurs informations.

---

## Note importante pour le branchement Django

Actuellement l'application utilise des **données simulées en mémoire**.
Pour la connecter à Django, il faudra :

1. Créer un fichier `.env` à la racine du projet front :
```env
VITE_API_BASE_URL=http://localhost:8000
```

2. Modifier deux fichiers :
   - `src/contexts/AuthContext.tsx` → brancher le vrai login JWT
   - `src/services/api.ts` → remplacer les fonctions mock par des `fetch()`

Les détails complets sont dans **`2_GUIDE_DJANGO.md`**.

