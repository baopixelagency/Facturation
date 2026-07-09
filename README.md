# Facturation

BaoPixel Studio Facturation — outil de gestion (devis, factures, reçus, dépenses, trésorerie) de BaoPixel Digital Agency, conforme au régime comptable OHADA/SMT.

App en ligne : https://baopixelagency.github.io/Facturation/

## Architecture

Fichier HTML unique (`index.html`), JS vanilla, aucun build/npm. PDF généré côté client avec jsPDF. Les données sont mises en cache dans `localStorage` et synchronisées en temps réel entre appareils via Supabase (Postgres + Realtime + Auth).

## Configuration Supabase

Le schéma de base de données à exécuter dans le SQL Editor de Supabase se trouve dans [`supabase_schema.sql`](supabase_schema.sql). Les clés de connexion (URL + clé publique) sont dans `index.html`.
