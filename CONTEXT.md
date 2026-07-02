# CONTEXT.md — Générateur de factures Factur-X (projet portfolio)
 
> Ce fichier donne le contexte complet du projet à un agent IA (Cursor, Claude Code, etc.).
> Il doit être lu avant toute contribution au code.
 
---
 
## 1. Contexte
 
### 1.1 Pourquoi ce projet existe
 
La France impose une réforme de la facturation électronique B2B :
 
- **Septembre 2026** : toutes les entreprises doivent pouvoir *recevoir* des factures électroniques ; grandes entreprises et ETI doivent en *émettre*.
- **Septembre 2027** : PME et TPE doivent émettre à leur tour.
Une "facture électronique" au sens de la réforme n'est **pas un simple PDF** : c'est une facture dont les données sont structurées (norme européenne **EN 16931**) afin qu'un logiciel puisse les lire sans OCR ni ressaisie.
 
Le format retenu pour ce projet est **Factur-X** : un fichier hybride composé d'un **PDF/A-3** (lisible par un humain) contenant en pièce jointe embarquée un **XML CII** (Cross Industry Invoice, lisible par une machine). Un seul fichier, deux lecteurs.
 
### 1.2 Écosystème de la réforme (schéma en Y)
 
- **OD (Opérateur de Dématérialisation)** : logiciel qui *fabrique* des factures conformes. **Ce projet est un OD.**
- **PA (Plateforme Agréée, ex-PDP)** : plateforme immatriculée par l'État qui *transmet* les factures. Le routage se fait via le SIREN du destinataire et l'annuaire central — jamais par email.
- **PPF** : concentrateur de l'État qui reçoit les données fiscales extraites des factures.
**Périmètre volontairement limité** : ce projet s'arrête à la frontière OD/PA. Il fabrique et valide des fichiers Factur-X conformes, mais ne transmet rien. La preuve de conformité est la **validation contre les schémas officiels**, pas l'envoi. L'intégration à une PA (ex. B2Brouter) est une évolution future hors périmètre.
 
### 1.3 Profils Factur-X
 
Du plus léger au plus complet : MINIMUM → BASIC WL → **BASIC** → EN 16931 → EXTENDED.
 
**Le projet cible le profil BASIC** : premier profil pleinement conforme EN 16931, avec lignes de facture et détail TVA. Suffisant pour une facture de freelance/TPE française.
 
---
 
## 2. But et objectifs
 
### 2.1 But
 
Application web qui permet de créer une facture simple via un formulaire et de télécharger un fichier **Factur-X valide** (PDF/A-3 + XML CII profil BASIC embarqué), conforme aux exigences françaises.
 
C'est un **projet portfolio** : la qualité d'ingénierie (architecture, tests, CI, documentation, justification des choix) compte autant que la fonctionnalité.
 
### 2.2 Objectifs fonctionnels (MVP)
 
1. **Formulaire de facture** :
   - Émetteur : raison sociale, adresse, SIREN/SIRET, n° TVA intracommunautaire
   - Client : deux modes — particulier (nom, adresse) ou entreprise (+ SIRET, n° TVA, requis pour le B2B)
   - Lignes : désignation, quantité, prix unitaire HT, taux de TVA (20 / 10 / 5.5 / 2.1 / 0)
   - Métadonnées : numéro de facture, date d'émission, date d'échéance, conditions de paiement
   - Mentions légales françaises obligatoires (pénalités de retard, indemnité forfaitaire de recouvrement 40 €, escompte, mention d'exonération de TVA le cas échéant — ex. art. 293 B du CGI pour la franchise en base)
2. **Génération du XML CII** profil BASIC (généré à la main depuis la spec, pas via une lib — choix pédagogique assumé)
3. **Génération du visuel PDF** de la facture
4. **Conversion PDF/A-3 + embedding** du XML (via la lib `atgp/factur-x` — l'embedding PDF/A-3 est le point le plus piégeux, on s'appuie sur la lib de référence)
5. **Validation** du fichier produit :
   - XSD (structure du XML)
   - Schematron (règles métier EN 16931 + profil BASIC, ex. cohérence des totaux)
   - Conformité PDF/A-3 (VeraPDF)
6. **Téléchargement** du fichier final
7. **Bonus (post-MVP)** : page "Valider mon Factur-X" — upload d'un fichier tiers, affichage du rapport de conformité
### 2.3 Objectifs non fonctionnels
 
- Architecture en couches : controllers fins → services métier → DTOs (pas de logique dans les controllers)
- Tests unitaires et fonctionnels significatifs (pas de tests cosmétiques)
- CI GitLab/GitHub : lint, analyse statique, tests, validation d'un fichier de référence
- README + write-up expliquant la réforme, les choix techniques et les limites
- Démo live hébergée (VPS/AlwaysData)
### 2.4 Hors périmètre (explicitement)
 
- Transmission via PA / appel d'API de plateforme
- Gestion de clients persistée / CRM
- Comptabilité, paiements, relances
- Multi-devise, multi-langue (EUR + français uniquement)
- Profils MINIMUM, EN 16931 complet, EXTENDED
- Signature électronique
---
 
## 3. Stack technique
 
- **Backend** : PHP 8.3+ / Symfony (dernière LTS ou stable), API REST
- **Frontend** : React + TypeScript (Vite)
- **PDF** : génération du visuel (ex. Dompdf ou wkhtmltopdf/Gotenberg — à trancher et documenter), embedding via `atgp/factur-x`
- **XML** : construction manuelle (DOMDocument ou XMLWriter) — pas de lib de génération, c'est un choix pédagogique à documenter
- **Validation** : XSD via DOMDocument::schemaValidate, Schematron (pipeline XSLT), VeraPDF en CLI
- **Infra** : Docker (dev), CI (PHPStan, PHP CS Fixer, ESLint, PHPUnit, Vitest), déploiement rsync
- **Pas de base de données pour le MVP** : la facture est construite en mémoire depuis le formulaire et restituée en fichier. (Si une persistance devient utile plus tard : SQLite/PostgreSQL.)
---
 
## 4. Architecture cible
 
```
frontend/                      # React + TS (Vite)
  src/
    components/InvoiceForm/    # formulaire (émetteur, client B2B/B2C, lignes, mentions)
    api/                       # client HTTP vers le backend
backend/
  src/
    Controller/
      InvoiceController.php    # POST /api/invoices → fichier Factur-X
      ValidationController.php # POST /api/validate → rapport (bonus)
    Dto/
      InvoiceDto.php           # données validées du formulaire (+ SellerDto, BuyerDto, LineDto)
    Service/
      Invoice/
        InvoiceCalculator.php  # totaux HT/TVA/TTC par taux, arrondis
      Xml/
        CiiXmlBuilder.php      # construction du XML CII profil BASIC
      Pdf/
        InvoicePdfRenderer.php # visuel PDF de la facture
      FacturX/
        FacturXAssembler.php   # PDF/A-3 + embedding XML (atgp/factur-x)
      Validation/
        XsdValidator.php
        SchematronValidator.php
        PdfAValidator.php      # wrapper VeraPDF
    resources/
      schemas/                 # XSD + Schematron officiels Factur-X BASIC (versionnés)
  tests/
    Unit/                      # calculs, builder XML
    Functional/                # endpoint → fichier valide de bout en bout
docs/
  writeup.md                   # récit du projet (réforme, choix, limites)
```
 
### Flux principal
 
```
Formulaire React
  → POST /api/invoices (JSON)
  → validation des données (contraintes Symfony sur le DTO)
  → InvoiceCalculator (totaux, arrondis)
  → CiiXmlBuilder (XML CII BASIC)
  → XsdValidator + SchematronValidator (fail fast si non conforme)
  → InvoicePdfRenderer (visuel)
  → FacturXAssembler (PDF/A-3 + embedding)
  → PdfAValidator
  → réponse : fichier facture-{numero}.pdf en téléchargement
```
 
Règle : **le backend ne renvoie jamais un fichier qui n'a pas passé toutes les validations.** Une facture non conforme = erreur 422 avec le détail des règles violées.
 
---
 
## 5. Points techniques critiques (pièges connus)
 
1. **PDF/A-3** : polices intégralement embarquées, pas de JavaScript, pas de transparence non déclarée, métadonnées XMP obligatoires (dont l'entrée XMP spécifique Factur-X : nom du fichier XML, profil, version). C'est le point d'échec n°1 des implémentations maison — d'où l'usage de `atgp/factur-x` pour l'assemblage.
2. **Nom du fichier embarqué** : `factur-x.xml` exactement, relation `Data` ou `Alternative` selon profil.
3. **Arrondis TVA** : les règles EN 16931 imposent la cohérence somme des lignes = totaux. Arrondir au niveau ligne puis sommer (et documenter ce choix). Utiliser des centimes en int ou BCMath — jamais de float pour les montants.
4. **Numéros identifiants** : SIREN = 9 chiffres, SIRET = 14, TVA intracom FR = FR + clé 2 chiffres + SIREN. Valider les formats ET les clés.
5. **Schematron** : la validation XSD seule ne suffit pas ; la plupart des règles métier (BR-*, BR-CO-*) sont dans le Schematron. Les deux sont obligatoires.
6. **Dates au format `102`** (AAAAMMJJ) dans le CII.
7. **Encodage** : UTF-8 partout, attention aux caractères spéciaux dans les raisons sociales.
---
 
## 6. Tests attendus
 
### Unitaires (PHPUnit)
- `InvoiceCalculator` : cas multi-taux, arrondis limites (ex. 3 lignes à 0,335 €), TVA 0% avec mention d'exonération
- `CiiXmlBuilder` : présence des balises obligatoires BASIC, valeurs correctement mappées, dates format 102
- Validateurs de SIREN/SIRET/TVA (clés valides et invalides)
### Fonctionnels (PHPUnit + client Symfony)
- POST facture valide B2B → 200, fichier retourné, XML extractible, XSD OK
- POST facture valide B2C → 200
- POST données invalides (SIRET faux, ligne sans prix, totaux incohérents) → 422 avec erreurs explicites
### Intégration / CI
- Fichier de référence généré dans la CI puis validé : XSD + Schematron + VeraPDF
- Frontend : Vitest sur le formulaire (validation des champs, toggle B2B/B2C)
### Validation croisée manuelle (documentée dans le write-up)
- Validateur en ligne FNFE-MPE
- Ouverture dans Adobe Reader (pièce jointe visible) et `pdfdetach -list`
---
 
## 7. Résultat attendu
 
- Un utilisateur remplit le formulaire et télécharge en quelques secondes un fichier `facture-XXXX.pdf` qui est :
  - lisible et propre visuellement,
  - un PDF/A-3 valide (VeraPDF pass),
  - porteur d'un `factur-x.xml` profil BASIC valide (XSD + Schematron pass),
  - porteur des mentions légales françaises obligatoires.
- Une CI verte qui prouve tout ça à chaque commit.
- Un write-up (`docs/writeup.md`) qui raconte : le contexte de la réforme, le rôle d'OD, les choix (XML à la main vs lib, choix du moteur PDF, gestion des arrondis), les pièges rencontrés, les limites et évolutions (intégration PA, profil EN 16931, page de validation publique).
---
 
## 8. Conventions pour l'agent IA
 
- **Langue** : code et identifiants en anglais, commentaires/documentation en français.
- **Commits** : conventionnels (`feat:`, `fix:`, `test:`, `docs:`, `refactor:`).
- **Pas de logique dans les controllers** : ils valident, délèguent au service, retournent une réponse.
- **Montants** : toujours en centimes (int) ou BCMath en interne ; conversion en décimal uniquement à la sérialisation XML/affichage.
- **Jamais de fichier retourné sans validation complète.**
- **Toute décision d'architecture non triviale** doit être notée dans `docs/writeup.md` (section décisions) au moment où elle est prise.
- **Les schémas officiels** (XSD/Schematron) sont versionnés dans le repo avec leur version et leur source notées ; ne pas les modifier.
- En cas de doute sur une règle métier EN 16931, ne pas inventer : marquer un `TODO(spec)` et le signaler.
---
 
## 9. Ressources de référence
 
- Spécification Factur-X (FNFE-MPE) — schémas XSD/Schematron par profil
- Norme EN 16931 (modèle sémantique + règles BR-*)
- Lib d'assemblage : `atgp/factur-x` (Packagist)
- Validation PDF/A : VeraPDF
- Réforme française : documentation impots.gouv.fr (facturation électronique, schéma en Y, annuaire, PA/PPF)