# BaoPixel Studio Facturation — Brief de passation pour Claude Code

## Qui, quoi, pourquoi

**BaoPixel Digital Agency** — agence créative digitale solo, fondateur **Tanor Fall**, basée Mbour/Saly, Petite-Côte, Sénégal. Statut légal : **Entreprise Individuelle**, régime comptable **OHADA / Système Minimal de Trésorerie (SMT)**. NINEA : 0103191193.

L'app **BaoPixel Studio Facturation** est son outil de gestion quotidien : devis, factures, reçus, dépenses, trésorerie. Elle est utilisée en conditions réelles, pas en prototype — Tanor a déjà généré de vrais documents avec.

**Ce qui déclenche cette passation** : Tanor veut que ses données soient **synchronisées en temps réel entre plusieurs appareils** (téléphone + laptop, utilisés en même temps). L'app actuelle est un fichier HTML unique en localStorage — zéro backend, zéro dépendance, mais donc aussi zéro synchronisation entre appareils. Ce chantier (base de données hébergée, authentification, sync temps réel) demande un vrai environnement de dev avec déploiement et tests multi-appareils, ce qu'on ne peut pas faire proprement dans une conversation de chat. D'où le passage à Claude Code.

---

## Architecture actuelle (à connaître avant de toucher au code)

- **Un seul fichier HTML** (~2600 lignes) : `BaoPixel_Studio_Facturation.html`, joint à cette conversation.
- **Stockage** : `localStorage` du navigateur, aucune donnée envoyée nulle part.
- **Génération PDF** : jsPDF chargé depuis un CDN (cdnjs), rendu 100% côté client.
- **Pas de build step, pas de npm, pas de framework.** JS vanilla, CSS inline dans le `<head>`.
- **Thème clair/sombre** géré en CSS variables + `data-theme` sur `<html>`.
- **Police standard jsPDF (Helvetica) ne supporte PAS les emojis/symboles Unicode** — piège rencontré plusieurs fois pendant le développement (signe moins spécial, coches ✓, carrés ■ qui sortaient en caractères garbled dans les PDF). Tous les textes envoyés à `doc.text()` doivent rester en ASCII/Latin-1 basique. Une fonction `stripEmoji()` existe déjà pour nettoyer les libellés avant impression PDF — à réutiliser si de nouveaux textes sont envoyés au PDF.

### Les 5 modules construits (tous fonctionnels et testés)

1. **📝 Devis & Facture** — calculateur (type de projet, gamme, délai, options éditables), génère un PDF devis ou facture. Numérotation `DEV-YYYY-[SECTEUR]-[ID]` / `FAC-YYYY-[SECTEUR]-[ID]`.
2. **📂 Classeur** — archive tous les devis/factures, statuts (`envoye`/`valide`/`refuse`/`converti`/`emise`/`partiel`/`payee`), recherche, conversion devis→facture.
3. **💸 Dépenses** — catégorisées avec comptes OHADA pré-mappés, compte figé à l'enregistrement (jamais recalculé rétroactivement si la catégorie change).
4. **🧾 Reçus de paiement** — génère un reçu PDF avec cachet (cercle + texte pivoté façon tampon), gère les paiements partiels/acomptes, met à jour `montantEncaisse` sur la facture liée.
5. **📊 Journal de trésorerie** — fusionne encaissements (reçus) et décaissements (dépenses) chronologiquement avec solde qui roule, filtres (mois précis via `<input type="month">`, mode de paiement, recherche), export PDF "Rapport comptable" avec récapitulatif par compte OHADA.

Plus : **🏠 Tableau de bord** (page d'accueil, chiffres clés du mois, devis en attente avec badges d'ancienneté, factures à encaisser) et **💾 Export/Import** (sauvegarde JSON manuelle, déjà fonctionnelle — voir plus bas).

---

## Schémas de données (clés localStorage actuelles)

```js
// bp_docs_v1 — devis et factures
{
  id: string,              // = invNum
  invNum: string,          // "DEV-2026-AIRBNB-XXXXX" ou "FAC-..."
  docType: 'devis' | 'facture',
  status: 'envoye' | 'valide' | 'refuse' | 'converti' | 'emise' | 'partiel' | 'payee',
  client: { name, tel, proj },
  typeName, gName, dName,  // type de projet, gamme, délai (libellés affichés)
  qty: number,
  total: number,           // FCFA
  validDays: number,       // devis uniquement
  untilStr: string,        // date limite validité devis
  invDate: string,         // YYYY-MM-DD
  createdAt: ISOString,
  updatedAt: ISOString,
  linkedDevisNum: string|null,   // si facture née d'un devis
  linkedFactureNum: string|null, // si devis converti en facture
  montantEncaisse: number  // factures uniquement, cumul des reçus liés
}

// bp_expenses_v1 — dépenses
{
  id: string,               // "EXP-..."
  date: string,             // YYYY-MM-DD
  catId: string,
  montant: number,
  mode: 'especes'|'wave'|'om'|'banque',
  projet: string, note: string,
  createdAt: ISOString,
  compte: string,            // compte OHADA FIGÉ à la création
  catNameSnapshot: string,   // nom de catégorie FIGÉ à la création
  catIcSnapshot: string      // icône FIGÉE à la création
}

// bp_expense_cats_v1 — catégories de dépenses (éditables par l'utilisateur)
{ id, ic (icône), nm (nom), compte (n° OHADA) }[]

// bp_recus_v1 — reçus de paiement
{
  id: string, recuNum: string,      // "REC-2026-..."
  invNum: string,                    // référence à la facture
  client: {...},
  montant: number, mode, date, note,
  resteApres: number, soldee: boolean,
  createdAt: ISOString
}

// bp_custom_types_v1 — types de projet personnalisés créés par l'utilisateur
// bp_addons_v1 — options/addons éditables du calculateur
```

**Comptes OHADA déjà mappés** (constante `OHADA_COMPTES` + `DEFAULT_EXP_CATS` dans le code) :
Ventes de services → 7061, Clients → 4111, Caisse → 571, Banque → 52, Mobile Money (Wave/OM) → 554. Dépenses : Transport/Carburant 6181, Location matériel 6223, Repas/Mission 6384, Télécom 6281, Fournitures 6055, Hébergement 6384, Sous-traitance 6327.

---

## Le nouveau besoin : synchronisation multi-appareils en temps réel

Tanor veut ouvrir l'app sur son téléphone ET son laptop et voir **les mêmes données, en même temps**, sans étape manuelle. Ça veut dire quitter le localStorage pur et introduire un vrai backend.

### Pistes recommandées à évaluer avec lui

1. **Supabase** (Postgres + Realtime + Auth) — probablement le choix le plus rapide à mettre en place. SDK JS utilisable directement depuis le frontend, tier gratuit généreux, souscriptions temps réel natives (`.on('postgres_changes', ...)`), pas besoin d'écrire un backend custom pour un cas d'usage mono-utilisateur comme celui-ci.
2. **Firebase / Firestore** — alternative équivalente, écosystème Google, aussi bon support realtime.
3. Dans les deux cas : il faudra une **authentification simple** (email/mot de passe ou magic link suffit, un seul utilisateur réel) pour rattacher les données à Tanor et sécuriser l'accès.

### Points d'attention pour la migration

- **Ne pas perdre les données existantes.** Tanor utilise déjà l'app en production. Le module Export/Import (déjà construit, voir ci-dessous) permet d'exporter un JSON complet de son état actuel — s'en servir comme point de départ pour un script de migration one-shot vers la nouvelle base.
- **Le format PDF, les calculs de compte OHADA, la logique de statut des factures/reçus** sont éprouvés et testés (voir historique) — ne pas les réinventer, juste changer la couche de persistance (`localStorage.getItem/setItem` → appels API/SDK).
- **jsPDF reste côté client** — pas de raison de faire générer les PDF côté serveur, ça ajouterait de la complexité sans bénéfice ici.
- **Le bug emoji/jsPDF documenté plus haut** doit rester en tête si de nouveaux écrans PDF sont ajoutés.

### Module Export/Import déjà en place (fonctionne dès aujourd'hui, indépendamment du chantier sync)

Sur le tableau de bord : boutons "Exporter une sauvegarde" (télécharge un JSON horodaté de toutes les clés `bp_*`) et "Restaurer une sauvegarde" (réimporte avec confirmation, remplace tout). Une bannière d'alerte apparaît si aucune sauvegarde n'a été faite depuis 14 jours. C'est un filet de sécurité manuel qui reste utile même après la mise en place d'un vrai backend (sauvegarde de secours supplémentaire).

---

## Fichiers à emporter dans Claude Code

- `BaoPixel_Studio_Facturation.html` — l'app complète actuelle, code source de référence pour tout ce qui est logique métier (calculs, PDF, statuts, comptes OHADA).

## Ce qui n'a PAS besoin d'être reconstruit

Toute la logique métier déjà validée : calcul des devis/factures, génération PDF (avec le fix emoji), gestion des statuts et conversions devis→facture, calcul du journal de trésorerie et du solde qui roule, mapping des comptes OHADA, génération du rapport comptable. Le travail à Claude Code porte sur la **couche de persistance et de synchronisation**, pas sur la refonte des fonctionnalités.
