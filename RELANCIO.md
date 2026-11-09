# Relancio — Cahier des charges pour Claude Code

> Ce fichier est référencé par `CLAUDE.md`, que Claude Code relit à chaque session.
> Premier message à envoyer : « Lis le cahier des charges. Commence par la Phase 0 uniquement, puis arrête-toi et dis-moi comment tester. »

## 1. Le produit

**Relancio** est un SaaS qui relance automatiquement les clients qui n’ont pas payé leurs factures.
Cibles : indépendants, agences et PME qui n’ont pas de service comptabilité.
Promesse : « Vos factures impayées se règlent toutes seules. »
Langue de l’interface : **français**. Devise par défaut : EUR (champ modifiable : EUR, XOF).

## 2. Règles de travail (à respecter absolument)

1. Travaille **une phase à la fois**. À la fin de chaque phase : résume ce qui a été fait, explique comment le tester, puis **attends ma validation**.
2. Je ne suis pas développeur. Explique simplement, et dis-moi précisément quoi cliquer ou copier.
3. Ne jamais mettre de clés secrètes dans le code. Tout passe par `.env.local` (et fournis un `.env.example`).
4. Avant toute action risquée (supprimer des données, changer le schéma, installer un gros paquet), demande-moi.
5. Écris des tests pour la logique de relance (Phase 4). Lance-les avant de dire que c’est terminé.
6. Code en anglais, textes visibles par l’utilisateur en français.

## 3. Stack technique

- **Next.js** (App Router) + **TypeScript** + **Tailwind CSS**
- **Supabase** : authentification (e-mail + mot de passe), base Postgres, **Row Level Security activée sur toutes les tables**
- **Resend** : envoi des e-mails de relance
- **Vercel** : hébergement + Vercel Cron pour les tâches planifiées
- **Stripe** : abonnements (Phase 6 seulement)

## 4. Design

Reprendre l’identité de la landing page existante (fichier `landing/index.html`, s’il est présent dans le projet) :
- Police : Plus Jakarta Sans
- Bleu principal `#2563EB`, accent orange `#EA580C` (texte sombre dessus), fond `#F8FAFC`, texte `#1E293B`
- Cartes blanches, bordures fines, coins arrondis, mode sombre automatique
- Accessibilité : contrastes 4,5:1 minimum, focus visibles, boutons d’au moins 44 px, respect de `prefers-reduced-motion`
- Mobile d’abord

## 5. Modèle de données (Supabase)

Toutes les tables ont `user_id uuid references auth.users` et une politique RLS : **chaque utilisateur ne voit et ne modifie que ses propres lignes**.

- `profiles` : id, user_id, company_name, sender_name, reply_to_email, currency, created_at
- `clients` : id, user_id, name, email, phone (optionnel), created_at
- `invoices` : id, user_id, client_id, number, amount (numeric), currency, issue_date, due_date, status (`pending` | `paid` | `cancelled`), payment_link (url optionnelle), paid_at, created_at
- `reminder_templates` : id, user_id, name, tone (`friendly` | `neutral` | `firm`), subject, body, days_offset (entier : −3 = 3 jours avant l’échéance, 0 = le jour J, 7 = 7 jours après), active
- `reminders` : id, user_id, invoice_id, template_id, scheduled_for, sent_at, status (`scheduled` | `sent` | `failed` | `skipped`), error_message

Variables disponibles dans les modèles : `{{client_name}}`, `{{invoice_number}}`, `{{amount}}`, `{{due_date}}`, `{{payment_link}}`, `{{sender_name}}`, `{{company_name}}`.

## 6. Phases

### Phase 0 — Socle
Créer le projet Next.js, configurer Tailwind, connecter Supabase, créer le fichier `.env.example`, ajouter les tables ci-dessus avec leurs politiques RLS (fichier de migration SQL).
**Validation :** l’appli démarre en local et la base est créée.

### Phase 1 — Comptes
Pages : inscription, connexion, déconnexion, mot de passe oublié. Redirection vers `/app` si connecté, vers `/connexion` sinon. Création automatique du `profile` à l’inscription. Page « Paramètres » (nom de l’entreprise, nom d’expéditeur, e-mail de réponse, devise).
**Validation :** je peux créer un compte, me déconnecter, me reconnecter, modifier mes paramètres. Un deuxième compte ne voit pas les données du premier.

### Phase 2 — Clients et factures
- Liste, création, modification, suppression de clients.
- Liste des factures avec filtres (toutes / en attente / en retard / payées) et recherche.
- Création et modification d’une facture (client, numéro, montant, dates, lien de paiement optionnel).
- Bouton « Marquer comme payée » (met `status = paid`, `paid_at = now`, annule les relances à venir).
- **Import CSV** (colonnes : client, email, numero, montant, date_echeance) avec aperçu avant import et message clair pour chaque ligne en erreur.
- Une facture est « en retard » si `due_date < aujourd’hui` et `status = pending`.
**Validation :** je peux ajouter 3 factures à la main et en importer 10 par CSV.

### Phase 3 — Tableau de bord
Page `/app` avec : total en attente, total en retard, total encaissé ce mois, nombre de factures en retard, liste des 5 prochaines relances, liste des factures les plus en retard. États vides soignés (message + bouton pour ajouter la première facture).
**Validation :** les chiffres correspondent à mes données.

### Phase 4 — Modèles et planification des relances
- Créer automatiquement 3 modèles par défaut à l’inscription : rappel avant échéance (−3 j, ton amical), relance à l’échéance (0 j, neutre), relance ferme (+7 j), dernière relance (+15 j).
- Page « Relances » pour modifier les modèles (sujet, corps, ton, délai, actif/inactif) avec **aperçu avec de vraies données**.
- Quand une facture est créée ou sa date modifiée : générer ses lignes `reminders` selon les modèles actifs. Ne rien planifier dans le passé.
- Quand une facture passe en `paid` ou `cancelled` : passer ses relances futures en `skipped`.
- Écrire des tests pour : calcul des dates, annulation au paiement, pas de doublons.
**Validation :** je crée une facture et je vois ses relances planifiées avec les bonnes dates.

### Phase 5 — Envoi automatique
- Route `/api/cron/reminders` appelée chaque heure par Vercel Cron, **protégée par un secret** (`CRON_SECRET`).
- Elle envoie via Resend toutes les relances `scheduled` dont la date est passée, remplace les variables, met `sent_at` et `status`, et enregistre l’erreur en cas d’échec (pas de réessai infini : 3 tentatives maximum).
- Avant d’envoyer, vérifier que la facture est toujours `pending`.
- E-mail envoyé au nom de l’utilisateur (`sender_name`), avec `reply-to` = son e-mail. Ajouter un lien de désinscription simple pour le destinataire.
- Bouton « Envoyer un e-mail de test à moi-même » et bouton « Envoyer maintenant » sur une facture.
- Page historique : toutes les relances envoyées, avec statut.
**Validation :** je reçois bien un e-mail de test, et une relance planifiée à l’instant est envoyée par le cron.

### Phase 6 — Abonnements (Stripe)
Offres : Solo 19 €/mois (50 factures/mois), Équipe 49 €/mois (500 factures/mois), Business 129 €/mois (illimité). Essai gratuit de 14 jours sans carte. Stripe Checkout + Customer Portal, webhooks pour synchroniser l’état de l’abonnement, blocage doux (message + lien pour passer à l’offre supérieure) quand la limite est atteinte.
Utiliser les **clés de test Stripe** tant que je n’ai pas dit « passe en production ».

### Phase 7 — Mise en production
Déploiement sur Vercel, variables d’environnement, domaine personnalisé, configuration SPF/DKIM pour l’envoi d’e-mails (explique-moi pas à pas), pages Mentions légales / Confidentialité / CGU, suppression de compte et export des données de l’utilisateur (RGPD), vérification finale de la sécurité (RLS, secrets, routes protégées).

## 7. Hors périmètre de la version 1 (ne pas faire)

SMS, WhatsApp, API publique, prévision de trésorerie, multi-utilisateurs par compte, connexion directe à des logiciels de facturation. À prévoir après les premiers clients payants.

## 8. Critères de qualité

- Aucune donnée d’un utilisateur visible par un autre (tester avec deux comptes).
- Toutes les pages utilisables sur téléphone.
- Messages d’erreur clairs, en français, jamais de texte technique brut.
- États de chargement et états vides sur chaque liste.
- Les actions destructives demandent une confirmation.

## 9. Variables d’environnement attendues

```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
RESEND_API_KEY=
RESEND_FROM_EMAIL=
CRON_SECRET=
NEXT_PUBLIC_APP_URL=
STRIPE_SECRET_KEY=            # Phase 6
STRIPE_WEBHOOK_SECRET=        # Phase 6
```
