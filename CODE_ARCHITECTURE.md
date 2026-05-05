# 📐 Architecture du code Front — UPSILON Helpdesk & SAV
## Guide de lecture du code React pour le développeur Django

---

## 🗂️ Structure complète du projet

```
src/
├── main.tsx                        ← Point d'entrée React (monte App dans #root)
├── App.tsx                         ← Routeur principal + providers globaux
├── index.css                       ← Variables CSS globales (thème, couleurs)
│
├── types/
│   └── index.ts                    ← ⭐ TOUS les types TypeScript du projet
│                                      = les modèles Django à reproduire exactement
│
├── services/
│   └── api.ts                      ← ⭐ FICHIER UNIQUE À REMPLACER
│                                      Toutes les fonctions mock → vrais fetch()
│
├── contexts/
│   └── AuthContext.tsx             ← Gestion de l'utilisateur connecté (JWT à brancher)
│
├── data/
│   └── seed.ts                     ← Données de démo (remplacées par Django)
│
├── lib/
│   ├── format.ts                   ← Fonctions utilitaires (dates, SLA)
│   └── utils.ts                    ← Helper CSS (cn())
│
├── hooks/
│   └── use-toast.ts                ← Notifications toast
│
├── components/
│   ├── auth/
│   │   └── ProtectedRoute.tsx      ← Garde de route par rôle
│   ├── layout/
│   │   ├── AppLayout.tsx           ← Layout principal (sidebar + contenu)
│   │   ├── AppHeader.tsx           ← En-tête avec bouton "Nouveau ticket" conditionnel
│   │   └── AppSidebar.tsx          ← Navigation latérale par rôle
│   ├── dashboard/
│   │   └── KpiCard.tsx             ← Carte KPI réutilisable
│   ├── tickets/
│   │   └── StatusBadge.tsx         ← Badges statut/priorité tickets
│   └── ui/                         ← Composants shadcn/ui (NE PAS MODIFIER)
│
└── pages/
    ├── Login.tsx                   ← Page de connexion
    ├── DashboardRouter.tsx         ← Aiguillage dashboard selon le rôle
    ├── NotFound.tsx                ← Page 404
    │
    ├── client/                     ← Pages accessibles au rôle "client"
    │   ├── ClientDashboard.tsx
    │   ├── ClientTickets.tsx
    │   ├── NewTicket.tsx
    │   ├── ClientContracts.tsx
    │   ├── ClientMaintenances.tsx
    │   └── ClientEquipments.tsx
    │
    ├── engineer/                   ← Pages accessibles au rôle "engineer"
    │   ├── EngineerDashboard.tsx
    │   ├── EngineerTickets.tsx
    │   ├── NewEngineerTicket.tsx
    │   └── EngineerMaintenance.tsx
    │
    ├── commercial/                 ← Pages accessibles au rôle "commercial"
    │   ├── CommercialDashboard.tsx
    │   ├── CommercialContracts.tsx ← aussi utilisé par engineer (lecture) et admin (lecture)
    │   └── CommercialClients.tsx
    │
    ├── admin/                      ← Pages accessibles au rôle "admin"
    │   ├── AdminDashboard.tsx
    │   ├── AdminTickets.tsx
    │   ├── AdminUsers.tsx
    │   ├── AdminEscalations.tsx
    │   ├── AdminAssignments.tsx
    │   └── AdminStatistics.tsx
    │
    └── shared/                     ← Pages accessibles à plusieurs rôles
        ├── TicketDetail.tsx        ← Vue détail ticket (client + engineer + admin)
        └── ProfilePage.tsx         ← Profil utilisateur (tous les rôles)
```

---

## 🔄 Flux de démarrage de l'application

```
main.tsx
  └── App.tsx
        ├── QueryClientProvider     (React Query — gestion des requêtes async)
        ├── TooltipProvider         (shadcn/ui)
        ├── Toaster / Sonner        (notifications toast)
        ├── BrowserRouter           (react-router-dom)
        └── AuthProvider            (contexte auth — utilisateur courant)
              └── Routes
                    ├── /login      → Login.tsx (public)
                    └── /*          → ProtectedRoute → AppLayout
                                          └── Pages selon le rôle
```

---

## 🔐 Système d'authentification (AuthContext.tsx)

### Fonctionnement actuel (mock)
```typescript
// L'utilisateur est simulé — trouvé dans seed.ts par email
const login = async (email, password) => {
  const found = seedUsers.find(u => u.email === email);
  localStorage.setItem('upsilon.auth.user', JSON.stringify(found));
  return found;
}
```

### Ce qu'il faut faire côté Django
Remplacer par un vrai appel HTTP :
```typescript
const login = async (email: string, password: string): Promise<User> => {
  const res = await fetch(`${API_URL}/api/auth/login/`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });
  if (!res.ok) throw new Error('Identifiants invalides');
  const data = await res.json();
  // data = { access: "eyJ...", refresh: "eyJ...", user: { id, email, role, ... } }
  localStorage.setItem('upsilon_token', data.access);
  localStorage.setItem('upsilon_refresh', data.refresh);
  return data.user; // ← l'objet User avec le rôle
}
```

### Le rôle dans le token
Le rôle de l'utilisateur est lu depuis l'objet `user` retourné par Django au login.
Le front le stocke dans le state React et l'utilise partout via `useAuth()`.

---

## 🛡️ Système de protection des routes (ProtectedRoute.tsx)

```typescript
// Chaque route dans App.tsx est protégée par un rôle :
<ProtectedRoute roles={["engineer"]}>
  <EngineerDashboard />
</ProtectedRoute>

// La logique interne :
// 1. Si pas connecté → redirect vers /login
// 2. Si rôle non autorisé → redirect vers /
// 3. Sinon → affiche le composant
```

**Toutes les routes et leurs rôles autorisés :**

| URL | Rôles autorisés |
|-----|----------------|
| `/` | Tous (aiguillage automatique) |
| `/login` | Public |
| `/profil` | Tous |
| `/mes-tickets` | client |
| `/mes-tickets/nouveau` | client |
| `/mes-tickets/:id` | client, engineer, admin |
| `/mes-contrats` | client |
| `/mes-maintenances` | client |
| `/mes-equipements` | client |
| `/ing/tickets` | engineer |
| `/ing/tickets/nouveau` | engineer |
| `/ing/maintenance` | engineer |
| `/ing/contrats` | engineer (lecture seule) |
| `/co/contrats` | commercial |
| `/co/renouvellements` | commercial |
| `/co/clients` | commercial, admin |
| `/admin/tickets` | admin |
| `/admin/escalades` | admin |
| `/admin/maintenance` | admin (lecture seule) |
| `/admin/attributions` | admin |
| `/admin/contrats` | admin (lecture seule) |
| `/admin/utilisateurs` | admin |
| `/admin/statistiques` | admin |

---

## 🔑 Système de permissions dans les composants

Le front applique une **double protection** :

### 1. Route protégée (App.tsx)
Empêche l'accès à la page si mauvais rôle.

### 2. UI conditionnelle dans les composants
Masque les boutons d'action selon le rôle.

**Les 4 flags principaux :**

```typescript
// AppHeader.tsx
const canCreateTicket = user.role === "client" || user.role === "engineer";

// CommercialContracts.tsx
const canEdit = user.role === "commercial";
// → affiche/masque : Nouveau contrat, Renouveler, Modifier, Supprimer

// EngineerMaintenance.tsx
const canPlan = user.role === "engineer";
// → affiche/masque : Planifier, Uploader rapport

// TicketDetail.tsx
const isEngineer = user.role === "engineer";
const isAdmin    = user.role === "admin";
const isClient   = user.role === "client";
// → Engineer : répondre, escalader, clôturer, mettre en attente
// → Admin    : attribuer à un ingénieur, lever l'escalade UNIQUEMENT
// → Client   : répondre, confirmer fermeture
```

---

## 📡 Couche service (api.ts) — Le seul fichier à modifier

Toutes les interactions avec les données passent par ce fichier.
Chaque service correspond à un groupe d'endpoints Django.

### Structure actuelle (mock)
```typescript
export const ticketsService = {
  async list(): Promise<Ticket[]> { ... },
  async byClient(clientId): Promise<Ticket[]> { ... },
  async byEngineer(engineerId): Promise<Ticket[]> { ... },
  async byId(id): Promise<Ticket | undefined> { ... },
  async create(data): Promise<Ticket> { ... },
  async update(id, patch): Promise<Ticket | undefined> { ... },
  async addMessage(id, message): Promise<Ticket | undefined> { ... },
}
```

### Les 5 services disponibles

| Service | Correspond à |
|---------|-------------|
| `usersService` | `/api/users/` + `/api/auth/` |
| `equipmentsService` | `/api/equipments/` |
| `contractsService` | `/api/contracts/` |
| `ticketsService` | `/api/tickets/` |
| `maintenancesService` | `/api/maintenances/` |

### Comment chaque page utilise les services
```typescript
// Pattern standard dans chaque page :
useEffect(() => {
  ticketsService.byClient(user.id).then(setTickets);
}, [user]);
```

---

## 📦 Types TypeScript = Modèles Django

Le fichier `src/types/index.ts` définit tous les modèles.
**Ces types doivent correspondre exactement aux serializers Django.**

```typescript
interface User {
  id: string;            // UUID
  email: string;
  fullName: string;      // → full_name en Django (snake_case)
  initials: string;      // calculé automatiquement
  role: UserRole;        // "client"|"engineer"|"commercial"|"admin"
  company?: string;
  phone?: string;
  createdAt: string;     // → created_at (ISO 8601)
  assignedEngineerId?: string; // → assigned_engineer_id
}
```

> ⚠️ **Important** : Le front utilise le **camelCase** (fullName, createdAt).
> Django retourne du **snake_case** (full_name, created_at).
> Il faut configurer DRF pour retourner du camelCase :
> ```python
> # settings.py
> REST_FRAMEWORK = {
>     'DEFAULT_RENDERER_CLASSES': ['djangorestframework_camel_case.render.CamelCaseJSONRenderer'],
>     'DEFAULT_PARSER_CLASSES': ['djangorestframework_camel_case.parser.CamelCaseJSONParser'],
> }
> # pip install djangorestframework-camel-case
> ```
> **OU** faire la conversion dans les serializers manuellement.

---

## 🔧 Utilitaires (lib/format.ts)

```typescript
fmtDate(iso)      // "15 nov. 2024"
fmtDateTime(iso)  // "15 nov. 2024, 14:30"
fmtRelative(iso)  // "il y a 3 heures"

slaInfo(slaDueAt) // retourne { breached: bool, text: string, tone: string }
// breached = true si la date SLA est dépassée
// tone     = classe CSS couleur (vert/orange/rouge)
```

---

## 🗃️ Données de démo (data/seed.ts)

Ce fichier contient les données mockées utilisées en mode demo.
**Il sera complètement inutile une fois Django branché.**
Tu n'as pas à t'en préoccuper.

---

## 🧩 Composants partagés importants

### DashboardRouter.tsx
Redirige automatiquement vers le bon dashboard selon le rôle :
```typescript
switch (user.role) {
  case "client":     return <ClientDashboard />;
  case "engineer":   return <EngineerDashboard />;
  case "commercial": return <CommercialDashboard />;
  case "admin":      return <AdminDashboard />;
}
```

### TicketDetail.tsx (partagé)
Utilisé par 3 rôles différents (client, engineer, admin) avec des actions différentes.
Le même composant s'adapte grâce aux flags `isClient`, `isEngineer`, `isAdmin`.

### CommercialContracts.tsx (partagé)
Utilisé par commercial (écriture), engineer (lecture), admin (lecture).
Le flag `canEdit = user.role === "commercial"` masque tous les boutons d'action.

### EngineerMaintenance.tsx (partagé)
Utilisé par engineer (écriture), admin (lecture).
Le flag `canPlan = user.role === "engineer"` masque Planifier et Upload.

---

## 🎨 Interface utilisateur

- **Framework UI** : shadcn/ui + Tailwind CSS
- **Graphiques** : Recharts (BarChart, PieChart)
- **Icônes** : lucide-react
- **Routing** : react-router-dom v6
- **Notifications** : Toast (shadcn/ui) + Sonner

Les composants `src/components/ui/` sont des composants shadcn/ui standards.
**Ne pas les modifier.**

---

## ⚡ Résumé — Ce que Django doit savoir

1. **Un seul fichier front à modifier** : `src/services/api.ts`
2. **Les types TypeScript** dans `src/types/index.ts` = les modèles Django
3. **camelCase côté front** ↔ **snake_case côté Django** (à gérer dans DRF)
4. **Le rôle** est dans le token JWT et lu par le front au login
5. **La logique métier** (validation, règles) = côté Django uniquement
6. **Le front masque les boutons** mais **Django bloque les requêtes non autorisées**
7. **Aucune page d'inscription publique** — l'admin crée tous les comptes

