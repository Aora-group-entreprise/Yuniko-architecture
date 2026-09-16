# YUNIKO — Carte technique complète (plan de construction, sans code)

> Document d'architecture. Aucune implémentation ici : uniquement structures, flux, fonctions, formules et décisions techniques justifiées.

---

## 0. Principes directeurs

1. **Le client ne décide jamais de la vérité.** Toute donnée sensible (compteurs, droits, classement, distribution) est calculée et validée côté serveur/base.
2. **Optimistic UI partout, réconciliation systématique.** L'utilisateur voit l'effet en < 16 ms, le serveur confirme ou annule.
3. **Événementiel.** Chaque action utilisateur produit un *event* persisté (`events`), qui alimente notifications, algorithme de feed et distribution mondiale. C'est le cœur de la scalabilité.
4. **Dénormalisation contrôlée.** Les compteurs (`like_count`, `comment_count`) sont stockés sur la ligne du post et maintenus par triggers, jamais par `COUNT(*)` en lecture.
5. **Une seule source de vérité de sécurité : la base (RLS).** L'API ne « protège » pas, elle expose ; les politiques de ligne protègent.

---

## 1. STACK TECHNIQUE — pourquoi, où, comment

### 1.1 Langage : TypeScript (client + serveur)

- **Pourquoi** : un seul langage des composants jusqu'aux fonctions serveur ; les types des tables sont générés depuis la base, donc un changement de schéma casse la compilation au lieu de casser la prod.
- **Où** : composants, hooks, server functions, algorithmes de ranking.
- **Alternatives** : Go ou Rust pour le service de ranking (meilleure latence CPU-bound), Python pour l'entraînement de modèles ML.
- **Quand basculer** : quand le ranking dépasse ~30 ms CPU par requête ou qu'on introduit un modèle ML entraîné → extraire un microservice Go/Python.

### 1.2 Framework front : React 19 + TanStack Start (SSR) + Vite

- **Pourquoi React** : écosystème de composants, concurrence (transitions) utile pour un feed infini ; recrutement facile.
- **Pourquoi TanStack Start plutôt qu'un SPA nu** : SSR du feed et des profils = premier rendu rapide + SEO sur les profils publics et les posts publics ; les *server functions* donnent un RPC typé sans écrire d'API REST à la main.
- **Où** : routes (`/`, `/u/$username`, `/p/$postId`, `/messages`, `/explore`), rendu du feed, hydratation.
- **Alternatives** : Next.js (écosystème plus large, vendor-lock Vercel plus fort), Remix, Astro (pour un site à dominante contenu), Expo/React Native (mobile natif, indispensable à terme pour caméra/stories/push).
- **Quand** : dès que les stories et les notifications push deviennent centrales → application React Native partageant les paquets `core/` et `algorithms/`.

### 1.3 État : trois couches distinctes (ne pas mélanger)

| Couche | Technologie | Contenu | Raison |
|---|---|---|---|
| État serveur | TanStack Query | posts, profils, commentaires, conversations | cache, invalidation, `optimistic updates`, pagination infinie, retry — tout est déjà résolu |
| État global client | Zustand | session, brouillons, lecteur de stories, UI globale (modales, thème) | minuscule, sans boilerplate, sélecteurs granulaires → pas de re-render du feed |
| État local | `useState` / `useReducer` | formulaires, hover, focus | inutile de globaliser |

- **Alternatives** : Redux Toolkit (utile si on veut du time-travel debugging et des middlewares d'audit), Jotai/Valtio (atomique), SWR (plus léger que Query mais moins d'outillage mutation).
- **Règle** : un like ne passe **jamais** par Zustand — c'est de l'état serveur, donc cache Query muté optimistiquement.

### 1.4 UI : Tailwind CSS v4 + shadcn/ui + Radix + Framer Motion

- **Tailwind v4** : tokens sémantiques en `oklch` dans un seul fichier ; aucune couleur codée en dur dans les composants.
- **Radix** (sous shadcn) : accessibilité clavier/ARIA des modales, menus, dialogues — coûteux à réécrire.
- **Framer Motion** : transitions de stories, double-tap like, feuilles modales.
- **Alternatives** : CSS Modules + design system maison (contrôle total, plus lent à produire), MUI (opinionné, difficile à singulariser — mauvais choix pour « une identité propre »).

### 1.5 Backend : PostgreSQL managé (Lovable Cloud / Supabase) + server functions

- **Pourquoi Postgres** : transactions ACID (essentiel pour like/unlike et compteurs), `jsonb` pour les payloads d'événements, extensions décisives :
  - `pg_trgm` → recherche floue de pseudonymes,
  - `tsvector`/GIN → recherche plein texte des légendes,
  - `pgvector` → recommandations par similarité d'embeddings (phase tardive),
  - `pg_cron` → jobs de distribution mondiale et de dégradation de scores,
  - `postgis` (optionnel) → géographie.
- **Où** : toutes les tables, RLS, triggers de compteurs, fonctions SQL de ranking.
- **Alternatives** : MongoDB (schéma souple, mais les compteurs et le graphe social souffrent), DynamoDB (scalabilité extrême, mais aucune requête analytique), Cassandra pour les timelines à très grande échelle.
- **Quand** : au-delà de ~10⁷ utilisateurs actifs, on sort les timelines de Postgres vers Redis/Cassandra (fan-out) en gardant Postgres comme registre autoritaire.

### 1.6 Cache et files : Redis

- **Rôle** : timelines pré-calculées (`feed:user:{id}` en liste), rate limiting (token bucket), déduplication d'affichage (set des post_id vus, TTL 7 j), compteurs chauds, présence « en ligne ».
- **Alternatives** : Memcached (pas de structures riches), cache Postgres `unlogged tables` (suffisant en phase 1).

### 1.7 Temps réel : WebSocket via Realtime Postgres (phase 1-3), puis service dédié

- **Rôle** : messages, notifications, typing, présence, compteurs live.
- **Alternatives** : Socket.IO auto-hébergé (contrôle fin des rooms), Ably/Pusher (SLA), SSE (unidirectionnel, suffisant pour les notifications seules).

### 1.8 Médias : stockage objet + CDN + transcodage

- **Upload direct** navigateur → stockage via URL signée (jamais via le serveur applicatif : économie de bande passante et de CPU).
- **Pipeline** : réception → job asynchrone → variantes (thumb 240, feed 1080, original) + AVIF/WebP → `blurhash` pour le placeholder → écriture de `media.status = 'ready'`.
- **Alternatives** : Cloudinary / imgix (transformations à la volée, plus cher), Mux (vidéo/stories à grande échelle).

### 1.9 Observabilité et qualité

Sentry (erreurs), OpenTelemetry (traces), PostHog (analytics produit et A/B testing du ranking), Vitest + Playwright (tests), Zod (validation partagée client/serveur).

---

## 2. ARCHITECTURE GLOBALE

```text
YUNIKO
│
├── Frontend ............ React 19 + TanStack Start (SSR) + Vite + Tailwind
│      rôle : rendu, interactions optimistes, gestion du scroll infini
│      relie : State management, API
│
├── State management .... TanStack Query (serveur) + Zustand (global) + useState (local)
│      rôle : cache, mutations optimistes, invalidations ciblées
│      relie : Components ↔ API ↔ Realtime
│
├── Components .......... shadcn/ui + Radix + Framer Motion
│      rôle : primitives réutilisables (PostCard, StoryRing, ChatBubble)
│      relie : Features (chaque feature compose ses composants)
│
├── Business logic ...... features/*/ (hooks + services) partagés web/mobile
│      rôle : orchestration d'une action utilisateur complète
│      relie : Services → API ; Hooks → State
│
├── Feed algorithm ...... SQL + TypeScript (candidate gen. en SQL, scoring en TS/SQL)
│      rôle : produire une timeline ordonnée personnalisée
│      relie : events, follows, posts, distribution mondiale
│
├── Messaging ........... tables conversations/messages + Realtime + Redis présence
│      rôle : 1-1 et groupes, accusés de lecture, requêtes de message
│
├── Notifications ....... table notifications + triggers + Realtime + push (FCM/APNs)
│      rôle : informer, agréger, respecter les préférences
│
├── Search .............. Postgres tsvector + pg_trgm (phase 1) → Meilisearch/OpenSearch
│      rôle : users, hashtags, légendes, suggestions instantanées
│
├── Authentication ...... Auth managée (JWT + refresh), OAuth Google/Apple, TOTP 2FA
│      rôle : identité, sessions, rôles ; source de auth.uid() pour RLS
│
├── Database ............ PostgreSQL + RLS + triggers + pg_cron + pgvector
│      rôle : vérité unique, intégrité transactionnelle, sécurité par ligne
│
├── API ................. Server functions typées (RPC) + routes publiques /api/public/*
│      rôle : logique privilégiée, webhooks, endpoints cron
│
├── Realtime ............ WebSocket (Postgres Realtime) + canaux par utilisateur
│      rôle : pousser messages, notifications, compteurs
│
└── Security ............ RLS, rate limiting, validation Zod, signatures, audit log,
                          modération (hash perceptuel + classifieur), chiffrement au repos
```

**Relations clés** :
- Components ne parlent **jamais** à la base : ils appellent des hooks de feature, qui appellent des services, qui appellent l'API.
- L'algorithme de feed ne lit **que** des données dénormalisées (`post_stats`, `user_affinity`), jamais des `JOIN` lourds en temps de requête.
- Le module de distribution mondiale est un **consommateur** du flux d'événements et un **producteur** de contraintes de visibilité pour le feed.

---

## 3. MODÈLE DE DONNÉES (tables principales)

```text
users(id, email, phone, created_at, status)                    -- géré par l'auth
profiles(id→users, username UNIQUE CITEXT, display_name, bio, avatar_url,
         is_private, country_code, created_at, search_vector)
follows(follower_id, following_id, status['pending','accepted'], created_at)
         PK(follower_id, following_id)
blocks(blocker_id, blocked_id, created_at) PK(blocker_id, blocked_id)
posts(id, author_id, caption, visibility, created_at, deleted_at,
      like_count, comment_count, save_count, share_count, view_count)
post_media(id, post_id, url, width, height, blurhash, position, status)
likes(user_id, post_id, created_at) PK(user_id, post_id)
comments(id, post_id, author_id, parent_id, body, like_count, created_at, deleted_at)
saves(user_id, post_id, collection_id, created_at) PK(user_id, post_id)
shares(id, post_id, user_id, channel, created_at)
stories(id, author_id, media_url, created_at, expires_at, visibility)
story_views(story_id, viewer_id, viewed_at) PK(story_id, viewer_id)
conversations(id, type['dm','group'], created_at, last_message_at)
conversation_members(conversation_id, user_id, role, joined_at,
                     last_read_message_id, is_archived, is_request)
messages(id, conversation_id, sender_id, body, media_url, reply_to_id,
         created_at, deleted_at)
notifications(id, recipient_id, actor_id, type, entity_type, entity_id,
              group_key, count, is_read, created_at)
events(id, user_id, post_id, type, weight, country_code, session_id,
       dwell_ms, created_at)                                   -- append-only, partitionné
post_stats(post_id, impressions, likes, comments, saves, shares,
           completion_rate, score, updated_at)
post_distribution(post_id, stage, countries[], last_eval_at,
                  impressions_at_stage, engagement_rate, velocity, status)
user_affinity(user_id, target_user_id, score, updated_at)
user_topic_affinity(user_id, topic_id, score)
seen_posts(user_id, post_id, seen_at)                          -- ou Redis TTL
reports(id, reporter_id, entity_type, entity_id, reason, status)
login_events(id, user_id, ip, user_agent, country, is_new_device, created_at)
```

**Index critiques** : `posts(author_id, created_at DESC)`, `events(post_id, created_at)`, `follows(following_id)` (pour la liste des followers), `profiles USING GIN(search_vector)`, `profiles USING GIN(username gin_trgm_ops)`, `messages(conversation_id, created_at DESC)`.

---

## 4. FONCTION PAR FONCTION

Chaque fonctionnalité suit le même canevas : **langage / framework / lib · données · état · fonction principale · fonctions secondaires · événements · logique clic · re-clic · erreurs · sécurité · notifications · effet feed · exemple**.

---

### 4.1 AUTHENTICATION — Register

- **Tech** : TypeScript, React Hook Form + Zod (validation partagée), auth managée (JWT + refresh token httpOnly).
- **Données** : `users`, `profiles` (créé par trigger `on_auth_user_created`).
- **État** : `status: 'idle'|'submitting'|'awaiting_confirmation'|'error'`, jamais « connecté » après `signUp` (la session est nulle tant que l'email n'est pas confirmé).
- **Fonction principale** : `registerUser({email, password, username})`.
- **Secondaires** : `checkUsernameAvailability(username)` (debounce 400 ms, requête indexée), `validatePasswordStrength`, `createProfileFromTrigger`, `sendConfirmationEmail`.
- **Événements** : `auth.user.created`, `profile.created`.
- **Erreurs** : email déjà pris → message neutre (« si ce compte existe, un email a été envoyé ») pour éviter l'énumération de comptes ; username pris → erreur explicite (public de toute façon).
- **Sécurité** : rate limit 5 tentatives / IP / heure ; hash Argon2id géré par l'auth ; unicité `username` en base et non en applicatif (course concurrente).
- **Feed** : nouvel utilisateur → *cold start* : feed peuplé par Discovery (posts populaires de son pays) jusqu'à 5 follows.
- **Exemple** : `Utilisateur → clic "Créer un compte" → registerUser() → status='awaiting_confirmation' → event auth.user.created → email de confirmation → profil créé en base → aucun feed personnalisé avant premier follow`.

### 4.2 Login

- `signIn({identifier, password})` — l'identifiant peut être un pseudonyme, mais **jamais** résolu côté client (fuite d'emails) : une fonction serveur `resolve_login_identifier` fait la correspondance.
- **Secondaires** : `startSession`, `registerDevice`, `detectNewDevice(ip, ua)`, `requireTwoFactorIfEnabled`.
- **Événements** : `auth.login.success`, `auth.login.failed`, `auth.login.new_device`.
- **Sécurité** : verrouillage progressif (backoff exponentiel), CAPTCHA après 3 échecs, journalisation dans `login_events`.
- **Notification** : « Nouvelle connexion depuis Antananarivo, Chrome/Android » si `is_new_device`.

### 4.3 Logout

- `signOut({scope: 'current'|'all'})` → révoque le refresh token, vide le cache Query (`queryClient.clear()`), ferme les canaux Realtime, supprime le token push de l'appareil. Oublier ce dernier point = notifications envoyées à un appareil déconnecté (fuite de vie privée).

### 4.4 Profile / Edit profile

- **Données** : `profiles`. **Lecture** : SSR pour SEO (`/u/$username`) avec `head()` dynamique.
- **Fonction principale** : `updateProfile(patch)` ; **secondaires** : `uploadAvatar` (URL signée + recadrage client), `changeUsername` (vérifie unicité + historique pour éviter le squatting), `togglePrivateAccount`.
- **Effet de bord majeur** : passer en privé **ne** rétracte pas les followers existants ; les nouvelles demandes deviennent `pending`.
- **Sécurité RLS** : `UPDATE ... USING (id = auth.uid())`.

### 4.5 Follow / Unfollow / Followers / Following

- **Données** : `follows(status)`. Compte privé → `status='pending'` → notification « demande d'abonnement » avec actions Accepter/Refuser.
- **Principale** : `toggleFollow(targetId)` ; **secondaires** : `acceptFollowRequest`, `rejectFollowRequest`, `removeFollower`, `listFollowers(cursor)`, `listFollowing(cursor)`.
- **État** : mutation optimiste sur la clé `['profile', targetId]` ; rollback si erreur.
- **Compteurs** : `follower_count` / `following_count` maintenus par trigger, pas par `COUNT(*)`.
- **Événements** : `follow.created`, `follow.accepted`, `follow.removed`.
- **Feed** : `follow.created` déclenche un **backfill** — insertion des 10 derniers posts du créateur dans les candidats, et `user_affinity` initialisée à 0.5.
- **Sécurité** : impossible de suivre quelqu'un qui vous a bloqué (contrainte vérifiée par policy, pas par le client).
- **Exemple** : `Alice → clic Suivre (Bob, privé) → toggleFollow → follows(status=pending) → event follow.requested → notification à Bob → Bob accepte → status=accepted → posts de Bob entrent dans les candidats d'Alice`.

### 4.6 Posts — Create / Edit / Delete

**Create**
- **Flux** : `createPostDraft()` (local, Zustand) → sélection médias → compression client (`browser-image-compression`) → `requestUploadUrls()` → upload direct parallèle → `createPost({caption, mediaIds, visibility})` en transaction → job de transcodage → `posts.status='ready'`.
- **Secondaires** : `extractHashtags`, `extractMentions`, `detectLanguage`, `generateBlurhash`, `moderateMedia`.
- **Événements** : `post.created` → mentions notifiées, hashtags indexés, `post_distribution` initialisée à **stage 1 (3 pays)**.
- **Erreurs** : upload interrompu → reprise par morceaux ; publication en échec → brouillon conservé localement (IndexedDB), jamais perdu.
- **Feed** : le post n'entre que dans les candidats des followers + des 3 pays initiaux.

**Edit** : seule la légende (et les mentions/hashtags dérivés) est modifiable ; les médias ne le sont pas (invalide le cache CDN et l'historique de modération). `post_edits` conserve l'historique.

**Delete** : *soft delete* (`deleted_at`), purge physique après 30 j par `pg_cron`. Les compteurs des autres entités sont ajustés par trigger ; les notifications liées sont supprimées.

### 4.7 LIKE / UNLIKE (référence détaillée)

- **Langage/framework** : TypeScript, React, TanStack Query (mutation), Framer Motion (animation du cœur), Postgres (table + trigger).
- **Structure de données** : `likes(user_id, post_id, created_at)` avec clé primaire composite — l'unicité est garantie par la base, jamais par le client. Compteur dénormalisé `posts.like_count`.
- **État nécessaire** : `{ isLiked: boolean, likeCount: number, pending: boolean }`, dérivé du cache Query `['post', postId]`, plus `likedByMe` renvoyé par la requête de feed via un `LEFT JOIN likes ON user_id = auth.uid()`.
- **Fonction principale** : `toggleLike(postId)`.
- **Secondaires** : `optimisticApply(cacheKey, delta)`, `rollback(snapshot)`, `enqueueOfflineAction`, `emitEvent('like')`, `debounceFlush(600ms)`.
- **Clic** : 1) mise à jour optimiste immédiate (`isLiked=true`, count+1, animation) ; 2) écriture en attente dans une file locale ; 3) après debounce de 600 ms, envoi de l'**état final** (et non de chaque bascule) à `setLike(postId, liked)` — idempotent (`INSERT ... ON CONFLICT DO NOTHING` / `DELETE`).
- **Re-clic** : annule localement ; si l'état final égale l'état serveur connu, **aucune** requête n'est envoyée. C'est ce qui empêche un utilisateur nerveux de générer 30 requêtes.
- **Compteur** : trigger `AFTER INSERT/DELETE ON likes` → `UPDATE posts SET like_count = like_count ± 1`. Sur les posts très chauds (> 100 likes/s), on bascule sur un compteur Redis agrandi, aplati en base toutes les 5 s.
- **Stockage temporaire** : cache Query + file d'actions en IndexedDB si hors ligne.
- **Stockage permanent** : `likes` + `events(type='like', weight=1)`.
- **Synchronisation** : à la reconnexion, rejeu de la file par ordre chronologique, opérations idempotentes ; en cas de conflit, le serveur gagne.
- **Erreurs** : 409 → on resynchronise depuis le serveur ; 429 → backoff et message discret ; 403 (post bloqué/supprimé) → retrait du post du feed.
- **Sécurité** : RLS `WITH CHECK (user_id = auth.uid())` ; rate limit 100 likes/min ; détection de bot (likes à intervalle constant < 200 ms → poids d'événement mis à 0 dans l'algorithme, sans le dire à l'utilisateur).
- **Notifications** : `like.created` → notification agrégée au format « Alice et 12 autres ont aimé votre publication » via `group_key = 'like:{post_id}'` et `count++`. Pas de notification si auto-like ou si l'auteur a désactivé ce type.
- **Effet feed** : `events(weight=1)` alimente `post_stats.likes`, augmente `user_affinity(user, auteur)` de +0.02, et compte dans l'`engagement_rate` de la distribution mondiale.
- **Exemple** : `Alice → double-tap → toggleLike('p42') → cache: isLiked=true, count 128→129 → debounce 600 ms → setLike('p42', true) → INSERT likes → trigger like_count=129 → event(like, weight 1) → notification agrégée à Bob → post_stats.likes++ → engagement_rate recalculé → affinité Alice→Bob +0.02`.

### 4.8 Comments / Replies

- **Données** : `comments` avec `parent_id` (2 niveaux seulement : commentaire + réponses — un arbre profond à la Reddit complique la pagination et l'UI mobile ; si on le veut, utiliser `ltree` ou `materialized path`).
- **Principale** : `createComment({postId, body, parentId?})` ; **secondaires** : `listComments(postId, cursor, sort)`, `listReplies(commentId)`, `likeComment`, `deleteComment`, `pinComment` (auteur du post).
- **Tri** : par défaut « pertinence » = `like_count * 1.5 + reply_count * 2 + récence`, l'auteur du post remonté en premier.
- **Événements** : `comment.created` → notification à l'auteur du post + aux mentionnés + à l'auteur du commentaire parent (dédupliqué : une seule notification si c'est la même personne).
- **Feed** : poids 3 (un commentaire coûte plus qu'un like, donc il vaut plus).
- **Sécurité** : commentaires désactivables par post ; filtre de mots-clés personnalisé ; anti-spam (même texte posté > 3 fois → shadow-hide).

### 4.9 Save (sauvegarde)

- Privé par nature : **aucune notification** à l'auteur. `saves(user_id, post_id, collection_id)`, collections nommées.
- Poids algorithmique **élevé (4)** : sauvegarder est le signal le plus fort d'intérêt durable, et il est invisible donc peu manipulable socialement.
- Même mécanique optimiste que le like.

### 4.10 Share

- **Interne** : partage en message privé → crée un `message` avec `entity_type='post'`.
- **Externe** : Web Share API (`navigator.share`) avec repli sur copie de lien ; lien signé `/p/{id}?ref={hash}` pour attribuer la provenance sans exposer l'identité du partageur.
- **Poids 5** : le partage est le signal le plus corrélé à la distribution virale.
- **Événement** : `share.created(channel)`. Notification uniquement pour le partage interne (« Alice a envoyé votre publication à quelqu'un » — sans dire à qui).

### 4.11 Stories / Story views / Story replies

- **Données** : `stories(expires_at = created_at + 24h)`, `story_views`.
- **Purge** : `pg_cron` toutes les heures ; les médias expirés passent en archive privée de l'auteur.
- **Lecture** : préchargement des 2 stories suivantes, barre de progression, pause au maintien du doigt — état dans un store Zustand dédié (`storyViewerStore`).
- **Views** : `INSERT ... ON CONFLICT DO NOTHING` (une vue par personne), liste des vues visible uniquement de l'auteur (RLS).
- **Replies** : crée ou réutilise une conversation DM, message avec référence à la story ; si la story expire, le message conserve une vignette figée.
- **Ordre du carrousel** : `user_affinity DESC, unviewed first, recency` — exactement le même signal d'affinité que le feed.

### 4.12 Messages / Conversations / Requests / Archive

- **Données** : `conversations`, `conversation_members`, `messages`.
- **Envoi** : ID de message généré côté client (UUIDv7, ordonnable) → message affiché immédiatement en `sending` → insertion → Realtime → `sent` → `delivered` → `read`.
- **Secondaires** : `markAsRead(lastMessageId)` (un seul curseur par membre, pas un flag par message), `typingIndicator` (canal éphémère, jamais persisté), `archiveConversation`, `muteConversation`, `deleteForMe` vs `deleteForEveryone`.
- **Message requests** : si l'expéditeur n'est pas suivi par le destinataire → `is_request=true`, conversation invisible dans la boîte principale, aucune notification push, aucun accusé de lecture tant que la demande n'est pas acceptée.
- **Réalité technique du temps réel** : un canal par conversation ouverte + un canal personnel `user:{id}` pour les badges. Ne jamais ouvrir 200 canaux.
- **Chiffrement** : au repos par défaut ; E2EE (Signal Protocol / libsignal) possible en phase tardive — mais il supprime la modération serveur et la recherche dans les messages : décision produit, pas technique.
- **Ordre** : `last_message_at DESC`, mis à jour par trigger.

### 4.13 Notifications

- **Données** : `notifications(recipient_id, actor_id, type, entity, group_key, count, is_read)`.
- **Génération** : triggers Postgres (`AFTER INSERT` sur likes/comments/follows) → insertion ou **agrégation** sur `group_key` avec fenêtre de 1 h (`UPDATE ... SET count = count+1, created_at = now()` si une notification non lue du même groupe existe).
- **Livraison** : Realtime (in-app) + push FCM/APNs (mobile) + email digest (job quotidien pour les inactifs).
- **Préférences** : table `notification_settings` par type et par canal ; heures de silence.
- **Lecture** : `markAllAsRead()` et `markAsRead(id)`. Le badge lit un compteur agrégé, pas une somme de lignes.
- **Anti-bruit** : pas de notification pour ses propres actions, pas plus de N pushes par heure et par utilisateur, priorité aux types à forte valeur (DM > follow > comment > like).

### 4.14 Search

- **Phase 1 (Postgres)** : `profiles.search_vector` (GIN) pour le plein texte, `pg_trgm` pour la tolérance aux fautes sur `username`, `posts.caption_tsv` pour le contenu, table `hashtags(tag, post_count)`.
- **Requête typique** : union de 3 sous-recherches (users, hashtags, posts) avec scores normalisés, + boost si la personne est suivie ou a une forte affinité.
- **Suggestions instantanées** : debounce 250 ms, requête annulable (`AbortController`), cache Query par préfixe.
- **Historique** : local (IndexedDB), effaçable.
- **Phase 3** : Meilisearch (typo-tolérance excellente, très simple) ou OpenSearch (agrégations, analytics, gros volumes). **Quand basculer** : quand la recherche dépasse 200 ms p95 ou qu'on veut du classement sémantique.
- **Sécurité** : jamais de résultats provenant de comptes bloquants/bloqués ; les posts de comptes privés n'apparaissent qu'aux followers acceptés.

### 4.15 Blocking / Private profiles / Privacy

- **Block** : opération lourde et transactionnelle — supprime les follows dans les deux sens, masque les commentaires réciproques, exclut des recherches, des feeds, des stories, empêche les DM. Vérifié par une fonction `is_blocked(a, b)` appelée dans **toutes** les policies pertinentes, car un oubli sur une seule route rend le blocage inefficace.
- **Private profile** : RLS `visibility = 'public' OR author_id = auth.uid() OR EXISTS(follow accepté)`.
- **Privacy settings** : qui peut commenter, mentionner, répondre aux stories, m'envoyer des messages, voir ma liste d'abonnés, voir mon statut en ligne. Chacun est une colonne consultée par une policy ou une fonction serveur, jamais un filtre client.

### 4.16 Security / Login alerts / 2FA

- **2FA TOTP** : génération d'un secret chiffré, QR code, vérification à 6 chiffres, **10 codes de récupération** hashés (sans eux, un utilisateur qui perd son téléphone perd son compte). Fenêtre de tolérance ±1 intervalle, anti-rejeu du même code.
- **Login alerts** : après `auth.login.success`, comparaison avec `login_events` (device fingerprint + pays) → si nouveau : notification + email avec bouton « ce n'était pas moi » qui révoque toutes les sessions.
- **Sessions** : liste des appareils actifs, révocation individuelle.
- **Transversal** : RLS partout, validation Zod aux deux bouts, rate limiting par IP et par utilisateur, CSP stricte, en-têtes de sécurité, secrets jamais dans le bundle client, audit log des actions sensibles, URLs de média signées et expirantes pour le contenu privé.

### 4.17 Content moderation / Reporting

- **Pipeline à l'upload** : hash perceptuel (pHash) contre une base de contenus interdits connus → classifieur d'image (NSFW/violence) → analyse texte (toxicité, spam) → décision : `allow` / `limit` (visible mais exclu des recommandations) / `block` (refusé) / `review` (file humaine).
- **Reporting** : `reports` + agrégation ; N signalements distincts en M minutes → réduction automatique de distribution en attendant la revue humaine (le « limit » est préférable au « block » automatique : moins de faux positifs destructeurs).
- **Sanctions graduées** : avertissement → limitation de portée → suspension temporaire → bannissement, chaque étape journalisée et contestable.

### 4.18 Realtime events

- Canaux : `user:{id}` (notifications, badges), `conversation:{id}` (messages, typing), `post:{id}` (compteurs live, uniquement si le post est ouvert).
- Reconnexion avec backoff exponentiel ; à la reconnexion, **refetch** des données manquées plutôt que rejeu d'événements (plus simple et plus fiable).
- Limite : un canal `post:{id}` par post ouvert, fermé au démontage — sinon fuite de connexions dans un scroll infini.

### 4.19 Image / media handling

Déjà décrit en 1.8. À retenir : compression client avant upload (divise par 5 le coût réseau), `blurhash` pour éviter le saut de mise en page, `loading="lazy"` + `srcset`, `content-visibility: auto` pour le feed long, et suppression physique différée (les CDN gardent des copies).

---

## 5. FEED ET ALGORITHME

### 5.1 Pipeline en 5 étages

```text
1. CANDIDATE GENERATION   ~2000 posts
2. FILTERING              ~800
3. LIGHT RANKING          ~200
4. HEAVY RANKING / PERSO  ~50
5. RE-RANKING (diversité, anti-répétition, second chance)  → 20 servis
```

### 5.2 Candidate generation (sources, avec quotas)

| Source | Part | Description |
|---|---|---|
| Following | 40 % | posts récents (< 72 h) des comptes suivis |
| Affinity graph | 15 % | posts aimés par les comptes à forte affinité (2ᵉ degré) |
| Topic match | 20 % | posts sur les sujets où `user_topic_affinity` est élevée |
| Trending local | 15 % | posts en forte vélocité dans le pays/la langue de l'utilisateur |
| Exploration | 10 % | aléatoire contrôlé, indispensable pour découvrir de nouveaux goûts et éviter la bulle |

### 5.3 Filtering (élimination dure)

Bloqués/bloquants · déjà vus (Redis `seen:{user}`, TTL 7 j) · auteur mis en sourdine · contenu `limit`/`block` par la modération · posts supprimés · visibilité privée non autorisée · pays hors périmètre de distribution du post (section 6) · plus de 2 posts du même auteur dans la session.

### 5.4 Signaux et formules

**Score d'engagement brut, normalisé par les impressions :**

\[
E = \frac{1\cdot L + 3\cdot C + 4\cdot S + 5\cdot Sh + 2\cdot W}{I}
\]

où L = likes, C = commentaires, S = saves, Sh = shares, W = vues « complètes » (dwell > 3 s ou > 80 % de la vidéo), I = impressions. Normaliser par les impressions est essentiel : sinon les gros comptes gagnent toujours et aucun nouveau créateur n'émerge.

**Vélocité d'engagement** (accélération, pas volume) :

\[
V = \frac{E_{[0,t]}}{\max(t,\ 0.5)}\quad\text{(t en heures)},\qquad V_{norm} = \min\!\left(\frac{V}{V_{p90}},\ 1\right)
\]

**Fraîcheur** (décroissance exponentielle, demi-vie 12 h) :

\[
F = e^{-\lambda \Delta t},\qquad \lambda = \frac{\ln 2}{12}
\]

**Relation au créateur :**

\[
R = 0.5\cdot A_{user\to author} + 0.3\cdot \mathbb{1}[\text{suivi}] + 0.2\cdot \frac{\text{interactions}_{30j}}{\text{interactions}_{max}}
\]

où \(A\) est l'affinité (mise à jour incrémentale : \(A \leftarrow 0.95A + \delta\), avec δ = 0.02 like, 0.05 commentaire, 0.08 save, 0.10 DM/partage, −0.15 « pas intéressé »).

**Pertinence sujet** = similarité cosinus entre l'embedding du post et le vecteur d'intérêt de l'utilisateur (pgvector) : \(P = \cos(\vec{u}, \vec{p})\).

**Signaux négatifs :**

\[
N = 0.4\cdot\text{hide} + 0.3\cdot\text{report} + 0.2\cdot\text{unfollow} + 0.1\cdot\text{scroll rapide} (\text{dwell} < 0.8\text{s})
\]

**Qualité du contenu** \(Q\) ∈ [0,1] : résolution, absence de watermark d'une autre plateforme, longueur de légende, score de modération.

**Score final :**

\[
\boxed{\;Score = \big(0.30\,E_{norm} + 0.20\,V_{norm} + 0.15\,F + 0.20\,R + 0.10\,P + 0.05\,Q\big)\times(1 - N)\times D\;}
\]

où \(D\) est le multiplicateur de diversité appliqué au re-ranking (0.6 si c'est le 2ᵉ post du même auteur, 0.8 si c'est le 3ᵉ post du même sujet d'affilée, 1.0 sinon).

### 5.5 Second chance

Un post dont \(I < 200\) et \(E > \text{médiane}\) reçoit un bonus `+0.15` et est réinjecté dans la génération de candidats pendant 24 h. Sans ce mécanisme, un post publié à 3 h du matin est condamné, et la plateforme devient un système où seule l'heure de publication compte.

### 5.6 Anti-répétition et fatigue

`seen_posts` (Redis, TTL 7 j) ; un post vu sans interaction 2 fois n'est plus jamais re-servi ; un auteur ne peut occuper plus de 2 slots sur 20 ; pas plus de 40 % du feed sur un même sujet.

### 5.7 EXEMPLE NUMÉRIQUE

Utilisateur : **Alice**. 5 candidats.

| Post | Auteur | Âge | I | L | C | S | Sh | W | Suivi | A |
|---|---|---|---:|---:|---:|---:|---:|---:|---|---:|
| P1 | Bob | 2 h | 1000 | 120 | 20 | 15 | 8 | 400 | oui | 0.80 |
| P2 | Chloé | 10 h | 5000 | 400 | 30 | 10 | 5 | 1500 | non | 0.10 |
| P3 | David | 0.5 h | 80 | 14 | 4 | 3 | 2 | 40 | non | 0.00 |
| P4 | Emma | 30 h | 2000 | 300 | 50 | 40 | 25 | 900 | oui | 0.60 |
| P5 | Farid | 1 h | 300 | 9 | 1 | 0 | 0 | 60 | non | 0.05 |

**Engagement E = (L + 3C + 4S + 5Sh + 2W)/I**

- P1 = (120 + 60 + 60 + 40 + 800)/1000 = **1.080**
- P2 = (400 + 90 + 40 + 25 + 3000)/5000 = **0.711**
- P3 = (14 + 12 + 12 + 10 + 80)/80 = **1.600**
- P4 = (300 + 150 + 160 + 125 + 1800)/2000 = **1.267**
- P5 = (9 + 3 + 0 + 0 + 120)/300 = **0.440**

Normalisation par le max (1.600) → `E_norm` : P1 0.675 · P2 0.444 · P3 1.000 · P4 0.792 · P5 0.275

**Vélocité V = E/max(t,0.5)**, puis normalisation par le max (3.200) :

- P1 = 1.080/2 = 0.540 → 0.169
- P2 = 0.711/10 = 0.071 → 0.022
- P3 = 1.600/0.5 = 3.200 → **1.000**
- P4 = 1.267/30 = 0.042 → 0.013
- P5 = 0.440/1 = 0.440 → 0.138

**Fraîcheur F = e^(−0.0578·Δt)** : P1 0.891 · P2 0.561 · P3 0.971 · P4 0.177 · P5 0.944

**Relation R = 0.5A + 0.3·suivi + 0.2·interactions** (interactions récentes : Bob 0.9, Emma 0.7, autres 0) :

- P1 = 0.40 + 0.30 + 0.18 = **0.880**
- P2 = 0.05 + 0 + 0 = **0.050**
- P3 = 0 → **0.000**
- P4 = 0.30 + 0.30 + 0.14 = **0.740**
- P5 = 0.025 → **0.025**

**Pertinence P** (cosinus) : P1 0.70 · P2 0.85 · P3 0.60 · P4 0.75 · P5 0.20
**Qualité Q** : P1 0.9 · P2 0.9 · P3 0.8 · P4 0.9 · P5 0.4
**Négatif N** : tous 0 sauf P5 = 0.30 (beaucoup de scrolls rapides)

**Score = 0.30E + 0.20V + 0.15F + 0.20R + 0.10P + 0.05Q, ×(1−N)**

- **P1** = 0.2025 + 0.0338 + 0.1337 + 0.1760 + 0.070 + 0.045 = **0.661**
- **P2** = 0.1332 + 0.0044 + 0.0842 + 0.0100 + 0.085 + 0.045 = **0.362**
- **P3** = 0.3000 + 0.2000 + 0.1457 + 0.0000 + 0.060 + 0.040 = **0.746** → + second chance (I=80 < 200 et E élevé) **+0.15 = 0.896**
- **P4** = 0.2376 + 0.0026 + 0.0266 + 0.1480 + 0.075 + 0.045 = **0.534**
- **P5** = (0.0825 + 0.0276 + 0.1416 + 0.0050 + 0.020 + 0.020) = 0.2967 × 0.70 = **0.208**

**Classement final : P3 (0.896) > P1 (0.661) > P4 (0.534) > P2 (0.362) > P5 (0.208).**

Lecture : P2 a le plus gros volume absolu (400 likes) mais finit 4ᵉ — la normalisation par impressions et l'absence de relation la pénalisent. P3, un petit post tout neuf d'un inconnu avec un excellent taux, gagne. C'est exactement le comportement recherché : Yuniko récompense la **qualité relative**, pas la notoriété acquise.

---

## 6. DISTRIBUTION MONDIALE PROGRESSIVE

### 6.1 Machine à états

```text
stage 1 : 3 pays    → évaluation → stage 2 : 5 pays   (les 3 initiaux conservés + 2)
stage 2 : 5 pays    → évaluation → stage 3 : 7 pays   (les 5 conservés + 2)
stage 3 : 7 pays    → évaluation → stage 4 : MONDIAL
échec → stage 'held' → une seconde chance à +12 h → sinon 'stopped'
```

Les pays précédents sont **toujours conservés** : l'expansion est additive, jamais un remplacement. Retirer un pays casserait l'audience déjà acquise.

### 6.2 Sélection initiale des 3 pays (le créateur ne choisit pas)

Score par pays candidat :

\[
S_{pays} = 0.40\cdot\text{followers}_{pays} + 0.25\cdot\text{langue} + 0.20\cdot\text{affinité historique du créateur} + 0.15\cdot\text{réceptivité du pays au sujet}
\]

- `pays du créateur` est toujours inclus d'office ;
- les 2 autres sont les mieux notés, avec une règle de diversité : pas deux pays du même fuseau à moins de 2 h d'écart si un autre marché viable existe (sinon l'évaluation se fait sur une seule tranche horaire).

### 6.3 Seuils d'évaluation

| Stage | Impressions minimales | ER minimal | Vélocité minimale | Fenêtre max |
|---|---:|---:|---:|---:|
| 1 → 2 | 300 | 6 % | 0.04 ER/h | 6 h |
| 2 → 3 | 1 500 | 5 % | 0.03 ER/h | 12 h |
| 3 → 4 (mondial) | 8 000 | 4 % | 0.02 ER/h | 24 h |

\[
ER = \frac{L + C + S + Sh}{I},\qquad V = \frac{ER}{\text{heures depuis l'entrée dans le stage}}
\]

Les seuils **baissent** quand on monte : plus l'audience s'élargit, plus elle est froide, donc exiger 6 % au niveau mondial reviendrait à ne jamais distribuer personne. Ces valeurs doivent être des **percentiles glissants** calculés par (catégorie de contenu × pays), pas des constantes codées : un seuil fixe favorise mécaniquement les formats à engagement facile.

### 6.4 Règles complémentaires

- **Calcul par rapport aux impressions uniquement** — jamais en valeur absolue, sinon un gros compte franchit tous les paliers sans mérite.
- **Impressions minimales obligatoires** avant toute décision : sous 300 impressions, l'ER n'est pas statistiquement exploitable (on utilise un intervalle de Wilson à 95 % plutôt que le taux brut pour comparer les petits échantillons).
- **Seconde chance** : un post recalé reste en `held` 12 h ; s'il gagne de l'engagement tardif (partage externe, reprise), il repart au même stage une seule fois.
- **Diversité** : au plus 20 % des slots « nouveaux contenus » d'un pays donné proviennent du même créateur ou du même pays source, pour que le palier mondial ne soit pas monopolisé par un seul marché.
- **Anti-spam** : plafond de posts par créateur et par heure entrant en distribution ; détection de fermes d'engagement (comptes récents, ratio abonnements/abonnés anormal, likes en rafale) → leurs événements ont un **poids nul** dans le calcul d'ER, sans que le fraudeur puisse le détecter.
- **Stockage de l'état** : table `post_distribution(post_id, stage, countries[], entered_stage_at, impressions_at_stage, engagement_rate, velocity, status, second_chance_used)`, évaluée par un job `pg_cron` toutes les 5 minutes sur les seuls posts `status='active'` dont la fenêtre n'est pas expirée.
- **Lien avec le feed** : l'étage *filtering* écarte tout post dont le pays de l'utilisateur n'est pas dans `post_distribution.countries` (sauf s'il suit l'auteur — les followers voient toujours tout, dès le stage 1).

### 6.5 Exemple

Post de Bob (Madagascar), stage 1 = [MG, FR, RE].
À t+4 h : I = 520, L = 41, C = 6, S = 4, Sh = 3 → ER = 54/520 = **10.4 %** ≥ 6 %, V = 0.104/4 = 0.026… — vélocité insuffisante ? Non : à t+2 h il était déjà à I=310, ER=9 %, V=0.045 ≥ 0.04 → **promu au stage 2** dès 2 h : [MG, FR, RE, BE, CA]. À t+9 h : I = 2 100, ER = 5.8 % ≥ 5 %, V = 0.0083… < 0.03 → l'ER tient mais la vélocité retombe → statut `held`. À t+21 h, un partage externe relance : I = 3 400, ER = 6.1 %, V repart → **seconde chance consommée**, passage stage 3 : + [US, DE]. À t+36 h : I = 9 800, ER = 4.3 % ≥ 4 % → **distribution mondiale**.

---

## 7. ORGANISATION DU CODE

```text
src/
├── routes/                 # une route = un fichier ; SSR, head() SEO, loaders
│   ├── index.tsx           # feed personnalisé
│   ├── explore.tsx         # discovery
│   ├── u.$username.tsx     # profil public (SSR + JSON-LD)
│   ├── p.$postId.tsx       # post en détail (SSR, og:image)
│   ├── messages/           # liste + conversation
│   ├── settings/           # profil, confidentialité, sécurité, notifications
│   └── api/public/         # webhooks, endpoints cron, health
│
├── features/               # LE cœur : une fonctionnalité = un dossier autonome
│   ├── auth/               # components/ hooks/ services/ schemas.ts types.ts
│   ├── profile/
│   ├── posts/
│   ├── likes/
│   ├── comments/
│   ├── saves/
│   ├── stories/
│   ├── messaging/
│   ├── notifications/
│   ├── search/
│   ├── feed/
│   ├── follow/
│   ├── moderation/
│   └── settings/
│
├── components/
│   ├── ui/                 # shadcn : button, dialog, sheet, avatar…
│   └── shared/             # PostCard, StoryRing, UserAvatar, EmptyState, InfiniteList
│
├── hooks/                  # transversaux : useHydrated, useIntersection, useDebounce,
│                           # useInfiniteScroll, useOnlineStatus, useMediaQuery
│
├── services/               # accès données : client base, upload, push, analytics
│                           # aucun JSX ici, testable isolément
│
├── stores/                 # Zustand : sessionStore, composerStore, storyViewerStore,
│                           # uiStore, offlineQueueStore
│
├── algorithms/             # PUR TypeScript, sans I/O, 100 % testable
│   ├── feed/ candidates.ts scoring.ts diversity.ts freshness.ts
│   ├── distribution/ stages.ts thresholds.ts evaluate.ts
│   ├── affinity.ts
│   └── ranking-comments.ts
│
├── lib/                    # server functions (*.functions.ts), utilitaires serveur
├── types/                  # types générés depuis la base + types de domaine
├── utils/                  # formatage (dates relatives, compteurs 1.2k), validation
└── styles.css              # design system unique (tokens oklch)
```

**Règles d'organisation** :
- Un composant de `components/shared/` ne connaît aucune feature ; une feature peut importer `shared`, l'inverse est interdit.
- `algorithms/` ne fait **aucun** appel réseau : entrée = données, sortie = score. C'est la seule façon de tester un ranking.
- Les schémas Zod vivent dans la feature et sont importés par le client **et** la fonction serveur : une seule définition de validation.

---

## 8. RÉFÉRENCES CONCEPTUELLES (techniques publiques, reproductibles légalement)

| Source | Technique générale reproductible | Application dans Yuniko |
|---|---|---|
| Facebook | EdgeRank historique (affinité × poids × décroissance temporelle) ; agrégation de notifications | formule de score, `group_key` |
| Instagram | signaux d'intérêt implicites (dwell time, saves) ; stories éphémères 24 h ; carrousel trié par affinité | poids `save=4`, `stories.expires_at` |
| TikTok | distribution progressive par paliers testée sur petits lots ; taux de complétion comme signal roi ; exploration forcée | section 6, signal W, 10 % exploration |
| X | vélocité d'engagement et détection de tendances par dérivée ; ratio réponses/likes comme signal de controverse | `V`, trending local |
| Reddit | classement par score de Wilson pour petits échantillons ; tri par « hot » combinant votes et âge logarithmique | intervalle de Wilson au stage 1 |

Ce qui est reproductible : les **concepts publiés** (articles d'ingénierie, brevets expirés, littérature académique sur les systèmes de recommandation). Ce qui ne l'est pas : code source, modèles entraînés, données, marques, et l'imitation servile de l'interface (risque de « trade dress »). Yuniko doit avoir son propre langage visuel et sa propre pondération de signaux.

---

## 9. PHASES DE DÉVELOPPEMENT ET DÉPENDANCES

```text
Authentication → Profile → Follow → Posts → Interactions → Notifications
     → Feed → Recommendation → Distribution → Realtime → Messaging → Moderation
```

**PHASE 1 — Fondations (socle non négociable)**
Base de données + RLS · Auth (register/login/logout/reset) · Profils + édition + avatar · Design system complet.
*Dépendance : tout le reste en dépend. Une erreur de schéma ici coûte 10× plus tard.*

**PHASE 2 — Graphe social et contenu**
Follow/unfollow · demandes pour comptes privés · listes followers/following · création/suppression de posts · upload et pipeline média · profil avec grille de posts · feed chronologique simple (following, tri par date).
*Le feed chronologique est volontaire : il valide la lecture, la pagination et le cache avant toute complexité algorithmique.*

**PHASE 3 — Interactions**
Like/unlike (avec optimisme et debounce) · commentaires et réponses · saves et collections · partages · compteurs par triggers · table `events` dès maintenant, même si rien ne la consomme encore.
*Dépend de Posts. La table `events` doit exister tôt : on ne peut pas classer sur des données qu'on n'a pas collectées.*

**PHASE 4 — Notifications et recherche**
Notifications avec agrégation · préférences · badge temps réel · recherche users/hashtags/posts en Postgres.
*Dépend des Interactions (elles produisent les notifications).*

**PHASE 5 — Feed algorithmique**
`post_stats` (job d'agrégation) · `user_affinity` · candidate generation multi-sources · scoring · diversité · anti-répétition · A/B testing des poids.
*Dépend de 3 et 4, et surtout d'un volume d'événements réel : inutile avant quelques milliers d'utilisateurs actifs.*

**PHASE 6 — Distribution mondiale**
`post_distribution` · job `pg_cron` d'évaluation · seuils en percentiles · seconde chance · anti-fraude · intégration au filtering du feed.
*Dépend de la Phase 5 (elle réutilise les mêmes signaux).*

**PHASE 7 — Temps réel et messagerie**
Canaux Realtime · DM 1-1 · conversations de groupe · message requests · archive · accusés de lecture · typing · présence.
*Indépendant du feed, peut être parallélisé avec 5-6 par une autre équipe.*

**PHASE 8 — Stories**
Création, lecteur, vues, réponses, expiration automatique, archive.
*Dépend de Messaging (les réponses aux stories sont des DM) et du pipeline média.*

**PHASE 9 — Sécurité avancée et modération**
2FA TOTP + codes de récupération · alertes de connexion · gestion des sessions · blocage complet · signalements · classifieurs · file de revue humaine · sanctions graduées.
*À ne pas repousser : dès l'ouverture publique, la modération devient une obligation légale (DSA en Europe) autant qu'un enjeu de survie produit.*

**PHASE 10 — Échelle et mobile**
Cache Redis des timelines · fan-out asynchrone · Meilisearch/OpenSearch · pgvector et embeddings · application React Native partageant `algorithms/` et `features/*/services` · push natif.

---

## 10. SYNTHÈSE DES FLUX (canevas type)

```text
Utilisateur → Action UI → hook de feature (useX) → mutation optimiste (cache Query)
     → service → server function (validation Zod + auth) → transaction Postgres
     → triggers (compteurs, notification) → INSERT events
     → Realtime push (destinataire) + invalidation de cache (émetteur)
     → jobs pg_cron (post_stats, distribution, affinité)
     → prochain calcul de feed
```

Toute fonctionnalité de Yuniko doit pouvoir se décrire dans ce canevas. Si une action n'y rentre pas — par exemple parce qu'elle écrit directement depuis le client sans validation serveur, ou parce qu'elle ne produit pas d'événement — c'est le signe d'une erreur d'architecture, pas d'une exception légitime.
