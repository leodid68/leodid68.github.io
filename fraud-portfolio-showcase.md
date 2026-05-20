# Banking Fraud Detection — Portfolio Showcase

> Sélection curatée des analyses les plus probantes du projet VILNIUS. Pour le carnet de bord complet (toutes les requêtes EDA, hypothèses testées, fausses pistes, méthodologie détaillée), voir [`07_EDA_notes_VILNIUS.md`](./07_EDA_notes_VILNIUS.md).

---

## La Big Idea

**97% des pertes financières viennent de 30% des cas de fraude — concentrés sur les virements (mule, account takeover, sim swap). Sur la fraude par carte, 92% des cas ont contourné le 3D Secure. Recommandation : rendre 3DS obligatoire au-dessus de 200€ sur les cartes ET renforcer l'authentification sur les virements >5 000€ → élimination théorique de plus de 95% du coût de la fraude.**

---

## Méthodologie

- **Dataset** : 5 000 clients, 501 134 transactions sur 24 mois (mai 2024 → avril 2026), 372 cas de fraude confirmés couvrant 7 typologies (card_testing, mule, velocity, account_takeover, card_not_present, friendly_fraud, sim_swap).
- **Stack** : PostgreSQL (Supabase cloud), DataGrip, Looker Studio — 9 vues analytiques déployées en production.
- **Approche** : EDA progressive en 4 phases — volumétrie de base, analyses descriptives, investigations croisées, requêtes avancées (window functions, CTE, scoring).
- **Livrables** : 6 requêtes portfolio (ci-dessous) + 9 vues Supabase + 5 dashboards Looker Studio.

---

## Les 6 analyses-phares

### Analyse 1 — 3D Secure neutralise 7 typologies sur 8

> **Insight** : 92% des fraudes carte ont contourné 3DS. La seule typologie immune à 3DS est le friendly fraud — par construction, puisque la victime authentifie elle-même la transaction qu'elle contestera ensuite.

```sql
SELECT
    ROUND(AVG(fr.amount_disputed), 0) AS avg_loss,
    t.is_3ds_authenticated,
    c.card_tier,
    fr.investigator_notes AS pattern,
    COUNT(*) AS n_cas
FROM fraud_reports fr
JOIN transactions t ON t.transaction_id = fr.transaction_id
JOIN cards c        ON c.card_id        = t.card_id
GROUP BY fr.investigator_notes, c.card_tier, t.is_3ds_authenticated
ORDER BY n_cas DESC;
```

**Résultat clé** :

| 3DS | Pattern | Cas | Avg loss (€) |
| :--- | :--- | :--- | :--- |
| FALSE | card_testing (toutes tiers) | 137 | 80 |
| FALSE | velocity (toutes tiers) | 83 | 470 |
| FALSE | card_not_present (toutes tiers) | 13 | 1 400 |
| **TRUE** | **friendly_fraud** (toutes tiers) | **19** | **375** |

**Recommandation chiffrée** : rendre 3DS obligatoire au-dessus de 200€ éliminerait théoriquement 234 cas sur 253 fraudes carte (~92% des pertes carte).

**Techniques SQL** : triple JOIN, GROUP BY multi-dimensions, ROUND, AVG.

---

### Analyse 2 — Le paradoxe volume / coût : 3 régimes économiques distincts

> **Insight** : la fraude bancaire n'est pas un phénomène unifié mais trois mondes économiques différents qui demandent chacun une défense spécifique.

```sql
SELECT
    fr.fraud_type,
    COUNT(*) AS n_cas,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) AS pct_volume,
    c.card_tier,
    ROUND(AVG(fr.amount_disputed), 0) AS avg_loss
FROM fraud_reports fr
JOIN transactions t  ON fr.transaction_id = t.transaction_id
LEFT JOIN cards c    ON t.card_id = c.card_id
WHERE fr.status = 'confirmed'
GROUP BY fr.fraud_type, c.card_tier
ORDER BY avg_loss DESC;
```

**Trois régimes** :

| Régime | Typologies | % volume | Avg / cas (€) | % pertes totales |
| :--- | :--- | :--- | :--- | :--- |
| **Spray and pray** | card_testing | 37% | 80 | ~1.5% |
| **Big game** | account_takeover, sim_swap, mule | 31% | 4 000 – 8 400 | **~97%** |
| **Mid-range** | velocity, friendly_fraud, CNP | 32% | 380 – 1 900 | ~1.5% |

**Techniques SQL** : window function `SUM(COUNT(*)) OVER ()`, GROUP BY multi-dimensions, LEFT JOIN.

---

### Analyse 3 — La fraude carte est opportuniste, pas ciblée

> **Insight** : la distribution des fraudes carte par tier suit quasi-exactement la distribution naturelle des cartes émises. Les fraudeurs tirent au hasard.

```sql
SELECT
    COALESCE(c.card_tier, 'wire') AS tier,
    COUNT(*) AS n_cas,
    ROUND(AVG(fr.amount_disputed), 0) AS avg_loss
FROM fraud_reports fr
JOIN transactions tr ON tr.transaction_id = fr.transaction_id
LEFT JOIN cards c    ON c.card_id        = tr.card_id
WHERE fr.status = 'confirmed'
GROUP BY c.card_tier;
```

**Distribution observée vs population** :

| Tier | Part en population | Part de la fraude |
| :--- | :--- | :--- |
| classic | 52% | 52% |
| gold | 25% | 10% |
| platinum | 12% | 7% |
| world | 7% | 3% |

**Recommandation** : mitigation comportementale (vélocité, géographie, anomalies) plutôt que règles par tier de carte.

**Techniques SQL** : COALESCE, LEFT JOIN, comparaison avec dataset de référence.

---

### Analyse 4 — Les biens digitaux concentrent 38% de la fraude carte

> **Insight** : les catégories digitales (direct marketing, streaming, tickets, gaming) cumulent 38% des cas de fraude carte. Cohérent avec une économie de la fraude qui privilégie la livraison immédiate.

```sql
SELECT
    COALESCE(m.subcategory, 'wire/transfer') AS subcategory,
    COUNT(*) AS n_cas,
    ROUND(SUM(fr.amount_disputed), 0) AS total_lost_eur,
    ROUND(AVG(fr.amount_disputed), 0) AS avg_loss
FROM fraud_reports fr
JOIN transactions t  ON fr.transaction_id = t.transaction_id
LEFT JOIN merchants m ON t.merchant_id = m.merchant_id
WHERE fr.status = 'confirmed'
GROUP BY m.subcategory
ORDER BY n_cas DESC
LIMIT 10;
```

**Top des cibles** :

| Subcategory | Cas | Total perdu (€) | Avg / cas |
| :--- | :--- | :--- | :--- |
| wire/transfer | 120 | 577 547 | 4 813 |
| direct_mkt | 40 | 13 683 | 342 |
| streaming | 22 | 7 055 | 321 |
| tickets | 19 | 4 415 | 232 |
| airline | 18 | 7 129 | 396 |
| gambling | 16 | 5 625 | 352 |

**Techniques SQL** : LEFT JOIN avec gestion des NULL, COALESCE, agrégations multiples.

---

### Analyse 5 — Signature Account Takeover : le changement précède la fraude

> **Insight** : une proportion significative des fraudes wire (account_takeover, sim_swap, mule) est précédée d'un changement d'email ou de téléphone dans les 24h — signal d'alerte puissant.

```sql
WITH fraud_tx AS (
    SELECT
        fr.fraud_id,
        fr.customer_id,
        fr.fraud_type,
        fr.amount_disputed,
        t.transaction_timestamp AS fraud_time
    FROM fraud_reports fr
    JOIN transactions t ON t.transaction_id = fr.transaction_id
    WHERE fr.status = 'confirmed'
      AND fr.fraud_type IN ('account_takeover', 'sim_swap', 'mule')
)
SELECT
    ft.fraud_type,
    COUNT(DISTINCT ft.fraud_id) AS total_fraud,
    COUNT(DISTINCT ft.fraud_id) FILTER (WHERE EXISTS (
        SELECT 1 FROM customer_changes cc
        WHERE cc.customer_id = ft.customer_id
          AND cc.field_name IN ('email', 'phone', 'password')
          AND cc.changed_at BETWEEN ft.fraud_time - INTERVAL '24 hours' AND ft.fraud_time
    )) AS preceded_by_change,
    ROUND(100.0 * COUNT(DISTINCT ft.fraud_id) FILTER (WHERE EXISTS (
        SELECT 1 FROM customer_changes cc
        WHERE cc.customer_id = ft.customer_id
          AND cc.field_name IN ('email', 'phone', 'password')
          AND cc.changed_at BETWEEN ft.fraud_time - INTERVAL '24 hours' AND ft.fraud_time
    )) / NULLIF(COUNT(DISTINCT ft.fraud_id), 0)::numeric, 1) AS pct_preceded
FROM fraud_tx ft
GROUP BY ft.fraud_type
ORDER BY pct_preceded DESC;
```

**Techniques SQL** : CTE, `COUNT FILTER (WHERE EXISTS(...))`, `INTERVAL`, sous-requête corrélée temporelle, NULLIF.

---

### Analyse 6 — Score de risque multi-signal (le morceau de bravoure)

> **Insight** : en combinant 5 signaux indépendants (3DS off, IP anonyme, pays différent, distance >500km, montant anomalie), on peut classifier chaque transaction de "safe" à "critical" sans ML.

```sql
WITH scored AS (
    SELECT
        tr.transaction_id,
        tr.customer_id,
        tr.amount_eur,
        tr.transaction_timestamp,
        CASE WHEN NOT tr.is_3ds_authenticated                                             THEN 1 ELSE 0 END AS flag_no_3ds,
        CASE WHEN ls.is_vpn OR ls.is_tor                                                  THEN 1 ELSE 0 END AS flag_anomalous_ip,
        CASE WHEN tr.ip_country IS NOT NULL AND tr.ip_country <> c.country                THEN 1 ELSE 0 END AS flag_different_country,
        CASE WHEN tr.distance_from_home_km > 500                                          THEN 1 ELSE 0 END AS flag_far_away,
        CASE WHEN tr.amount_eur > 3 * AVG(tr.amount_eur) OVER (PARTITION BY tr.customer_id)
                                                                                          THEN 1 ELSE 0 END AS flag_amount_anomaly
    FROM transactions tr
    JOIN customers c ON c.customer_id = tr.customer_id
    LEFT JOIN login_sessions ls ON ls.session_id = tr.session_id
),
risked AS (
    SELECT *,
           (flag_no_3ds + flag_anomalous_ip + flag_different_country + flag_far_away + flag_amount_anomaly) AS risk_score
    FROM scored
)
SELECT
    transaction_id, customer_id, amount_eur, risk_score,
    CASE
        WHEN risk_score >= 4 THEN 'critical'
        WHEN risk_score  = 3 THEN 'high'
        WHEN risk_score  = 2 THEN 'medium'
        WHEN risk_score  = 1 THEN 'low'
        ELSE                      'safe'
    END AS risk_level
FROM risked
WHERE risk_score >= 2
ORDER BY risk_score DESC, amount_eur DESC;
```

**Extrait résultats** :

| transaction_id | risk_score | risk_level | amount_eur |
| :--- | :--- | :--- | :--- |
| 700191870 | 4 | critical | 1971.49 |
| 700500764 | 4 | critical | 1437.40 |
| 700500451 | 4 | critical | 1360.43 |

**Techniques SQL** : CTE empilées, `AVG() OVER (PARTITION BY customer_id)`, scoring arithmétique multi-flags, CASE WHEN de classification.

---

## Recommandations chiffrées

### 1. Rendre 3DS obligatoire au-dessus de 200€ sur les cartes

- **Cas impactés** : ~234 fraudes carte sur 253 (92%)
- **Coût d'implémentation** : faible (acquéreurs supportent déjà 3DS)
- **Friction utilisateur** : modérée et déjà familière post-PSD2

### 2. Renforcer l'authentification sur les virements >5 000€

- **Cas impactés** : ~120 fraudes wire (mule + ATO + sim_swap)
- **Pertes potentiellement évitées** : ~95% des pertes totales
- **Mesures** : vérification out-of-band, délai de carence 24h nouveaux bénéficiaires, scoring du destinataire

### 3. Surveillance accrue sur les merchants digitaux haut-risque

- **Cible** : MCC direct marketing, streaming, gambling, gaming, tickets
- **Cas impactés** : ~111 fraudes carte (38% du total carte)
- **Mesure** : règles de vélocité plus strictes + 3DS step-up automatique

---

## Compétences SQL démontrées

| Catégorie | Techniques | Démontré dans |
| :--- | :--- | :--- |
| **Jointures** | INNER JOIN, LEFT JOIN, triple JOIN | Analyses 1, 2, 3, 4 |
| **Window functions** | `SUM(COUNT(*)) OVER ()`, `AVG() OVER (PARTITION BY)`, `ROW_NUMBER()`, `LAG()` | Analyses 2, 6 + EDA 4.2, 4.6 |
| **CTE** | Simple et empilées (`WITH ... AS`) | Analyses 5, 6 |
| **Gestion des NULL** | COALESCE, LEFT JOIN avec interprétation, NULLIF | Analyses 3, 4, 5 |
| **Date arithmetic** | `EXTRACT(EPOCH FROM ...)`, `AGE()`, `INTERVAL`, soustraction timestamps | Analyse 5 + EDA 3.1, 4.1 |
| **Agrégations** | COUNT, SUM, AVG, ROUND avec casting, FILTER | Toutes |
| **Logique conditionnelle** | CASE WHEN, scoring pondéré multi-flags | Analyse 6 |
| **EXISTS / corrélation** | Sous-requêtes corrélées temporelles | Analyse 5 |
| **Multi-dimensions** | GROUP BY sur 2-3 colonnes | Analyses 1, 2 |
| **Vues & matérialisées** | 9 vues déployées sur Supabase pour Looker Studio | Ensemble du projet |

---

## Stack technique complète

- **Base de données** : PostgreSQL 17 (Supabase cloud, région eu-west-1)
- **IDE** : DataGrip (connexion via connection pooler Supabase)
- **Visualisation** : Looker Studio — 5 dashboards (Executive Overview, Fraud Patterns, Geographic, Customer Risk, Investigation Tool)
- **Lien Looker Studio** : https://datastudio.google.com/s/n3Pt2w_imAA
- **Vues analytiques** : 9 vues déployées (`mv_monthly_kpis`, `v_fraud_by_type`, `v_fraud_by_country`, `v_fraud_by_merchant`, `v_fraud_heatmap`, `v_3ds_impact`, `v_fraud_amount_hour`, `v_customer_risk`, `v_risk_score`)

---

## Limites et extensions

**Limites du dataset (synthétique)** :
- `root_cause` et `detection_method` générés uniformément, non corrélés au reste
- Pas de fraude card-present modélisée (skimming, vol de carte physique)

**Extensions possibles** :
- Layer ML : régression logistique scikit-learn sur les features identifiées pour comparer aux règles SQL
- dbt : encapsuler les transformations en modèles versionnés
- Streaming : Kafka + ksqlDB pour détection temps-réel

---

## Préparation entretien

1. **Commencer par l'Analyse 2** (les 3 régimes) — c'est la grande idée du projet, elle capte l'attention.
2. **Puis l'Analyse 1** (3DS) — recommandation chiffrée, montre l'impact business.
3. **Puis l'Analyse 6** (score de risque) — démontre la maîtrise technique avancée.
4. Laisser l'intervieweur conduire vers les autres analyses selon ses intérêts.

Ne pas montrer tous les dashboards d'emblée — commencer par le Dashboard 1 (exec), puis proposer d'aller plus profond.
