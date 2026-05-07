# 📐 Architecture du code Front — UPSILON Helpdesk & SAV
## Ce qu'il faut comprendre avant de toucher quoi que ce soit

---

## 🎯 En une phrase

Le front React est **100% terminé**. Il tourne avec des données simulées en mémoire.
Ton travail = **remplacer ces données simulées par de vrais appels à ton API Django**.
**Un seul fichier à modifier : `src/services/api.ts`**

---

## 🗂️ Structure du projet — fichier par fichier

```
src/
│
├── main.tsx                      ← Point d'entrée. Monte App.tsx dans le DOM. Ne pas toucher.
├── App.tsx                       ← Routeur + tous les providers. Contient TOUTES les routes.
│
├── types/
│   └── index.ts                  ⭐ LIS CE FICHIER EN PREMIER
│                                    Contient tous les types TypeScript = tes modèles Django
│
├── services/
│   └── api.ts                    ⭐ LE SEUL FICHIER QUE TU MODIFIES
│                                    Toutes les fonctions mock → remplace par fetch() vers Django
│
├── contexts/
│   └── AuthContext.tsx           → Gère l'utilisateur connecté. À modifier pour brancher JWT.
│
├── data/
│   └── seed.ts                   → Données de démo factices. Inutile une fois Django branché.
│
├── lib/
│   ├── format.ts                 → Fonctions utilitaires : formatage dates, calcul SLA
│   └── utils.ts                  → Helper CSS uniquement (cn()). Ne pas toucher.
│
├── components/
│   ├── auth/
│   │   └── ProtectedRoute.tsx    → Bloque l'accès aux pages selon le rôle
│   ├── layout/
│   │   ├── AppLayout.tsx         → Mise en page principale : sidebar + zone de contenu
│   │   ├── AppHeader.tsx         → En-tête : bouton "Nouveau ticket" conditionnel selon rôle
│   │   └── AppSidebar.tsx        → Menu latéral : chaque rôle voit ses propres liens
│   ├── dashboard/
│   │   └── KpiCard.tsx           → Carte KPI réutilisable (nombre de tickets, etc.)
│   ├── tickets/
│   │   └── StatusBadge.tsx       → Badges colorés statut/priorité des tickets
│   └── ui/                       → Composants shadcn/ui (Button, Dialog, Input...) NE PAS TOUCHER
│
└── pages/
    ├── Login.tsx                 → Page de connexion (email + mot de passe)
    ├── DashboardRouter.tsx       → Aiguille vers le bon dashboard selon le rôle connecté
    ├── NotFound.tsx              → Page 404
    │
    ├── client/                   → Pages du rôle "client"
    │   ├── ClientDashboard.tsx   → Tableau de bord client
    │   ├── ClientTickets.tsx     → Liste tickets client
    │   ├── NewTicket.tsx         → Formulaire création ticket client
    │   ├── ClientContracts.tsx   → Contrats du client (lecture seule)
    │   ├── ClientMaintenances.tsx→ Maintenances du client (lecture seule + PDF)
    │   └── ClientEquipments.tsx  → Équipements du client (lecture seule)
    │
    ├── engineer/                 → Pages du rôle "engineer"
    │   ├── EngineerDashboard.tsx → Tableau de bord ingénieur
    │   ├── EngineerTickets.tsx   → Liste tickets ingénieur
    │   ├── NewEngineerTicket.tsx → Formulaire création ticket ingénieur
    │   └── EngineerMaintenance.tsx → Planification + gestion maintenances
    │
    ├── commercial/               → Pages du rôle "commercial"
    │   ├── CommercialDashboard.tsx → Tableau de bord commercial
    │   ├── CommercialContracts.tsx → Gestion complète des contrats ⚠️ PARTAGÉ (voir bas)
    │   └── CommercialClients.tsx   → Création et liste des clients
    │
    ├── admin/                    → Pages du rôle "admin"
    │   ├── AdminDashboard.tsx    → Tableau de bord super admin
    │   ├── AdminTickets.tsx      → Attribution tickets aux ingénieurs
    │   ├── AdminUsers.tsx        → Création/suppression de tous les comptes
    │   ├── AdminEscalations.tsx  → Gestion des escalades
    │   ├── AdminAssignments.tsx  → Attribution clients aux ingénieurs
    │   └── AdminStatistics.tsx   → Statistiques de performance par ingénieur
    │
    └── shared/                   → Pages accessibles à plusieurs rôles
        ├── TicketDetail.tsx      → Détail d'un ticket ⚠️ PARTAGÉ (voir bas)
        └── ProfilePage.tsx       → Profil utilisateur (tous les rôles)
```

---

## ⚠️ Pages partagées entre plusieurs rôles — Important

Trois pages sont utilisées par plusieurs rôles avec des comportements différents.
Elles s'adaptent grâce à des **flags calculés depuis le rôle de l'utilisateur**.

### 1. `CommercialContracts.tsx`
Utilisée par : commercial (écriture), engineer (lecture), admin (lecture)
```typescript
const canEdit = user.role === "commercial";
// Si canEdit = false → les boutons Nouveau/Renouveler/Modifier/Supprimer sont masqués
```

### 2. `EngineerMaintenance.tsx`
Utilisée par : engineer (écriture), admin (lecture)
```typescript
const canPlan = user.role === "engineer";
// Si canPlan = false → les boutons Planifier et Uploader rapport sont masqués
```

### 3. `TicketDetail.tsx`
Utilisée par : client, engineer, admin — chacun voit des actions différentes
```typescript
const isEngineer = user.role === "engineer";
const isAdmin    = user.role === "admin";
const isClient   = user.role === "client";

// Engineer → peut répondre, escalader, clôturer, mettre en attente
// Admin    → peut UNIQUEMENT attribuer à un ingénieur + lever l'escalade
// Client   → peut répondre + confirmer la fermeture
```

---

## 🔄 Flux de démarrage de l'application

```
Navigateur charge index.html
    └── main.tsx                    Monte l'app React
          └── App.tsx
                ├── QueryClientProvider   (gestion requêtes async)
                ├── BrowserRouter         (navigation SPA)
                └── AuthProvider          (contexte utilisateur connecté)
                      └── Routes
                            ├── /login    → Login.tsx (accessible sans connexion)
                            └── /*        → ProtectedRoute
                                              └── AppLayout (sidebar + en-tête)
                                                    └── Page selon l'URL
```

---

## 🔐 Authentification — AuthContext.tsx

### Comment ça marche actuellement (mock)
```typescript
// L'utilisateur est cherché dans seed.ts par son email
// Le mot de passe n'est PAS vérifié en mode mock
const login = async (email, password) => {
  const found = seedUsers.find(u => u.email === email);
  // Stocké dans localStorage pour persister entre les rechargements
  localStorage.setItem('upsilon.auth.user', JSON.stringify(found));
  return found; // → objet User avec le rôle
}
```

### Ce que tu dois faire pour brancher Django
Remplacer le corps de la fonction `login` dans `AuthContext.tsx` :
```typescript
const API_URL = import.meta.env.VITE_API_BASE_URL; // http://localhost:8000

const login = async (email: string, password: string): Promise<User> => {
  const res = await fetch(`${API_URL}/api/auth/login/`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });
  if (!res.ok) throw new Error('Identifiants invalides');
  const data = await res.json();
  // data = { access: "eyJ...", refresh: "eyJ...", user: { ...User } }
  localStorage.setItem('upsilon_token', data.access);
  localStorage.setItem('upsilon_refresh', data.refresh);
  return data.user; // L'objet user contient le rôle → tout le reste s'adapte automatiquement
}
```

---

## 🛡️ Protection des routes — ProtectedRoute.tsx

```typescript
// Chaque page dans App.tsx est protégée :
<ProtectedRoute roles={["engineer"]}>
  <EngineerDashboard />
</ProtectedRoute>

// Logique interne :
// 1. Pas connecté          → redirect /login
// 2. Rôle non autorisé    → redirect /
// 3. Rôle autorisé        → affiche la page
```

**Tableau complet des routes et leurs rôles :**

| URL | Rôles autorisés | Page |
|-----|----------------|------|
| `/login` | Public | Login |
| `/` | Tous | Dashboard selon rôle |
| `/profil` | Tous | ProfilePage |
| `/mes-tickets` | client | ClientTickets |
| `/mes-tickets/nouveau` | client | NewTicket |
| `/mes-tickets/:id` | client, engineer, admin | TicketDetail |
| `/mes-contrats` | client | ClientContracts |
| `/mes-maintenances` | client | ClientMaintenances |
| `/mes-equipements` | client | ClientEquipments |
| `/ing/tickets` | engineer | EngineerTickets |
| `/ing/tickets/nouveau` | engineer | NewEngineerTicket |
| `/ing/maintenance` | engineer | EngineerMaintenance |
| `/ing/contrats` | engineer | CommercialContracts (lecture) |
| `/co/contrats` | commercial | CommercialContracts (écriture) |
| `/co/renouvellements` | commercial | CommercialContracts (écriture) |
| `/co/clients` | commercial, admin | CommercialClients |
| `/admin/tickets` | admin | AdminTickets |
| `/admin/escalades` | admin | AdminEscalations |
| `/admin/maintenance` | admin | EngineerMaintenance (lecture) |
| `/admin/attributions` | admin | AdminAssignments |
| `/admin/contrats` | admin | CommercialContracts (lecture) |
| `/admin/utilisateurs` | admin | AdminUsers |
| `/admin/statistiques` | admin | AdminStatistics |

---

## 📡 api.ts — Le seul fichier à modifier

### Principe
Chaque fonction mock retourne des données depuis la mémoire.
Tu les remplaces par des `fetch()` vers Django.
**Les signatures (paramètres et types de retour) ne changent pas.**
Donc aucune page React ne casse quand tu fais le remplacement.

### Les 5 services et leurs méthodes

```typescript
// USERS
usersService.list()                          → GET  /api/users/
usersService.byRole(role)                    → GET  /api/users/?role=X
usersService.byId(id)                        → GET  /api/users/:id/
usersService.create(data)                    → POST /api/users/
usersService.remove(id)                      → DEL  /api/users/:id/
usersService.update(id, patch)               → PATCH /api/users/:id/
usersService.changePassword(id, old, new)    → POST /api/auth/change-password/

// EQUIPMENTS
equipmentsService.list()                     → GET  /api/equipments/
equipmentsService.byClient(clientId)         → GET  /api/equipments/?client_id=X

// CONTRACTS
contractsService.list()                      → GET  /api/contracts/
contractsService.byClient(clientId)          → GET  /api/contracts/?client_id=X
contractsService.byEngineer(engineerId)      → GET  /api/contracts/?engineer_id=X
contractsService.create(data)                → POST /api/contracts/
contractsService.renew(id, start, end)       → PATCH /api/contracts/:id/renew/
contractsService.updateScope(id, patch)      → PATCH /api/contracts/:id/scope/
contractsService.remove(id)                  → DEL  /api/contracts/:id/

// TICKETS
ticketsService.list()                        → GET  /api/tickets/
ticketsService.byClient(clientId)            → GET  /api/tickets/?client_id=X
ticketsService.byEngineer(engineerId)        → GET  /api/tickets/?engineer_id=X
ticketsService.byId(id)                      → GET  /api/tickets/:id/
ticketsService.create(data)                  → POST /api/tickets/
ticketsService.update(id, patch)             → PATCH /api/tickets/:id/
ticketsService.addMessage(id, message)       → POST /api/tickets/:id/messages/

// MAINTENANCES
maintenancesService.list()                   → GET  /api/maintenances/
maintenancesService.byClient(clientId)       → GET  /api/maintenances/?client_id=X
maintenancesService.byEngineer(engineerId)   → GET  /api/maintenances/?engineer_id=X
maintenancesService.create(data)             → POST /api/maintenances/
maintenancesService.update(id, patch)        → PATCH /api/maintenances/:id/
maintenancesService.uploadReport(id, file)   → POST /api/maintenances/:id/upload-report/
```

### Exemple concret — avant / après

**Avant (mock) :**
```typescript
async list(): Promise<Ticket[]> {
  await delay();
  return [..._tickets]; // données en mémoire
}
```

**Après (Django branché) :**
```typescript
async list(): Promise<Ticket[]> {
  const token = localStorage.getItem('upsilon_token');
  const res = await fetch(`${API_URL}/api/tickets/`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return res.json();
}
```

---

## 📦 types/index.ts = Tes modèles Django

**Attention : le front utilise camelCase, Django retourne snake_case.**
Il faut installer `djangorestframework-camel-case` pour la conversion automatique :

```bash
pip install djangorestframework-camel-case
```

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'djangorestframework_camel_case.render.CamelCaseJSONRenderer',
    ],
    'DEFAULT_PARSER_CLASSES': [
        'djangorestframework_camel_case.parser.CamelCaseJSONParser',
    ],
}
```

Correspondance des types :

| TypeScript (front) | Django (back) |
|-------------------|---------------|
| `fullName: string` | `full_name = CharField()` |
| `createdAt: string` | `created_at = DateTimeField()` |
| `assignedEngineerId?: string` | `assigned_engineer = ForeignKey(self)` |
| `slaDueAt: string` | `sla_due_at = DateTimeField()` |
| `equipmentIds: string[]` | `equipments = ManyToManyField()` |
| `reportPdfName?: string` | `report_pdf_path = CharField()` |

---

## 🔧 lib/format.ts — Fonctions utilitaires

```typescript
fmtDate("2024-11-15T14:00:00Z")     // → "15 nov. 2024"
fmtDateTime("2024-11-15T14:00:00Z") // → "15 nov. 2024, 14:00"
fmtRelative("2024-11-15T14:00:00Z") // → "il y a 3 heures"

slaInfo("2024-11-15T14:00:00Z")
// → { breached: true/false, text: "Dépassé de 2h", tone: "text-destructive" }
// Le calcul SLA est fait CÔTÉ FRONT à partir de la date slaDueAt fournie par Django
// Django calcule et stocke slaDueAt, le front l'affiche
```

---

## 🗃️ data/seed.ts — Données de démo

Contient des utilisateurs, tickets, contrats, maintenances factices.
Utilisé uniquement en mode mock pour que le front soit testable sans Django.
**Une fois Django branché, ce fichier devient inutile.**
Tu n'as pas à y toucher.

---

## 📋 Résumé — Les 8 choses à savoir

| # | Point |
|---|-------|
| 1 | **Un seul fichier à modifier** : `src/services/api.ts` |
| 2 | **Lire `src/types/index.ts`** pour connaître les modèles attendus |
| 3 | **camelCase côté front** ↔ **snake_case côté Django** → installer `djangorestframework-camel-case` |
| 4 | **Le rôle** est dans l'objet `user` retourné par Django au login |
| 5 | **Aucune page d'inscription publique** — l'admin crée tous les comptes |
| 6 | **3 pages partagées** : `TicketDetail`, `CommercialContracts`, `EngineerMaintenance` — s'adaptent selon le rôle |
| 7 | **Le front masque les boutons** selon le rôle — Django doit aussi bloquer les requêtes non autorisées |
| 8 | **La logique métier** (validation, règles) = côté Django uniquement — le front affiche, Django décide |

