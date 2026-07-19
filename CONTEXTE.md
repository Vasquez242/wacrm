# CONTEXTE TECHNIQUE ET ARCHITECTURAL — WACRM

Ce document sert de spécification technique de référence, de mémoire technique globale et de guide d'onboarding complet pour le projet **WACRM**. Il est conçu pour les développeurs humains (du junior à l'architecte) ainsi que pour les assistants de programmation par IA (Claude Code, Cursor, ChatGPT, RooCode, etc.).

---

## TABLE DES MATIÈRES

1. [Vue d'ensemble](#1-vue-densemble)
2. [Architecture complète](#2-architecture-complète)
3. [Stack technique](#3-stack-technique)
4. [Arborescence complète](#4-arborescence-complète)
5. [Fonctionnalités métier clés](#5-fonctionnalités-métier-clés)
6. [Documentation complète des fonctions critiques](#6-documentation-complète-des-fonctions-critiques)
7. [Documentation des composants stratégiques](#7-documentation-des-composants-stratégiques)
8. [Documentation des hooks React](#8-documentation-des-hooks-react)
9. [Documentation des services clés](#9-documentation-des-services-clés)
10. [Documentation de l'API REST v1](#10-documentation-de-lapi-rest-v1)
11. [Documentation complète de la base de données](#11-documentation-complète-de-la-base-de-données)
12. [Authentification et Sécurité](#12-authentification-et-sécurité)
13. [Variables d'environnement](#13-variables-denvironnement)
14. [Déploiement et Infrastructure](#14-déploiement-et-infrastructure)
15. [Sécurité OWASP & Mesures défensives](#15-sécurité-owasp--mesures-défensives)
16. [Performance & Optimisations](#16-performance--optimisations)
17. [Dette technique & Limitations connues](#17-dette-technique--limitations-connues)
18. [Workflow Développeur](#18-workflow-développeur)
19. [Workflow Git](#19-workflow-git)
20. [FAQ d'intégration](#20-faq-dintégration)
21. [Glossaire](#21-glossaire)
22. [Annexes](#22-annexes)

---

## 1. VUE D'ENSEMBLE

### Nom du projet
**WACRM** (WhatsApp Customer Relationship Manager).

### Objectif et Vision
WACRM est un modèle (template) de CRM auto-hébergable (self-hostable) et marque blanche dédié à WhatsApp Business. Son objectif est de fournir une alternative robuste, multi-agent et sans abonnement par siège aux solutions SaaS propriétaires (type ManyChat, Sirena, Wati) en s'appuyant directement sur l'API WhatsApp Cloud officielle de Meta et sur la puissance de calcul/stockage de Supabase.

### Proposition de valeur
- **Souveraineté des données** : Hébergement autonome des conversations, des contacts et des fichiers multimédias sur votre propre instance Supabase.
- **Zéro coût par agent** : Pas de tarification par utilisateur/siège ; l'infrastructure coûte le prix brut des ressources (Supabase + Meta Cloud API).
- **Intégration d'Agents IA** : RAG local avec pgvector et auto-reply intelligent sans surcoût d'intermédiaire.
- **Customisation maximale** : Code-first, modulaire, conçu pour être cloné, modifié, rebrandé pour des besoins d'entreprise spécifiques.

### Utilisateurs cibles
- **Équipes de support client** : Gestion des messages multi-agents, assignations et notes privées.
- **Équipes Growth & Marketing** : Envoi de broadcasts (campagnes de masse) basés sur des templates approuvés par Meta avec substitution de variables personnalisées par destinataire.
- **Équipes Opérations & Produit** : Automatisation de flux conversationnels (chatbot procédural à graphes) et synchronisation de webhooks sortants.

### Périmètre fonctionnel
- **Shared Inbox** : Messagerie bidirectionnelle en temps réel avec indicateurs de présence des agents, indicateurs de lecture, support des médias, notes internes privées, et réponses rapides.
- **Gestion des Contacts** : CRM léger avec attributs personnalisés (custom fields), étiquettes (tags) avec prévention des boucles infinies, et déduplication automatique par numéro de téléphone.
- **Pipelines Kanban** : Suivi visuel des transactions (deals) et étapes configurables liées à des fils de discussion.
- **Broadcasts** : Planification de campagnes massives avec ré-agrégation d'état O(1) (Sent, Delivered, Read, Failed) et gestion du budget d'envoi.
- **Moteur d'automations** : Déclencheurs événementiels (Inbound, Contact créé, Tag ajouté) exécutant un arbre d'actions conditionnelles.
- **Moteur de Flows** : Chatbot procédural modélisable dans un éditeur visuel basé sur React Flow et Dagre.
- **AI RAG & Assistant** : Système d'auto-réponse basé sur des documents de connaissances indexés avec recherche hybride (lexicale + vectorielle) et handoff intelligent vers un agent humain.

---

## 2. ARCHITECTURE COMPLÈTE

WACRM est conçu comme un **monolithe modulaire** intégrant des Route Handlers Next.js agissant comme des contrôleurs légers et des modules métier découplés vivant dans `src/lib/`. La persistance, la sécurité d'accès multi-tenant (RLS) et la publication en temps réel sont déléguées à Supabase.

### Diagramme d'architecture logique et flux de données

```mermaid
graph TD
    %% Clients
    Browser[Navigateur Client / SPA]
    MCP[Client MCP / Claude / Cursor]
    MetaAPI[Meta Cloud API Webhook]
    
    %% Gateway / Middleware
    Middleware[Next.js Middleware.ts]
    
    %% Application Layer
    API_V1[API REST Publique /api/v1/*]
    API_Internal[API Interne /api/whatsapp/*]
    Auth_Context[React AuthContext / useAuth]
    
    %% Business Domain Modules (src/lib/)
    WA_Send[WhatsApp Outbound Orchestrator]
    Flow_Engine[Flow Node Engine]
    Auto_Engine[Automation Engine]
    AI_Module[AI Reply & RAG Engine]
    Contact_Dedupe[Contact Deduplication]
    Webhook_Deliver[Outbound Webhooks Sign & SSRF]
    
    %% Database Layer (Supabase)
    Supa_Auth[Supabase Auth]
    Supa_DB[(Supabase Postgres Database)]
    Supa_Storage[(Supabase Storage Buckets)]
    
    %% Connections
    Browser -->|JWT Cookie| Middleware
    Middleware -->|Vérifié| Auth_Context
    Auth_Context --> API_Internal
    
    MCP -->|API Key Authorization Header| API_V1
    API_V1 -->|Service Role Client| WA_Send
    
    MetaAPI -->|Post Request & HMAC Sign| API_Internal
    API_Internal -->|Webhook Receiver| WA_Send
    
    %% Logic Dispatches
    API_Internal -->|Dispatch| Flow_Engine
    API_Internal -->|Dispatch| Auto_Engine
    API_Internal -->|Dispatch| AI_Module
    
    WA_Send --> Contact_Dedupe
    WA_Send -->|Meta API Call| MetaAPI
    
    Flow_Engine --> Supa_DB
    Auto_Engine --> Supa_DB
    AI_Module -->|Vector Index Search| Supa_DB
    
    Supa_Auth -->|Trigger handle_new_user| Supa_DB
```

### Modèle d'accès multi-tenant (Tenancy)

Le projet utilise un modèle de colocation étanche basé sur la colonne `account_id` présente sur toutes les tables opérationnelles de domaine. 

```mermaid
sequenceDiagram
    autonumber
    actor Client as Client / Agent
    participant Route as Route Handler (API)
    participant Lib as lib/auth/api-context
    participant DB as Supabase DB (Postgres RLS)
    
    Client->>Route: Requête (ex. GET /api/v1/contacts) avec Bearer API Key
    Route->>Lib: requireApiKey(request, 'contacts:read')
    Lib->>DB: Recherche clé active (Key Hash SHA-256)
    DB-->>Lib: Retourne account_id + scopes associés
    Note over Lib: Vérification du scope 'contacts:read' dans le tableau
    Lib-->>Route: Retourne Context (supabaseAdminClient + accountId)
    Route->>DB: SELECT * FROM contacts WHERE account_id = accountId
    DB-->>Route: Liste des contacts isolés du locataire
    Route-->>Client: Réponse JSON enveloppée { data: [...] }
```

---

## 3. STACK TECHNIQUE

| Technologie | Version | Usage | Pourquoi | Alternatives envisagées |
|---|---|---|---|---|
| **Next.js** | `16.2.6` | Framework Full-stack, App Router, ISR | SSR natif, middleware performant, support de `after()` | Remix, Vite SPA |
| **React** | `19.2.4` | Bibliothèque UI | Requis par Next.js 16, support expérimental optimisé | Vue.js, Svelte |
| **Tailwind CSS** | `^4.0.0` | Framework utilitaire de style | Rapidité de développement, thémage natif en variables CSS | CSS Modules, Sass |
| **Supabase Client** | `^2.107` | Client de base de données / RLS | Backend-as-a-Service, Auth intégrée, Realtime par CDC Postgres | Firebase, Prisma + Postgres hébergé |
| **Supabase SSR** | `^0.12.0` | Gestion de session sur le serveur | Synchronisation robuste des cookies de session client-serveur | Auth0, NextAuth.js |
| **@xyflow/react** | `^12.11.0` | Rendu de graphes de nœuds | Rendu robuste et fluide du concepteur visuel de chatbot | Custom SVG Canvas |
| **@dagrejs/dagre** | `^3.0.0` | Positionnement automatique de graphes | Calcul algorithmique des coordonnées des nœuds (Dagre layout) | Graphviz |
| **dnd-kit** | `^6.3.1` | Drag & Drop dans les pipelines | Performance sur mobile et accessibilité pour le tableau Kanban | React Beautiful DND (déprécié) |
| **next-intl** | `^4.13.2` | Internationalisation (i18n) | Support natif de Next.js App Router, gestion de namespaces | react-i18next |
| **Vitest** | `^4.1.10` | Lanceur de tests unitaires | Vitesse d'exécution en environnement Node pur, intégration TS | Jest |
| **opus-recorder** | `^8.0.5` | Enregistrement de notes vocales | Encodage OGG/Opus côté client compatible WhatsApp Business API | Web Audio API brute |

---

## 4. ARBORESCENCE COMPLÈTE

### Organisation globale des dossiers

```
.
├── .github/                 # Workflows CI, templates d'issues et de Pull Requests
├── docs/                    # Documentation utilisateur et développeur (API, MCP)
├── mcp-server/              # Code source du serveur MCP autonome (wacrm-mcp)
├── messages/                # Fichiers de localisation i18n (en.json, ko.json)
├── public/                  # Assets statiques, scripts et encodeur audio Opus
├── supabase/                # Code source de la DB (migrations SQL)
└── src/                     # Code source de l'application Next.js
    ├── app/                 # Routes Next.js, Layouts et Route Handlers API
    ├── components/          # Composants React de l'interface (par domaine)
    ├── hooks/               # Hooks React transversaux (auth, realtime, presence)
    ├── i18n/                # Initialisation et configuration next-intl
    ├── lib/                 # Logique métier et interfaces de services (par domaine)
    └── types/               # Single source of truth des types TS de la DB et de l'app
```

### Analyse détaillée des fichiers structurants

#### `src/middleware.ts`
Intercepte chaque requête Next.js. Rôle :
1. **Refresh de token Supabase SSR** : Assure que le jeton de session est rafraîchi en toute sécurité et que les en-têtes `Set-Cookie` réinjectés par la rotation de jetons ne sont pas perdus lors des redirections ou des réponses JSON (mitige le syndrome du "session wedge").
2. **Garde de routes de l'application** : Redirige vers `/login` pour les pages protégées (`/dashboard`, `/inbox`, `/settings`, etc.) si aucune session n'est active.
3. **Garde des APIs protégées** : Renvoie une erreur `401 Unauthorized` pour les requêtes vers `/api/whatsapp/*` (hors `/webhook` public) si l'utilisateur n'est pas connecté.

#### `src/types/index.ts`
Porte l'intégralité des contrats d'interface TypeScript reflétant les schémas de la base de données. Il contient les types exhaustifs pour les entités (ex. `Profile`, `Account`, `Contact`, `Message`, `Broadcast`, `Automation`, `FlowNode`, `QuickReply`) et les enums associés, garantissant le typage statique fort sans ORM.

#### `src/lib/auth/api-context.ts`
Fournit la fonction `requireApiKey` utilisée par tous les contrôleurs de l'API publique `/api/v1/*`. Il valide les en-têtes d'autorisation `Bearer wacrm_live_...`, résout la clé par un hachage SHA-256 en base de données, gère le limiteur de débit lié à la clé, et renvoie un `ApiKeyContext` sécurisé contenant un client d'administration Supabase (RLS-bypassing) restreint logiquement à l'identifiant du compte (`accountId`).

#### `src/lib/whatsapp/send-message.ts`
L'orchestrateur central de l'envoi de messages sortants WhatsApp. Partagé entre l'interface utilisateur de la boîte de réception partagée et les routes de l'API REST publique, il contient la logique complexe de retry sur les variantes de numéros de téléphone et la mise à jour transactionnelle des compteurs et états dans Supabase.

---

## 5. FONCTIONNALITÉS MÉTIER CLÉS

### 5.1 Shared Inbox (Boîte de réception partagée)
Permet à plusieurs agents humains de lire et d'envoyer des messages WhatsApp en temps réel, de s'assigner des conversations, de réagir avec des emojis et de laisser des notes privées sur les fiches des contacts.
- **Déclencheur** : Réception d'un message Meta Cloud API (inbound) ou action manuelle d'un agent (outbound).
- **Sécurité** : RLS opérationnelle — lecture pour tous les membres du compte, écriture réservée aux rôles `agent`, `admin` et `owner`.
- **Fichiers clés** :
  - Composants : `src/components/inbox/message-composer.tsx`, `src/components/inbox/message-thread.tsx`, `src/components/inbox/conversation-list.tsx`
  - Métier : `src/lib/whatsapp/send-message.ts`

### 5.2 Moteur de chatbot (Flow Engine)
Exécute des arbres de décision complexes (nœuds interactifs, boutons, listes de choix, délais, conditions logiques) créés visuellement.
- **Déclencheur** : Webhook WhatsApp inbound avec analyse de mot-clé ou premier message entrant, ou avancement via un clic sur un bouton interactif.
- **Validation** : Protection contre l'avancement simultané (Optimistic Lock) et garde unique d'une exécution active par contact (`idx_one_active_run_per_contact`).
- **Fichiers clés** :
  - Composants : `src/components/flows/flow-canvas.tsx`, `src/components/flows/flow-builder.tsx`
  - Métier : `src/lib/flows/engine.ts`, `src/lib/flows/fallback.ts`

### 5.3 Moteur d'automations (Automation Engine)
Exécute des automations linéaires et structurées de type Trigger-Action (ex. "Quand un contact est créé, envoyer le template X et attendre 5 minutes").
- **Déclencheur** : Événements système (`new_contact_created`, `first_inbound_message`, `tag_added`, `keyword_match`).
- **Persistance** : Les exécutions différées (Wait steps) sont stockées dans `automation_pending_executions` et purgées par un cron régulier.
- **Fichiers clés** :
  - Métier : `src/lib/automations/engine.ts`, `src/app/api/automations/cron/route.ts`

### 5.4 AI Auto-reply & RAG
Permet au système d'analyser le message entrant, de rechercher des correspondances sémantiques (pgvector) ou lexicales (Postgres FTS) dans la base de connaissances interne, et de générer une réponse via l'API OpenAI ou Anthropic de l'utilisateur.
- **Déclencheur** : Message inbound non consommé par le moteur de flows.
- **Sécurité** : Protection stricte contre les fuites de données (les fonctions SQL d'appariement sont en `SECURITY INVOKER` pour forcer le filtrage RLS sur le compte). Débrayage immédiat de l'IA (handoff) dès qu'un agent humain répond ou s'assigne le fil de discussion.
- **Fichiers clés** :
  - Métier : `src/lib/ai/auto-reply.ts`, `src/lib/ai/knowledge.ts`

---

## 6. DOCUMENTATION COMPLÈTE DES FONCTIONS CRITIQUES

### 6.1 `sendMessageToConversation`

#### Signature
```typescript
export async function sendMessageToConversation(
  db: SupabaseClient,
  accountId: string,
  params: SendMessageParams
): Promise<SendMessageResult>
```

#### Description & Utilité
Cette fonction orchestre l'envoi de messages WhatsApp à un destinataire (texte, média, template interactif), gère le format du numéro de téléphone, résout les parents de réponse, met à jour le fil de discussion et suspend le cas échéant le chatbot actif de l'interlocuteur pour laisser la main à l'agent humain.

#### Algorithme détaillé
1. **Validation syntaxique** : Appel à `validateSendMessageParams` pour vérifier la présence des champs requis selon le type de message (ex: `mediaUrl` obligatoire si type image/audio).
2. **Extraction de la conversation** : Requête sur la table `conversations` jointe avec `contacts` filtrée par `id` et `account_id` (tenancy).
3. **Validation du numéro de téléphone** : Nettoyage et vérification de la conformité E.164.
4. **Récupération des identifiants et tokens Meta** : Requête sur `whatsapp_config` pour récupérer le `phone_number_id` et l' `access_token` déchiffré via AES-256-GCM.
5. **Tentative d'envoi avec auto-correction** :
   - Génère les variantes du numéro de téléphone (ex: sans indicatif, indicatif répété).
   - Boucle sur les variantes. Si Meta renvoie une erreur *Recipient Not In Allowed List*, essaie la variante suivante.
   - Dès qu'une variante réussit, met à jour de manière asynchrone le numéro du contact en base de données.
6. **Enregistrement du message** : Insertion de la ligne correspondante dans `messages` avec le statut `'sent'` et l'identifiant unique fourni par Meta (`message_id`).
7. **Mise à jour de la conversation** : Met à jour la colonne `last_message_text` et la date du dernier message.
8. **Pause du robot** : Force le statut de toute exécution en cours dans `flow_runs` pour ce contact à `'paused_by_agent'`.

#### Exemple d'utilisation
```typescript
const result = await sendMessageToConversation(supabase, accountId, {
  conversationId: '47d79b9a-4c28-4ec2-be75-d14fbfdf123a',
  messageType: 'text',
  contentText: 'Bonjour ! Comment puis-je vous aider ?'
});
// result => { messageId: '...', whatsappMessageId: 'wamid.HBgLMzM2M...' }
```

---

### 6.2 `dispatchInboundToFlows`

#### Signature
```typescript
export async function dispatchInboundToFlows(input: {
  accountId: string
  userId: string
  contactId: string
  conversationId: string
  message: ParsedInbound
  isFirstInboundMessage: boolean
}): Promise<DispatchInboundResult>
```

#### Description & Utilité
Point d'entrée du moteur de flows lors de la réception d'un message client. Il vérifie s'il existe une session de chatbot en cours pour ce contact afin de lui transmettre l'entrée utilisateur, ou si le message correspond à un déclencheur de mot-clé configuré pour lancer un nouveau robot.

#### Algorithme détaillé
1. **Établissement du verrou de concurrence** : Recherche de toute ligne active dans `flow_runs` pour l' `account_id` et le `contact_id` donnés.
2. **Scénario A : Une session est active** :
   - Récupère le nœud actuel (`current_node_key`) dans `flow_nodes`.
   - Si le message reçu porte le même identifiant Meta (`meta_message_id`) qu'un événement déjà traité, interrompt l'exécution (protection contre les doublons Meta).
   - Valide la réponse de l'utilisateur contre les options du nœud (comparaison d'identifiants de boutons interactifs ou expression régulière textuelle).
   - Si valide, avance au nœud cible et exécute les effets secondaires (ex: ajout d'étiquettes, écriture de champs personnalisés).
   - Si invalide, applique la politique de repli (reprompt, relance, ou sortie vers agent humain).
3. **Scénario B : Aucune session n'est active** :
   - Recherche les déclencheurs de type *Mot-clé* ou *Premier message entrant* sur les flows configurés comme `'active'`.
   - Si match, instancie une ligne dans `flow_runs` (les conflits d'insertion parallèles sont capturés via l'index unique Postgres et résolus par un abandon propre).
   - Exécute le nœud de départ (`start`) et progresse automatiquement sur les nœuds auto-exécutants (envoi de texte, branche conditionnelle) jusqu'au premier nœud d'attente d'entrée utilisateur (`collect_input`, `send_buttons`, `send_list`).
4. **Retour** : Renvoie un booléen indiquant si l'entrée a été consommée par le moteur de flows (bloquant l'exécution ultérieure des réponses automatiques par IA).

---

## 7. DOCUMENTATION DES COMPOSANTS STRATÉGIQUES

### 7.1 `FlowCanvas`
Composant d'affichage interactif du graphe de nœuds sous-jacent à un flow.
- **Rôle** : Rendre visuellement l'arborescence des actions et choix du chatbot, gérer le glisser-déposer de nœuds, et lier les connexions logiques.
- **Librairies majeures** : `@xyflow/react` pour le canvas, `@dagrejs/dagre` pour le calcul de placement automatique des nœuds.
- **Fichier** : `src/components/flows/flow-canvas.tsx`

### 7.2 `MessageComposer`
Barre d'action et éditeur de messages sortants pour les agents.
- **Rôle** : Saisie de messages textuels, pièces jointes multimédias, sélection de snippets de réponses rapides (`quick_replies`), prévisualisation de templates de messages officiels Meta, et génération automatique de brouillons par IA (bouton ✨).
- **Fichier** : `src/components/inbox/message-composer.tsx`

---

## 8. DOCUMENTATION DES HOOKS REACT

### 8.1 `useAuth`
Context Provider centralisant l'identité, le compte et les habilitations de l'agent.
- **Propriétés retournées** :
  - `user` : Objet utilisateur authentifié Supabase Auth.
  - `profile` : Ligne correspondante de la table `profiles` (contient le rôle applicatif).
  - `accountId` : Identifiant UUID du compte tenant.
  - `accountRole` : Rôle (`owner`, `admin`, `agent`, `viewer`).
  - `canSendMessages`, `canEditSettings`, `canManageMembers` : Habilités déduites selon le rôle.
- **Fichier** : `src/hooks/use-auth.tsx`

### 8.2 `useRealtime`
Abonnement générique aux canaux de communication en temps réel Supabase.
- **Logique** : Gère la création, la souscription et la désinscription sécurisée d'un `RealtimeChannel` à la volée. Stabilise les références mémoire des callbacks pour éviter les reconnexions infinies induites par le cycle de rendu de React 19.
- **Fichier** : `src/hooks/use-realtime.ts`

---

## 9. DOCUMENTATION DES SERVICES CLÉS

### 9.1 `MetaApi` (`src/lib/whatsapp/meta-api.ts`)
Encapsule les appels directs vers l'API WhatsApp Graph de Meta.
- **Méthodes** :
  - `sendTextMessage` : Envoi de texte brut.
  - `sendTemplateMessage` : Envoi de messages pré-approuvés.
  - `sendMediaMessage` : Envoi d'images, vidéos, audios et documents.
  - `sendInteractiveButtons` / `sendInteractiveList` : Envoi de messages interactifs structurés.
  - `getMediaUrl` / `downloadMedia` : Résolution et téléchargement de pièces jointes Meta stockées temporairement chez Meta.

### 9.2 `AI Engine` (`src/lib/ai/generate.ts`)
Gère l'interfaçage avec les APIs de modèles de langage (LLM) configurés par l'utilisateur.
- **Méthodes** :
  - `generateReply` : Envoie l'historique de discussion et le contexte récupéré au provider sélectionné (OpenAI ou Anthropic) en injectant les clés d'API configurées (déchiffrées à la volée sur le serveur).

---

## 10. DOCUMENTATION DE L'API REST V1

L'API publique exposée sous `/api/v1/*` permet la manipulation programmatique des entités du CRM à l'aide d'une clé API Bearer. Toutes les requêtes sont formatées en JSON enveloppé et utilisent une pagination par curseur opaque (`cursor`).

### Liste des Endpoints et Habilitations requises

| Méthode | Route | Rôle / Scope | Description |
|---|---|---|---|
| **GET** | `/api/v1/me` | Aucun (clé valide) | Retourne les métadonnées de la clé API et le compte associé |
| **GET** | `/api/v1/contacts` | `contacts:read` | Recherche et liste les contacts du compte |
| **POST** | `/api/v1/contacts` | `contacts:write` | Crée ou résout un contact par numéro de téléphone |
| **PATCH** | `/api/v1/contacts/{id}` | `contacts:write` | Met à jour les informations d'un contact existant |
| **GET** | `/api/v1/conversations` | `conversations:read` | Liste les conversations actives avec tris et filtres |
| **POST** | `/api/v1/messages` | `messages:send` | Envoie un message WhatsApp sortant à un numéro de téléphone |
| **POST** | `/api/v1/broadcasts` | `broadcasts:send` | Lance une campagne de broadcast vers une liste de destinataires |

### Exemples d'appels API

#### 1. Envoi de message (`POST /api/v1/messages`)
- **Headers** :
  ```http
  Authorization: Bearer wacrm_live_58c279c7820da...
  Content-Type: application/json
  ```
- **Requête (Payload)** :
  ```json
  {
    "to": "+33612345678",
    "type": "text",
    "text": "Bonjour, votre commande a été expédiée."
  }
  ```
- **Réponse (Succès 200 OK)** :
  ```json
  {
    "data": {
      "message_id": "8a719d9b-1cc8-4e89-b2c3-c283624e58a2",
      "whatsapp_message_id": "wamid.HBgLMzM2MTIzNDU2NzgVAgIGFjEzRDRD..."
    }
  }
  ```

---

## 11. DOCUMENTATION COMPLÈTE DE LA BASE DE DONNÉES

Toutes les requêtes de l'application s'appuient sur PostgREST nativement exposé par Supabase. L'isolement multi-tenant est assuré au niveau du serveur de base de données par l'application de politiques RLS.

### Schéma Entité-Association (diagramme partiel)

```mermaid
erDiagram
    accounts ||--o{ profiles : "contient"
    accounts ||--o{ contacts : "possède"
    accounts ||--o{ whatsapp_config : "configure"
    contacts ||--o{ conversations : "ouvre"
    conversations ||--o{ messages : "contient"
    messages ||--o{ message_reactions : "reçoit"
    contacts ||--o{ contact_tags : "étiqueté"
    tags ||--o{ contact_tags : "définit"
    
    profiles {
        uuid id PK
        uuid user_id FK
        uuid account_id FK
        account_role_enum account_role
        string full_name
        string email
    }
    
    contacts {
        uuid id PK
        uuid account_id FK
        string phone
        string phone_normalized
        string name
    }
    
    conversations {
        uuid id PK
        uuid account_id FK
        uuid contact_id FK
        string status
        uuid assigned_agent_id
    }
    
    messages {
        uuid id PK
        uuid conversation_id FK
        string sender_type
        string content_type
        string content_text
        string message_id
        string status
    }
```

### Principales politiques RLS et Sécurité

Toutes les tables intègrent un contrôle de type :
```sql
ALTER TABLE contacts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Access by account members" ON contacts
    FOR ALL
    TO authenticated
    USING (is_account_member(account_id, 'viewer'));
```
La fonction SQL `is_account_member(UUID, account_role_enum)` utilise un niveau de privilège `SECURITY DEFINER` pour lire la table `profiles` sans s'exposer à des récursions infinies d'évaluation de politique RLS.

---

## 12. AUTHENTIFICATION ET SÉCURITÉ

La gestion de session s'appuie sur le moteur d'authentification natif de Supabase (Supabase Auth) par l'émission de jetons JWT stockés sous forme de cookies sécurisés.

```mermaid
sequenceDiagram
    autonumber
    actor User as Agent (Navigateur)
    participant App as Next.js App Router
    participant Middleware as middleware.ts
    participant SupaAuth as Supabase Auth Server

    User->>App: Requête vers une page sécurisée (/inbox)
    App->>Middleware: Intercepte la requête
    Middleware->>SupaAuth: supabase.auth.getUser() (Transmet les cookies)
    alt Token expiré mais Refresh Token valide
        SupaAuth-->>Middleware: Renvoie un nouveau token d'accès (Rotation)
        Note over Middleware: Copie du nouveau Set-Cookie sur la réponse
    else Session invalide
        Note over Middleware: Redirection forcée vers /login
    end
    Middleware-->>User: Rendu de la page inbox (avec cookies rafraîchis)
```

---

## 13. VARIABLES D'ENVIRONNEMENT

| Variable | Description | Obligatoire | Valeur par défaut | Impact en cas d'absence |
|---|---|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | URL de l'instance API Supabase | **Oui** | (aucune) | Crash au boot de l'application |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY`| Clé publique anonyme Supabase | **Oui** | (aucune) | Impossible de s'authentifier côté client |
| `SUPABASE_SERVICE_ROLE_KEY` | Clé secrète de contournement RLS | **Oui** | (aucune) | Crash des webhooks et des moteurs de background |
| `ENCRYPTION_KEY` | Clé hexagonale de 64 caractères (32 octets) | **Oui** | (aucune) | Impossible de lire les jetons Meta et clés d'APIs |
| `META_APP_SECRET` | Secret d'application Meta WhatsApp | **Oui** | (aucune) | Rejet systématique des messages entrants (signature KO) |
| `AUTOMATION_CRON_SECRET` | Clé secrète protégeant la purge cron | Non | (aucune) | Risque d'exécution frauduleuse de la route de purge |
| `NEXT_PUBLIC_APP_LOCALE` | Code de langue par défaut de l'interface | Non | `en` | Traduction initialisée en anglais |

---

## 14. DÉPLOIEMENT ET INFRASTRUCTURE

### Processus de Build
Le build est entièrement standardisé pour les environnements de conteneurs Node.js :
1. Analyse statique : `npm run lint` et `npm run typecheck`.
2. Tests unitaires automatisés : `npm test`.
3. Compilation Next.js : `npm run build` (génère le dossier de sortie optimisé `.next/`).

### Intégration Continue (GitHub Actions)
Le fichier `.github/workflows/ci.yml` automatise la validation sur chaque branche :
- Déclenchement automatique sur pull request ou push vers la branche principale `main`.
- Montage d'un environnement Node 20 sous Ubuntu, exécution de `npm ci`, validation du typage, exécution des tests Vitest, et tentative de build Next.js avec injection de clés factices sécurisées.

---

## 15. SÉCURITÉ OWASP & MESURES DÉFENSIVES

- **Chiffrement des secrets en DB** : Les secrets tiers (ex. `access_token` WhatsApp, clés API d'intégration d'agents IA) ne sont jamais écrits en clair en base de données. Ils sont chiffrés en utilisant l'algorithme AES-256-GCM avec la clé définie dans `ENCRYPTION_KEY` via `src/lib/whatsapp/encryption.ts`.
- **Protection contre le Webhook Spoofing** : Chaque requête entrante sur `/api/whatsapp/webhook` est validée en calculant une signature HMAC-SHA256 à partir du corps brut (`request.text()`) et du secret `META_APP_SECRET`, bloquant les injections de faux payloads.
- **Mitigation des attaques SSRF** : Les webhooks sortants configurés par les utilisateurs sont filtrés via `src/lib/webhooks/ssrf.ts` pour s'assurer qu'aucun appel ne cible une adresse IP privée de la plage locale (ex: `127.0.0.1`, `10.0.0.0/8`, `192.168.0.0/16`), protégeant l'infrastructure réseau interne.

---

## 16. PERFORMANCE & OPTIMISATIONS

- **Pas de jointures SQL à fort coût** : Les lookups d'appartenance sont faits par des appels directs indexés en UUID ou par des requêtes parallélisées pour éviter le blocage du cache de schéma PostgREST de Supabase (ex: résolution asynchrone découplée du profil et du compte dans `AuthProvider`).
- **Verrous optimistes** : Utilisation de clauses restrictives dans les clauses `UPDATE` SQL du moteur de flows (ex: `.eq('current_node_key', expected)`) pour empêcher deux messages concurrents d'avancer un même fil de chatbot en même temps.
- **Indexation vectorielle et lexicale dédiée** : Recherche hybride dans la base de connaissances via un index composite FTS GIN et un index HNSW (`vector_cosine_ops`) sur 1536 dimensions.

---

## 17. DETTE TECHNIQUE & LIMITATIONS CONNUES

- **Rate Limiting mémoire** : Le module `src/lib/rate-limit.ts` stocke les compteurs de requêtes dans une structure `Map` globale en mémoire Node.js. Si l'application est déployée sur plusieurs conteneurs en répartition de charge (clustering horizontal) ou en architecture Serverless pure, le limiteur de débit est divisé et perd son uniformité.
- **Rétrocompatibilité de chiffrement** : Présence de routines de déchiffrement prenant en charge l'ancien format CBC (pour éviter de bloquer les instances historiques lors de la migration vers GCM). Ces routines de rétrocompatibilité méritent d'être nettoyées après s'être assuré du cycle de renouvellement des jetons de toutes les configurations actives.
- **Pas d'upload natif multimédia** : Les envois de fichiers multimédias complexes via l'API publique exigent une URL publique préexistante (Meta télécharge le document depuis cette adresse) au lieu de permettre le téléchargement de fichiers binaires bruts via un flux HTTP multipart.

---

## 18. WORKFLOW DÉVELOPPEUR

### Installation locale
1. Cloner le dépôt et se placer à la racine :
   ```bash
   git clone https://github.com/ArnasDon/wacrm.git
   cd wacrm
   ```
2. Installer les dépendances Node :
   ```bash
   npm install
   ```
3. Configurer l'environnement :
   ```bash
   cp .env.local.example .env.local
   # Remplir les clés Supabase et encryption de démonstration
   ```
4. Lancer le serveur de développement local :
   ```bash
   npm run dev
   ```

### Exécution des tests unitaires
- Exécuter la suite complète via Vitest :
  ```bash
  npm test
  ```
- Exécuter en mode interactif (watch mode) :
  ```bash
  npm run test:watch
  ```

---

## 19. WORKFLOW GIT

- **Branches** : Créer des branches descriptives à partir de `main` (ex. `fix/inbox-unread-count`, `feat/custom-template-header`).
- **Conventions de commit** : Suivre la spécification des commits conventionnels (Conventional Commits) :
  - `feat(...) :` Nouvelle fonctionnalité.
  - `fix(...) :` Correction de bug.
  - `docs(...) :` Modifications de documentation.
  - `test(...) :` Ajout ou modification de tests unitaires.
- **Pull Requests** : Remplir précisément le template de Pull Request incluant le plan de test unitaire et le comportement observé manuellement sur navigateur.

---

## 20. FAQ D'INTÉGRATION

#### Q : Pourquoi mes images WhatsApp entrantes ne s'affichent pas dans l'Inbox partagée ?
**R** : Les en-têtes d'accès à l'API Meta ou l'ordre des arguments de `getMediaUrl` étaient erronés dans les anciennes versions. Assurez-vous que votre clé `access_token` est correctement déchiffrable et que vous êtes aligné sur la version récente du module WhatsApp core.

#### Q : L'index unique Postgres sur `conversations` a-t-il un impact sur la suppression des contacts ?
**R** : Oui, la suppression d'un contact entraîne la mise à `'null'` de la référence `contact_id` sur la table `conversations`. L'index unique composite `(account_id, contact_id)` intègre une clause d'exclusion partielle pour ignorer les valeurs nulles et éviter le blocage de suppression d'un contact orphelin.

---

## 21. GLOSSAIRE

- **WABA** : WhatsApp Business Account. Compte professionnel Meta gérant les numéros de téléphone et les modèles de messages.
- **RLS** : Row Level Security. Dispositif de sécurité natif PostgreSQL permettant de filtrer dynamiquement les lignes retournées par une requête en fonction du contexte de l'utilisateur connecté.
- **RAG** : Retrieval-Augmented Generation. Technique d'IA combinant la recherche documentaire sémantique (via embeddings et vecteurs) et la génération de texte par un modèle de langage (LLM) pour répondre précisément à partir d'un corpus privé.
- **WAMID** : WhatsApp Message ID. Identifiant unique de message généré et garanti par les serveurs de Meta.
- **MCP** : Model Context Protocol. Protocole de communication standardisé permettant à un modèle d'IA d'interagir avec des outils locaux ou distants via une interface stdio sécurisée.

---

## 22. ANNEXES

### Commandes rapides de secours

- **Typecheck complet du projet** :
  ```bash
  npm run typecheck
  ```
- **Validation du formatage du code** :
  ```bash
  npm run format:check
  ```
- **Application manuelle du formatage Prettier** :
  ```bash
  npm run format
  ```
- **Serveur MCP en mode local** :
  ```bash
  npx -y wacrm-mcp
  ```
