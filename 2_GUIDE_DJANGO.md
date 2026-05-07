# 🛠️ Guide Backend Django — UPSILON Helpdesk & SAV
## Tout ce qu'il faut faire, étape par étape

---

## 🎯 Objectif

Construire une API REST Django qui remplace les données simulées du front React.
Le front est prêt. Tes endpoints doivent respecter exactement les signatures ci-dessous.

---

## 📦 Installation

```bash
pip install django djangorestframework djangorestframework-simplejwt \
            django-cors-headers psycopg2-binary django-storages \
            boto3 django-anymail djangorestframework-camel-case \
            python-dotenv
```

---

## 📁 Structure du projet Django

```
backend/
├── manage.py
├── .env
├── config/
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
└── apps/
    ├── users/          → User + auth + profil
    ├── tickets/        → Ticket + Message + Attachment
    ├── contracts/      → Contract + table pivot M2M équipements
    ├── equipments/     → Equipment
    └── maintenances/   → Maintenance + upload PDF + email
```

---

## ⚙️ settings.py complet

```python
import os
from datetime import timedelta
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()
BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = os.getenv('SECRET_KEY')
DEBUG = os.getenv('DEBUG', 'True') == 'True'
ALLOWED_HOSTS = ['*']

INSTALLED_APPS = [
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'rest_framework',
    'rest_framework_simplejwt',
    'rest_framework_simplejwt.token_blacklist',
    'corsheaders',
    'apps.users',
    'apps.tickets',
    'apps.contracts',
    'apps.equipments',
    'apps.maintenances',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',   # ← DOIT être en PREMIER
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
]

AUTH_USER_MODEL = 'users.User'

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME', 'upsilon_db'),
        'USER': os.getenv('DB_USER', 'upsilon_user'),
        'PASSWORD': os.getenv('DB_PASSWORD'),
        'HOST': os.getenv('DB_HOST', 'localhost'),
        'PORT': os.getenv('DB_PORT', '5432'),
    }
}

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    # Conversion automatique snake_case → camelCase pour le front React
    'DEFAULT_RENDERER_CLASSES': [
        'djangorestframework_camel_case.render.CamelCaseJSONRenderer',
    ],
    'DEFAULT_PARSER_CLASSES': [
        'djangorestframework_camel_case.parser.CamelCaseJSONParser',
    ],
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(hours=8),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
}

CORS_ALLOWED_ORIGINS = [
    "http://localhost:5173",       # Vite dev
    "https://votre-domaine.com",
]

# Upload PDF rapports de maintenance
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
AWS_STORAGE_BUCKET_NAME = os.getenv('AWS_BUCKET', 'upsilon-reports')
AWS_S3_REGION_NAME = os.getenv('AWS_REGION', 'eu-west-3')
MEDIA_URL = f'https://{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com/'

# Email notifications maintenance
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = os.getenv('EMAIL_HOST')
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = os.getenv('EMAIL_USER')
EMAIL_HOST_PASSWORD = os.getenv('EMAIL_PASSWORD')
DEFAULT_FROM_EMAIL = 'noreply@upsilon.com'
```

---

## 📄 .env à créer

```env
SECRET_KEY=une-cle-secrete-longue
DEBUG=True
DB_NAME=upsilon_db
DB_USER=upsilon_user
DB_PASSWORD=votre_mot_de_passe
DB_HOST=localhost
DB_PORT=5432
AWS_BUCKET=upsilon-reports
AWS_REGION=eu-west-3
EMAIL_HOST=smtp.votre-provider.com
EMAIL_USER=noreply@upsilon.com
EMAIL_PASSWORD=votre_mot_de_passe
VITE_API_BASE_URL=http://localhost:8000
```

---

## 👤 Modèles Django

### apps/users/models.py

```python
import uuid
from django.contrib.auth.models import AbstractBaseUser, BaseUserManager
from django.db import models

class UserManager(BaseUserManager):
    def create_user(self, email, full_name, role, password=None, **kwargs):
        user = self.model(email=email, full_name=full_name, role=role, **kwargs)
        user.set_password(password)
        user.save(using=self._db)
        return user

    def create_superuser(self, email, full_name, password=None, **kwargs):
        return self.create_user(email, full_name, 'admin', password, **kwargs)

class User(AbstractBaseUser):
    class Role(models.TextChoices):
        CLIENT     = 'client',     'Client'
        ENGINEER   = 'engineer',   'Ingénieur'
        COMMERCIAL = 'commercial', 'Commercial'
        ADMIN      = 'admin',      'Super Admin'

    id                = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    email             = models.EmailField(unique=True)
    full_name         = models.CharField(max_length=150)
    initials          = models.CharField(max_length=5)
    role              = models.CharField(max_length=20, choices=Role.choices)
    company           = models.CharField(max_length=150, blank=True, null=True)
    phone             = models.CharField(max_length=30, blank=True, null=True)
    is_active         = models.BooleanField(default=True)
    created_at        = models.DateTimeField(auto_now_add=True)
    assigned_engineer = models.ForeignKey(
        'self', null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='assigned_clients',
        limit_choices_to={'role': 'engineer'}
    )

    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['full_name']
    objects = UserManager()

    def save(self, *args, **kwargs):
        # Calcul automatique des initiales
        if not self.initials and self.full_name:
            parts = self.full_name.strip().split()
            self.initials = (parts[0][0] + parts[-1][0]).upper() if len(parts) >= 2 \
                            else parts[0][:2].upper()
        super().save(*args, **kwargs)

    class Meta:
        db_table = 'upsilon_user'
```

### apps/equipments/models.py

```python
class Equipment(models.Model):
    id           = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    serial       = models.CharField(max_length=100, unique=True)
    label        = models.CharField(max_length=200)
    type         = models.CharField(max_length=100)
    brand        = models.CharField(max_length=100)
    model        = models.CharField(max_length=100)
    client       = models.ForeignKey('users.User', on_delete=models.CASCADE,
                                     related_name='equipments',
                                     limit_choices_to={'role': 'client'})
    warranty_end = models.DateField(null=True, blank=True)
    installed_at = models.DateField(null=True, blank=True)

    class Meta:
        db_table = 'upsilon_equipment'
```

### apps/contracts/models.py

```python
class Contract(models.Model):
    class Status(models.TextChoices):
        ACTIF          = 'actif'
        EXPIRE_BIENTOT = 'expire_bientot'
        EXPIRE         = 'expire'
        RENOUVELE      = 'renouvele'

    id           = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    reference    = models.CharField(max_length=50, unique=True)
    client       = models.ForeignKey('users.User', on_delete=models.CASCADE,
                                     related_name='contracts',
                                     limit_choices_to={'role': 'client'})
    start_date   = models.DateField()
    end_date     = models.DateField()
    status       = models.CharField(max_length=20, choices=Status.choices, default=Status.ACTIF)
    scope        = models.TextField()
    equipments   = models.ManyToManyField('equipments.Equipment', blank=True)
    amount       = models.DecimalField(max_digits=12, decimal_places=3, null=True, blank=True)
    created_by   = models.ForeignKey('users.User', on_delete=models.SET_NULL,
                                     null=True, related_name='created_contracts')
    created_at   = models.DateTimeField(auto_now_add=True)

    def save(self, *args, **kwargs):
        # Génération automatique de la référence
        if not self.reference:
            count = Contract.objects.count() + 1
            self.reference = f"CTR-{self.start_date.year}-{str(count).zfill(3)}"
        # Calcul automatique du statut selon les dates
        from django.utils.timezone import now
        today = now().date()
        days_left = (self.end_date - today).days
        if days_left < 0:
            self.status = 'expire'
        elif days_left <= 60:
            self.status = 'expire_bientot'
        super().save(*args, **kwargs)

    class Meta:
        db_table = 'upsilon_contract'
```

### apps/tickets/models.py

```python
class Ticket(models.Model):
    class Status(models.TextChoices):
        OUVERT     = 'ouvert'
        EN_COURS   = 'en_cours'
        EN_ATTENTE = 'en_attente'
        RESOLU     = 'resolu'
        FERME      = 'ferme'

    class Priority(models.TextChoices):
        FAIBLE   = 'faible'
        MOYENNE  = 'moyenne'
        HAUTE    = 'haute'
        CRITIQUE = 'critique'

    class Category(models.TextChoices):
        PANNE       = 'panne'
        DEMANDE     = 'demande'
        INCIDENT    = 'incident'
        MAINTENANCE = 'maintenance'
        AUTRE       = 'autre'

    id          = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    reference   = models.CharField(max_length=20, unique=True)
    subject     = models.CharField(max_length=255)
    description = models.TextField()
    category    = models.CharField(max_length=20, choices=Category.choices)
    status      = models.CharField(max_length=20, choices=Status.choices, default=Status.OUVERT)
    priority    = models.CharField(max_length=10, choices=Priority.choices, default=Priority.MOYENNE)
    escalated   = models.BooleanField(default=False)
    client      = models.ForeignKey('users.User', on_delete=models.CASCADE,
                                    related_name='tickets', limit_choices_to={'role': 'client'})
    equipment   = models.ForeignKey('equipments.Equipment', on_delete=models.CASCADE)
    contract    = models.ForeignKey('contracts.Contract', on_delete=models.SET_NULL,
                                    null=True, blank=True)
    assignee    = models.ForeignKey('users.User', on_delete=models.SET_NULL,
                                    null=True, blank=True, related_name='assigned_tickets',
                                    limit_choices_to={'role': 'engineer'})
    sla_due_at  = models.DateTimeField()
    created_at  = models.DateTimeField(auto_now_add=True)
    updated_at  = models.DateTimeField(auto_now=True)

    def save(self, *args, **kwargs):
        if not self.reference:
            count = Ticket.objects.count() + 1
            self.reference = f"TKT-{2418 + count}"
        if not self.sla_due_at:
            from django.utils.timezone import now
            from datetime import timedelta
            SLA = {'critique': 4, 'haute': 8, 'moyenne': 24, 'faible': 72}
            self.sla_due_at = now() + timedelta(hours=SLA.get(self.priority, 24))
        super().save(*args, **kwargs)

    class Meta:
        db_table = 'upsilon_ticket'
        ordering = ['-created_at']


class TicketMessage(models.Model):
    id         = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    ticket     = models.ForeignKey(Ticket, on_delete=models.CASCADE, related_name='messages')
    author     = models.ForeignKey('users.User', on_delete=models.CASCADE)
    content    = models.TextField()
    internal   = models.BooleanField(default=False)  # note interne ingénieur
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'upsilon_ticket_message'
        ordering = ['created_at']


class MessageAttachment(models.Model):
    id         = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    message    = models.ForeignKey(TicketMessage, on_delete=models.CASCADE,
                                   related_name='attachments')
    name       = models.CharField(max_length=255)
    size_label = models.CharField(max_length=30)
    file_path  = models.CharField(max_length=500)

    class Meta:
        db_table = 'upsilon_message_attachment'
```

### apps/maintenances/models.py

```python
class Maintenance(models.Model):
    class Type(models.TextChoices):
        PREVENTIVE = 'preventive'
        CURATIVE   = 'curative'

    class Status(models.TextChoices):
        PLANIFIEE = 'planifiee'
        EN_COURS  = 'en_cours'
        REALISEE  = 'realisee'
        ANNULEE   = 'annulee'

    id                 = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    reference          = models.CharField(max_length=30, unique=True)
    type               = models.CharField(max_length=15, choices=Type.choices)
    status             = models.CharField(max_length=15, choices=Status.choices,
                                          default=Status.PLANIFIEE)
    client             = models.ForeignKey('users.User', on_delete=models.CASCADE,
                                           related_name='maintenances_client',
                                           limit_choices_to={'role': 'client'})
    equipment          = models.ForeignKey('equipments.Equipment', on_delete=models.CASCADE)
    engineer           = models.ForeignKey('users.User', on_delete=models.CASCADE,
                                           related_name='maintenances_engineer',
                                           limit_choices_to={'role': 'engineer'})
    contract           = models.ForeignKey('contracts.Contract', on_delete=models.SET_NULL,
                                           null=True, blank=True)
    scheduled_at       = models.DateTimeField()
    duration_minutes   = models.IntegerField(default=60)
    notes              = models.TextField(blank=True, null=True)
    report_pdf_path    = models.CharField(max_length=500, blank=True, null=True)
    report_uploaded_at = models.DateTimeField(null=True, blank=True)
    created_at         = models.DateTimeField(auto_now_add=True)

    def save(self, *args, **kwargs):
        if not self.reference:
            from django.utils.timezone import now
            count = Maintenance.objects.count() + 1
            self.reference = f"MNT-{now().year}-{str(count).zfill(3)}"
        super().save(*args, **kwargs)

    class Meta:
        db_table = 'upsilon_maintenance'
        ordering = ['scheduled_at']
```

---

## 🔐 Permissions par rôle

```python
# apps/users/permissions.py
from rest_framework.permissions import BasePermission

class IsClient(BasePermission):
    def has_permission(self, request, view):
        return request.user.role == 'client'

class IsEngineer(BasePermission):
    def has_permission(self, request, view):
        return request.user.role == 'engineer'

class IsCommercial(BasePermission):
    def has_permission(self, request, view):
        return request.user.role == 'commercial'

class IsAdmin(BasePermission):
    def has_permission(self, request, view):
        return request.user.role == 'admin'

class IsClientOrEngineer(BasePermission):
    def has_permission(self, request, view):
        return request.user.role in ('client', 'engineer')

class IsEngineerOrAdmin(BasePermission):
    def has_permission(self, request, view):
        return request.user.role in ('engineer', 'admin')

class IsCommercialOrAdmin(BasePermission):
    def has_permission(self, request, view):
        return request.user.role in ('commercial', 'admin')
```

---

## 🌐 Tous les endpoints à implémenter

### AUTH

```python
# config/urls.py
from rest_framework_simplejwt.views import TokenRefreshView

urlpatterns = [
    path('api/auth/login/',           LoginView.as_view()),
    path('api/auth/refresh/',         TokenRefreshView.as_view()),
    path('api/auth/logout/',          LogoutView.as_view()),
    path('api/auth/change-password/', ChangePasswordView.as_view()),
    path('api/users/',                include('apps.users.urls')),
    path('api/equipments/',           include('apps.equipments.urls')),
    path('api/contracts/',            include('apps.contracts.urls')),
    path('api/tickets/',              include('apps.tickets.urls')),
    path('api/maintenances/',         include('apps.maintenances.urls')),
]
```

**POST /api/auth/login/**
Réponse attendue par le front :
```json
{
  "access": "eyJ...",
  "refresh": "eyJ...",
  "user": {
    "id": "uuid",
    "email": "ali@upsilon.com",
    "fullName": "Ali Ben Salem",
    "initials": "AB",
    "role": "engineer",
    "company": null,
    "phone": "+216 XX XXX XXX",
    "createdAt": "2024-01-15T10:00:00Z",
    "assignedEngineerId": null
  }
}
```

---

### USERS

| Méthode | URL | Permission | Description |
|---------|-----|-----------|-------------|
| GET | `/api/users/` | Admin | Liste tous les utilisateurs |
| GET | `/api/users/?role=engineer` | Admin, Commercial | Filtre par rôle |
| GET | `/api/users/:id/` | Admin | Détail |
| POST | `/api/users/` | Admin | Créer compte + mot de passe |
| PATCH | `/api/users/:id/` | Admin ou propriétaire | Modifier nom/téléphone/entreprise |
| DELETE | `/api/users/:id/` | Admin | Supprimer |
| POST | `/api/auth/change-password/` | Propriétaire | Changer son mot de passe |

**Body POST /api/users/ :**
```json
{
  "email": "sara@tunisair.com",
  "fullName": "Sara Trabelsi",
  "role": "client",
  "password": "motdepasse123",
  "company": "Tunisair",
  "phone": "+216 71 000 000"
}
```

**Body POST /api/auth/change-password/ :**
```json
{
  "currentPassword": "ancien_mdp",
  "newPassword": "nouveau_mdp"
}
```
> Django vérifie `currentPassword` avec `check_password()` avant d'accepter.

---

### EQUIPMENTS

| Méthode | URL | Permission | Description |
|---------|-----|-----------|-------------|
| GET | `/api/equipments/` | Tous | Liste |
| GET | `/api/equipments/?client_id=:id` | Tous | Par client |

---

### CONTRACTS

| Méthode | URL | Permission | Description |
|---------|-----|-----------|-------------|
| GET | `/api/contracts/` | Commercial, Admin, Engineer | Liste |
| GET | `/api/contracts/?client_id=:id` | Tous | Par client |
| GET | `/api/contracts/?engineer_id=:id` | Engineer, Admin | Par ingénieur |
| POST | `/api/contracts/` | Commercial | Créer |
| PATCH | `/api/contracts/:id/renew/` | Commercial | Renouveler |
| PATCH | `/api/contracts/:id/scope/` | Commercial | Modifier périmètre |
| DELETE | `/api/contracts/:id/` | Commercial | Supprimer |

**Body POST /api/contracts/ :**
```json
{
  "clientId": "uuid",
  "startDate": "2024-01-01",
  "endDate": "2026-01-01",
  "scope": "Maintenance préventive + assistance 24/7",
  "equipmentIds": ["uuid-eq1", "uuid-eq2"],
  "amount": 48000
}
```

**Body PATCH /api/contracts/:id/renew/ :**
```json
{ "newStartDate": "2026-01-01", "newEndDate": "2028-01-01" }
```

**Body PATCH /api/contracts/:id/scope/ :**
```json
{ "scope": "Nouveau périmètre...", "equipmentIds": ["uuid-eq1", "uuid-eq3"] }
```

> 💡 **Logique métier contrats** : Le front affiche toujours les boutons Renouveler/Modifier/Supprimer.
> C'est Django qui décide si l'action est autorisée. Ex : refuser de supprimer un contrat
> avec des tickets ouverts → retourner `400 Bad Request` avec message explicatif.

---

### TICKETS

| Méthode | URL | Permission | Description |
|---------|-----|-----------|-------------|
| GET | `/api/tickets/` | Admin | Tous les tickets |
| GET | `/api/tickets/?client_id=:id` | Client, Engineer, Admin | Par client |
| GET | `/api/tickets/?engineer_id=:id` | Engineer, Admin | Par ingénieur |
| GET | `/api/tickets/:id/` | Client, Engineer, Admin | Détail + messages |
| POST | `/api/tickets/` | Client, Engineer | Créer |
| PATCH | `/api/tickets/:id/` | Engineer, Admin | Modifier statut/assignation/escalade |
| POST | `/api/tickets/:id/messages/` | Client, Engineer | Ajouter message/pièce jointe |

**Body POST /api/tickets/ :**
```json
{
  "subject": "Onduleur HS salle serveur",
  "description": "L'onduleur ne répond plus depuis ce matin...",
  "category": "incident",
  "priority": "haute",
  "clientId": "uuid-client",
  "equipmentId": "uuid-eq",
  "contractId": "uuid-contrat"
}
```

> ⚠️ **Catégories ingénieur** : uniquement `incident` et `demande`.
> Le client peut utiliser toutes les catégories.

**Calcul automatique SLA côté Django :**
```python
SLA_HOURS = {'critique': 4, 'haute': 8, 'moyenne': 24, 'faible': 72}
sla_due_at = now() + timedelta(hours=SLA_HOURS[priority])
```

**Body PATCH /api/tickets/:id/ (exemples) :**
```json
{ "status": "en_cours" }
{ "assigneeId": "uuid-engineer" }
{ "escalated": true, "priority": "critique" }
{ "escalated": false }
```

**Body POST /api/tickets/:id/messages/ :**
```json
{
  "authorId": "uuid-user",
  "content": "Le problème vient de la batterie...",
  "internal": false,
  "attachments": [
    { "name": "procedure.pdf", "size": "42 Ko", "url": "/media/procedure.pdf" }
  ]
}
```

---

### MAINTENANCES

| Méthode | URL | Permission | Description |
|---------|-----|-----------|-------------|
| GET | `/api/maintenances/` | Admin | Toutes |
| GET | `/api/maintenances/?client_id=:id` | Client, Engineer, Admin | Par client |
| GET | `/api/maintenances/?engineer_id=:id` | Engineer, Admin | Par ingénieur |
| POST | `/api/maintenances/` | Engineer | Planifier |
| PATCH | `/api/maintenances/:id/` | Engineer | Modifier date/type/durée/notes |
| POST | `/api/maintenances/:id/upload-report/` | Engineer | Upload PDF + email client |

**Body POST /api/maintenances/ :**
```json
{
  "type": "preventive",
  "clientId": "uuid-client",
  "equipmentId": "uuid-eq",
  "engineerId": "uuid-engineer",
  "scheduledAt": "2024-11-20T09:00:00Z",
  "durationMinutes": 120,
  "notes": "Vérification complète batterie + firmware"
}
```

**Upload rapport + email :**
```python
# apps/maintenances/views.py
class UploadReportView(APIView):
    permission_classes = [IsEngineer]

    def post(self, request, pk):
        maintenance = get_object_or_404(Maintenance, pk=pk)
        pdf_file = request.FILES.get('file')

        # Sauvegarder le fichier
        maintenance.report_pdf_path = default_storage.save(
            f'reports/{maintenance.reference}.pdf', pdf_file
        )
        maintenance.report_uploaded_at = now()
        maintenance.status = 'realisee'
        maintenance.save()

        # Envoyer email au client
        send_mail(
            subject=f'Rapport maintenance {maintenance.reference}',
            message=(
                f'Bonjour {maintenance.client.full_name},\n\n'
                f'Le rapport de votre maintenance du '
                f'{maintenance.scheduled_at.strftime("%d/%m/%Y")} '
                f'est disponible sur votre espace client.\n\n'
                f'Cordialement,\nUpsilon Support'
            ),
            from_email=settings.DEFAULT_FROM_EMAIL,
            recipient_list=[maintenance.client.email],
        )

        return Response(MaintenanceSerializer(maintenance).data)
```

---

## 🔄 Comment brancher le front React

### 1. Créer le fichier .env à la racine du front

```env
VITE_API_BASE_URL=http://localhost:8000
```

### 2. Modifier AuthContext.tsx

Remplacer la fonction `login` par le vrai appel HTTP (voir `1_CODE_ARCHITECTURE.md`).

### 3. Modifier api.ts — Exemple complet pour ticketsService

```typescript
const API_URL = import.meta.env.VITE_API_BASE_URL;

const getToken = () => localStorage.getItem('upsilon_token');
const headers = () => ({
  'Content-Type': 'application/json',
  'Authorization': `Bearer ${getToken()}`
});

export const ticketsService = {
  async list() {
    const res = await fetch(`${API_URL}/api/tickets/`, { headers: headers() });
    return res.json();
  },
  async byClient(clientId: string) {
    const res = await fetch(`${API_URL}/api/tickets/?client_id=${clientId}`, { headers: headers() });
    return res.json();
  },
  async byEngineer(engineerId: string) {
    const res = await fetch(`${API_URL}/api/tickets/?engineer_id=${engineerId}`, { headers: headers() });
    return res.json();
  },
  async byId(id: string) {
    const res = await fetch(`${API_URL}/api/tickets/${id}/`, { headers: headers() });
    return res.json();
  },
  async create(data: any) {
    const res = await fetch(`${API_URL}/api/tickets/`, {
      method: 'POST', headers: headers(), body: JSON.stringify(data)
    });
    return res.json();
  },
  async update(id: string, patch: any) {
    const res = await fetch(`${API_URL}/api/tickets/${id}/`, {
      method: 'PATCH', headers: headers(), body: JSON.stringify(patch)
    });
    return res.json();
  },
  async addMessage(id: string, message: any) {
    const res = await fetch(`${API_URL}/api/tickets/${id}/messages/`, {
      method: 'POST', headers: headers(), body: JSON.stringify(message)
    });
    return res.json();
  },
};
```

> Répéter le même pattern pour `usersService`, `contractsService`, `equipmentsService`, `maintenancesService`.

---

## 🚀 Mise en place pas à pas

```bash
# 1. Créer la base PostgreSQL
psql -U postgres -c "CREATE DATABASE upsilon_db;"
psql -U postgres -c "CREATE USER upsilon_user WITH PASSWORD 'votre_mdp';"
psql -U postgres -c "GRANT ALL PRIVILEGES ON DATABASE upsilon_db TO upsilon_user;"

# 2. Migrations Django
python manage.py makemigrations
python manage.py migrate

# 3. Créer le premier Super Admin
python manage.py createsuperuser
# Saisir : email, full_name, password

# 4. Lancer Django
python manage.py runserver 8000

# 5. Le front React tourne en parallèle
# (dans le dossier front)
bun run dev   # ou npm run dev → port 5173
```

---

## 📋 Résumé final

| Tâche | Fichier concerné |
|-------|-----------------|
| Modèles | `apps/*/models.py` |
| Serializers (camelCase) | `apps/*/serializers.py` |
| Vues/Endpoints | `apps/*/views.py` |
| Permissions | `apps/users/permissions.py` |
| Routes | `config/urls.py` + `apps/*/urls.py` |
| Auth JWT | `apps/users/views.py` (LoginView) |
| Email maintenance | `apps/maintenances/views.py` |
| Upload PDF | `apps/maintenances/views.py` |
| Branchement front | `src/services/api.ts` + `src/contexts/AuthContext.tsx` |

