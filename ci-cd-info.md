# CI/CD – Explications, KPIs & Analyse

## Objectif
Mettre en place une chaîne CI/CD qui :
- **Vérifie les PR** (tests + analyse qualité) avant merge.
- **Construit** le front & le back et **publie** des images Docker sur Docker Hub après merge sur `main`.
- **Automatise la couverture de code** (LCOV côté front, JaCoCo côté back) et l’expose dans SonarQube/SonarCloud.

---

## 1) Étapes du workflow & objectifs

### Frontend (`.github/workflows/frontend-ci.yml`)
Déclencheurs :
- `pull_request` vers `main` → **CI de validation PR**
- `push` sur `main` → **CD** (incluant push d’image)
- `workflow_dispatch` → exécution manuelle

Étapes (dans l’ordre) :
1. **Checkout**  
   *Récupère le code.*

2. **Setup Node 20 + cache npm**  
   *Environnement reproductible + install plus rapide.*

3. **Install dependencies (`npm ci`)**  
   *Installation propre via `package-lock.json`.*

4. **Setup Chrome (CI)**  
   *Navigateur headless pour Karma.*

5. **(Optionnel) Build**  
   *Construit l’app pour détecter les erreurs de build tôt.*

6. **Génération `karma-ci.conf.js`**  
   *Active reporters `coverage` + `junit`, configure ChromeHeadlessNoSandbox.*

7. **Tests (Karma) avec couverture**  
   *Produit `coverage/**/lcov.info` (LCOV) + JUnit XML.*

8. **Test report (dorny/test-reporter)**  
   *Affiche un résumé des tests dans l’onglet Actions, échoue si tests ko.*

9. **(Optionnel) Upload coverage + dist**  
   *Artifacts téléchargeables (ne sont pas requis par Sonar).*

10. **SonarQube scan**  
    *Analyse qualité + ingestion de la couverture LCOV via `sonar.javascript.lcov.reportPaths`.*

11. **Login Docker Hub** *(seulement sur `push` vers `main`)*  
    *Authentification pour publier l’image.*

12. **Build & Push Docker** *(seulement sur `push` vers `main`)*  
    *Construit l’image depuis `front/Dockerfile` et la pousse sur Docker Hub (`:latest`).*

---

### Backend (`.github/workflows/backend-ci.yml`)
Déclencheurs identiques (PR, push sur `main`, manuel).

Étapes (dans l’ordre) :
1. **Checkout**
2. **Setup JDK 11 + cache Maven**
3. **Cache SonarQube**
4. **Build, tests & JaCoCo**  
   *`mvn clean verify` + `jacoco:report` → génère `target/site/jacoco/jacoco.xml`.*

5. **SonarQube analysis**  
   *Analyse qualité + couverture (assurez-vous que Sonar lit bien le XML JaCoCo, via `pom.xml` ou `-Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml`).*

6. **Test report (dorny/test-reporter)**  
   *Parse Surefire XML → résumé & annotations ; échoue si tests ko.*

7. **(Optionnel) Upload JaCoCo HTML**  
   *Artifact de confort.*

8. **Login Docker Hub** *(seulement sur `push` vers `main`)*
9. **Build & Push Docker** *(seulement sur `push` vers `main`)*  
   *Construit l’image depuis `back/Dockerfile` et la pousse sur Docker Hub (`:latest`).*

**Règle d’or appliquée :** pas de `continue-on-error` ni `if: always()` → **chaque étape doit réussir** avant de passer à la suivante. Les étapes Docker ne s’exécutent **que si** toutes les étapes précédentes ont réussi **et** que l’événement est un `push` sur `main`.

---

## 2) KPIs proposés (avec seuils)

Bob souhaite fixer des seuils. Voici 4 KPIs recommandés (au moins 2 requis, dont la couverture minimale) :

1. **Couverture minimale sur le code nouveau** *(obligatoire)*
    - **Seuil proposé** : **≥ 80 %** (front et back).
    - Mise en œuvre : Quality Gate Sonar → *Coverage on New Code ≥ 80%*.

2. **Aucune issue bloquante/critique sur le code nouveau**
    - **Seuil** : **0 blocker / 0 critical** (Bugs, Vulnerabilities).
    - Mise en œuvre : Quality Gate Sonar (conditions sur “New Code”).

3. **Taux de réussite pipeline CI**
    - **Seuil** : **≥ 95 %** des exécutions passent sur 30 jours.
    - Mesure : onglet Actions GitHub → succès/échecs par workflow.

4. **Durée médiane du pipeline**
    - **Seuil** : **≤ 10 minutes** (front) / **≤ 12 minutes** (back).
    - Mesure : durée des jobs GitHub Actions (réduire via cache & build Docker optimisé).

*(Optionnel)* KPIs additionnels côté qualité : **Maintainability Rating = A** sur nouveau code ; **Duplications on New Code = 0%**.

---

## 3) Analyse des premières métriques (modèle)
> Renseignez ce tableau après la première exécution complète des pipelines (post-merge sur `main`). Les valeurs sont visibles dans Sonar et dans GitHub Actions.

### 3.1 Frontend – métriques initiales
| Indicateur | Valeur | Source |
|---|---:|---|
| Coverage global | `…%` | Sonar (LCOV) |
| Coverage sur nouveau code | `…%` | Sonar |
| Bugs / Vuln / Code Smells (new code) | `B: … / V: … / CS: …` | Sonar |
| Durée pipeline | `… min … s` | GitHub Actions |
| Statut Quality Gate | ✅ / ❌ | Sonar |

### 3.2 Backend – métriques initiales
| Indicateur | Valeur | Source |
|---|---:|---|
| Coverage global | `…%` | Sonar (JaCoCo) |
| Coverage sur nouveau code | `…%` | Sonar |
| Bugs / Vuln / Code Smells (new code) | `B: … / V: … / CS: …` | Sonar |
| Durée pipeline | `… min … s` | GitHub Actions |
| Statut Quality Gate | ✅ / ❌ | Sonar |

**Écarts vs KPIs**
- Couverture min (80%) : **Atteint / À améliorer** → action : écrire/renforcer des tests sur modules `X/Y`.
- 0 issue critique/bloquante : **OK / KO** → action : corriger règles Sonar `SXXXX`, `java:SXXXX`, etc.
- Durée pipeline : **OK / Trop long** → action : améliorer cache, limiter artefacts, optimiser Dockerfile.

---

## 4) Notes & avis utilisateurs (exemples)
- **Dev Front** : “Les annotations de tests via `dorny/test-reporter` aident à corriger rapidement les specs qui échouent.”
- **Dev Back** : “Le rapport JaCoCo HTML (artifact) est pratique pour explorer les lignes non couvertes.”
- **PO** : “Le Quality Gate Sonar (80% sur nouveau code) est lisible et utile pour valider les PR.”

**Problèmes prioritaires identifiés**
1. Coverage nouveau code back à **72%** (objectif 80%) → écrire des tests unitaires pour `ServiceX`.
2. 1 vulnérabilité critique dans le front (règle Sonar) → upgrade dépendance `…`.
3. Durée pipeline front à **14 min** → activer cache Docker Buildx ou supprimer l’upload `dist` si inutile.

---

## 5) Où trouver quoi ?

- **SonarQube/SonarCloud** : couverture, Quality Gate, issues (Bugs/Vuln/Smells).
- **GitHub Actions** : statut CI, résumé tests (JUnit), durée des jobs, artifacts (LCOV/HTML, JaCoCo/HTML).
- **Docker Hub** : images `bobapp-front:latest` et `bobapp-back:latest` (et éventuellement `:<commit-sha>` si vous ajoutez ce tag).

---

## 6) Annexes techniques

- **Protection de branche** : GitHub → Settings → Branches → *Protect `main`* → *Require PR* + *Require status checks to pass* (sélectionner vos jobs Frontend/Backend).
- **Conditions Docker** : les étapes Docker (`login`, `build-push`) s’exécutent uniquement sur `push` vers `main` :
  ```yaml
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  ```
- **Couverture**
    - Front : Karma `coverageReporter` → `lcov.info`, Sonar lit `sonar.javascript.lcov.reportPaths=coverage/**/lcov.info`.
    - Back : JaCoCo `jacoco.xml`, Sonar lit `sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml` (dans `pom.xml` ou via `-D…`).

---

### Comment utiliser ce document
1. Commitez ce fichier dans `docs/` (ou dans le README).
2. Après la **première exécution** post-merge, **complétez les tableaux** de métriques (sections 3.1/3.2) et la section *Notes & avis*.
3. Ajustez les **KPIs** si nécessaire et activez/paramétrez le **Quality Gate** équivalent dans Sonar.