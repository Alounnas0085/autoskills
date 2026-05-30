# Fondamentaux à enseigner lors d'un audit

Ce document liste les concepts clés à expliquer quand ils apparaissent dans un audit de code IA.
Chaque concept inclut une définition courte et un exemple concret.

## Architecture

**Séparation des responsabilités (SRP)**
Une fonction, classe ou module ne fait qu'une chose. Si tu dois utiliser "et" pour décrire ce que fait un composant, c'est un signal d'alerte.

**Couplage vs Cohésion**
- Couplage fort = les modules dépendent les uns des autres de façon rigide → difficile à changer
- Cohésion forte = tout ce qui appartient ensemble est ensemble → facile à comprendre

**Couches applicatives**
Présentation → Logique métier → Accès aux données. Chaque couche ne connaît que la suivante, jamais en sens inverse.

## Sécurité

**Injection**
Ne jamais construire des requêtes ou commandes par concaténation de chaînes. Utiliser les requêtes préparées, les ORM, les validateurs.

**Principe du moindre privilège**
Un composant ne doit avoir accès qu'à ce dont il a strictement besoin — ni plus.

**Validation à la frontière**
Tout ce qui vient de l'extérieur (utilisateur, API, fichier) est suspect. Valider, typer, assainir avant tout traitement interne.

## Design Patterns

**Quand utiliser un pattern**
Un pattern résout un problème récurrent connu. L'IA sur-utilise souvent les patterns — demande toujours : quel problème concret ce pattern résout-il ici ?

**Antipatterns fréquents générés par l'IA**
- **God Object** : une classe qui fait tout et sait tout
- **Abstraction prématurée** : créer des interfaces pour un seul cas d'usage
- **Callback hell / Promise hell** : logique asynchrone imbriquée illisible

## Bases de données

**Problème N+1**
Faire 1 requête pour récupérer une liste, puis N requêtes pour les détails de chaque élément. Solution : JOIN ou chargement eager.

**Index**
Les colonnes utilisées dans WHERE, JOIN, ORDER BY doivent être indexées. L'IA oublie souvent de les mentionner.

**Logique métier dans la BDD**
Les triggers et procédures stockées cachent la logique — préférer la logique dans l'application pour la maintenabilité.

## Logique produit

**Pourquoi comprendre les utilisateurs avant de coder**
L'IA génère du code pour le problème que tu lui décris. Si ta description est mauvaise, le code sera parfait... pour le mauvais problème.

**Ce que les utilisateurs font vraiment**
Les utilisateurs contournent les problèmes avec des solutions bricolées (Excel, copier-coller, mémoire). Identifier ces workarounds révèle le vrai besoin.

**La sensibilité produit**
Savoir pourquoi quelque chose est construit, pour qui, et ce qui compte vraiment — c'est ce qui permet de trancher quand l'IA propose 3 solutions équivalentes techniquement.
