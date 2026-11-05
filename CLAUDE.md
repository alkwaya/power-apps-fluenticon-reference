# CLAUDE.md - Projet « Payflows » (Power Apps, SharePoint & Power BI)

Bienvenue dans le dépôt du projet **Payflows**. Ce document résume l'architecture, les spécifications techniques, la structure de données et les standards de code pour développer et maintenir l'application dans l'écosystème **Microsoft Power Platform**.

---

## 1. Vue d'Ensemble du Projet

- **Nom de l'application** : Payflows
- **Rôle** : Système d'émission, de validation, de suivi et d'impression d'instructions de paiement (ordres de virement ponctuels, lots d'honoraires consultants, lots de salaires).
- **Backend de données** : SharePoint Online (7 listes normalisées, indexées, sans risque de dépassement de délégation).
- **Frontend** : Microsoft Power Apps (Application Canvas avec Contrôles Modernes Fluent 2).
- **Cible Matérielle Principale** : Écrans macOS (iMac 24 pouces et MacBook 14 pouces) avec mise en page responsive via Conteneurs Flex.
- **Reporting & Business Intelligence** : Microsoft Power BI (Modélisation en étoile, mesures DAX, intégré directement via la vignette Power BI).
- **Génération Documentaire** : Impression directe haute fidélité via la fonction native `Print()` reproduisant exactement le gabarit `IP.png`.

---

## 2. Charte Graphique & UI/UX (Microsoft Fluent 2)

| Élément | Couleur / Valeur | Usage |
| :--- | :--- | :--- |
| **Deep Navy** | `#0A2540` / `#06182A` | Bandeau d'en-tête, barre latérale (Sidebar) de navigation |
| **Primary Blue** | `#0078D4` / `#106EBE` | Boutons d'action principaux, onglets actifs, sélecteurs |
| **Canvas Background**| `#F4F6F9` | Fond d'écran global (neutre doux pour le confort visuel) |
| **Surface Card** | `#FFFFFF` | Conteneurs de données, cartes de listes, fenêtres modales |
| **Border / Divider** | `#E1DFDD` | Bordures nettes de cartes et séparateurs de champs |
| **Dark Neutral** | `#242424` | Titres, textes principaux, montants |
| **Secondary Neutral**| `#605E5C` | Libellés de champs, métadonnées, sous-titres |
| **Success Status** | `#107C41` (Fond `#DFF6DD`)| Badge statut « Validé », « Exécuté » |
| **Warning Status** | `#797673` (Fond `#F3F2F1`)| Badge statut « En attente » |
| **Danger Status** | `#A80000` (Fond `#FDE7E9`)| Badge statut « Rejeté », « Annulé » |

### Typographie & Composants Modernes
- **Police** : *Segoe UI Variable* (ou Segoe UI).
- **Contrôles modernes recommandés** : `Button` (Primary / Subtle), `TabList` (navigation), `Badge` (statuts dynamiques), `DatePicker`, `TextInput`, `Table` / `Gallery` dans conteneurs automatiques.
- **Mise en page** : Orientation Paysage (16:9), résolution native 1920 × 1080 avec *Adapter à la taille = Désactivé* pour garantir une flexibilité parfaite sur iMac 24" et MacBook 14".

---

## 3. Architecture des Données SharePoint Online

### Modèle Relationnel des 7 Listes

```
[ParametresApp]
       │
[Projets] ───┐
[Banques] ───┼──> [InstructionsPaiement] (En-têtes : N° ordre, Date, Statut, Signataire)
[Signataires]┘              │ (1 à N via IdInstruction)
                            ▼
[Beneficiaires] ────> [LignesInstruction] (Détails : Bénéficiaire, RIB complet, Montant XOF, Motif)
```

### Schéma Détaillé des Colonnes

1. **`InstructionsPaiement`** (En-tête de l'ordre) :
   - `Title` (Texte unique, Indexé) : Code IP (ex: `IP-25-001`, `HON-25-06`).
   - `TypeDocument` (Choix) : `Instruction de Paiement`, `Honoraire`, `Salaire`.
   - `Annee` (Nombre) : Année fiscale (ex: `2025`).
   - `NumeroOrdre` (Nombre) : Séquence numérique (ex: `1`, `2`).
   - `DateEmission` (Date, Indexé) : Date de création.
   - `Ville` (Texte) : Ex: `Bamako`.
   - `ProjetCode` (Texte, Indexé) : Ex: `MLI/805`.
   - `BanqueEmettrice` (Texte) : Ex: `BNDA`.
   - `CompteDebiteur` (Texte) : Compte source à débiter.
   - `MontantTotal` (Nombre Décimal 0) : Somme totale en XOF.
   - `Devise` (Texte) : `XOF`.
   - `MontantEnLettres` (Plusieurs lignes texte) : Mention légale certifiée.
   - `SignataireNom` (Texte) : Nom du signataire approuvant.
   - `Statut` (Choix, Indexé) : `Brouillon`, `En attente`, `Validé`, `Exécuté`, `Annulé`.
   - `RefFacture` (Texte) & `RefBonCommande` (Texte).
   - `InstructionsSpeciales` (Texte multi-lignes).

2. **`LignesInstruction`** (Lignes individuelles du virement) :
   - `Title` (Texte) : Nom du bénéficiaire.
   - `IdInstruction` (Nombre, Indexé) : Clé étrangère vers l'ID d'`InstructionsPaiement`.
   - `CodeInstruction` (Texte, Indexé) : Code lisible (ex: `IP-25-001`).
   - `BeneficiaireNom` (Texte).
   - `CodeBanque` (Texte) : Code Banque (CB, ex: `ML043`).
   - `CodeGuichet` (Texte) : Code Guichet (CG, ex: `06600`).
   - `NumeroCompte` (Texte) : Numéro de compte à créditer.
   - `CleRIB` (Texte) : Clé de contrôle RIB.
   - `BanqueBeneficiaire` (Texte) : Ex: `BNDA`, `BMS`, `BDM`.
   - `AgenceVille` (Texte) : Agence et localité.
   - `Montant` (Nombre) : Montant unitaire en Francs CFA.
   - `Devise` (Texte) : `XOF`.
   - `MotifVirement` (Texte) : Ex: `Honoraires Juin 2025`.

3. **`Beneficiaires`** (Répertoire d'adresses et coordonnées bancaires) :
   - `Title` (Texte, Indexé) : Nom complet.
   - `Categorie` (Choix) : `Consultant / Expert`, `Salarié`, `Fournisseur`.
   - `CodeBanque`, `CodeGuichet`, `NumeroCompte`, `CleRIB`, `BanqueNom`, `AgenceVille`.
   - `Telephone`, `Email`, `StatutActif` (Oui/Non).

4. **`Projets`** :
   - `Title` (Code Projet, ex: `MLI/805`), `Description`, `Adresse`, `Telephone`, `BoitePostale`.

5. **`BanquesEmettrices`** :
   - `Title` (Nom Banque, ex: `BNDA`), `Ville`, `Agence`, `NumeroCompteDebiteur`, `Devise`.

6. **`Signataires`** :
   - `Title` (Nom complet, ex: `Boubou Bah DIABY`), `Fonction`, `Actif` (Oui/Non).

7. **`ParametresApp`** :
   - `Title` (`ConfigGlobale`), `ProjetParDefaut`, `BanqueParDefaut`, `DernierNumeroIP`, `DeviseDefaut`.

---

## 4. Règles & Extraits Clés Power Fx

### Initialisation Globale (`App.OnStart`)
```powerfx
Set(varCurrentUser, User());
Set(varCurrentTheme, {
    Primary: ColorValue("#0078D4"),
    HeaderNavy: ColorValue("#0A2540"),
    Background: ColorValue("#F4F6F9"),
    Surface: ColorValue("#FFFFFF"),
    Border: ColorValue("#E1DFDD")
});
Set(varConfig, LookUp(ParametresApp, Title = "ConfigGlobale"));
Set(varCurrentScreen, "TableauDeBord");
Clear(colLignesIP);
```

### Ajout d'une Ligne au Panier d'Instruction
```powerfx
Collect(
    colLignesIP,
    {
        LigneID: CountRows(colLignesIP) + 1,
        BeneficiaireNom: drp_Beneficiaire.Selected.Title,
        BanqueBeneficiaire: drp_Beneficiaire.Selected.BanqueNom,
        CB: drp_Beneficiaire.Selected.CodeBanque,
        CG: drp_Beneficiaire.Selected.CodeGuichet,
        NC: drp_Beneficiaire.Selected.NumeroCompte,
        CleRIB: drp_Beneficiaire.Selected.CleRIB,
        Montant: Value(txt_Montant.Value),
        Devise: "XOF",
        Motif: txt_Motif.Value,
        Lieu: drp_Beneficiaire.Selected.AgenceVille
    }
);
Reset(txt_Montant);
Reset(txt_Motif);
```

### Enregistrement Relationnel Parent-Enfant (Patch)
```powerfx
Set(varTotalIP, Sum(colLignesIP, Montant));
Set(varCodeIP, "IP-" & Text(Year(Today()), "yy") & "-" & Text(varConfig.DernierNumeroIP + 1, "000"));

// Création de l'en-tête parent
Set(
    varNewInstruction,
    Patch(
        InstructionsPaiement,
        Defaults(InstructionsPaiement),
        {
            Title: varCodeIP,
            TypeDocument: { Value: "Instruction de Paiement" },
            Annee: Year(Today()),
            NumeroOrdre: varConfig.DernierNumeroIP + 1,
            DateEmission: dp_DateEmission.Value,
            Ville: txt_Ville.Value,
            ProjetCode: drp_Projet.Selected.Title,
            BanqueEmettrice: drp_Banque.Selected.Title,
            CompteDebiteur: drp_Banque.Selected.NumeroCompteDebiteur,
            MontantTotal: varTotalIP,
            Devise: "XOF",
            SignataireNom: drp_Signataire.Selected.Title,
            Statut: { Value: "Validé" }
        }
    )
);

// Création des lignes enfants rattachées
ForAll(
    colLignesIP,
    Patch(
        LignesInstruction,
        Defaults(LignesInstruction),
        {
            Title: BeneficiaireNom,
            IdInstruction: varNewInstruction.ID,
            CodeInstruction: varCodeIP,
            BeneficiaireNom: BeneficiaireNom,
            CodeBanque: CB,
            CodeGuichet: CG,
            NumeroCompte: NC,
            CleRIB: CleRIB,
            BanqueBeneficiaire: BanqueBeneficiaire,
            AgenceVille: Lieu,
            Montant: Montant,
            Devise: Devise,
            MotifVirement: Motif
        }
    )
);

// Mise à jour de l'incrémenteur
Patch(ParametresApp, varConfig, { DernierNumeroIP: varConfig.DernierNumeroIP + 1 });
Set(varConfig, LookUp(ParametresApp, Title = "ConfigGlobale"));
```

---

## 5. Gabarit d'Impression Directe (`IP.png` & `Print()`)

L'application intègre un écran dédié `scr_PrintIP` configuré au format portrait A4 (794 × 1123 px).

### Déclenchement de l'Impression
```powerfx
Print()
```

### Masquage des Éléments d'Interface (Boutons, Menus)
Définir la propriété `Visible` des boutons sur :
```powerfx
!scr_PrintIP.Printing
```

### Mention Légale Obligatoire en Français
Formule de libellé dynamique affichée au bas du tableau de virement :
```powerfx
"Arrêté la présente instruction de virement à la somme de " &
Proper(fn_NombreEnLettresFR(varSelectedIP.MontantTotal)) & " Francs CFA"
```

---

## 6. Modélisation Power BI (Tableau de Bord Financier)

### Connexion
- Source : Connecteur **SharePoint Online List 2.0**.
- Charger : `InstructionsPaiement`, `LignesInstruction`, `Beneficiaires`, `Projets`.

### Mesures DAX Clés
```dax
// Montant Total
Total Decaisse = SUM(LignesInstruction[Montant])

// Nombre d'instructions
Nb Instructions = DISTINCTCOUNT(InstructionsPaiement[ID])

// Nombre de bénéficiaires
Nb Beneficiaires = DISTINCTCOUNT(LignesInstruction[BeneficiaireNom])

// Cumul annuel
Montant YTD = TOTALYTD([Total Decaisse], Dim_Calendrier[Date])
```

### Intégration dans Power Apps
- Publier le rapport sur Power BI Service.
- Épingler les visuels sur un tableau de bord `Payflows_Dashboard`.
- Dans Power Apps, ajouter le contrôle **Vignette Power BI (Power BI Tile)** et lier au tableau de bord.

---

## 7. Commandes et Déploiement

- **Export Solution** : Packager les listes SharePoint et l'application Canvas dans une solution Power Platform pour faciliter le passage Dev -> Test -> Prod.
- **Sauvegarde de la configuration** : Maintenir à jour les valeurs dans `ParametresApp`.
