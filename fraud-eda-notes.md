# Projet VILNIUS : EDA SQL & Investigations Fraude

> **Carnet de bord d'exploration.** Ce fichier capture l'intégralité de la démarche d'EDA et d'investigation menée sur le dataset fraude bancaire. Pour la version "portfolio" focalisée sur les analyses-phares, voir [`08_portfolio_showcase.md`](./08_portfolio_showcase.md).

> Dataset : 5 000 clients, 501 134 transactions sur 24 mois (mai 2024 → avril 2026), 372 cas de fraude confirmés répartis en 7 typologies. Base PostgreSQL (Supabase), exploration via DataGrip.

---

## Table des matières

- [Limitations connues du dataset](#limitations-connues-du-dataset)
- [Techniques SQL démontrées](#techniques-sql-démontrées)
- [Vues analytiques créées](#vues-analytiques-créées)
- [Phase 1 : Volumétrie de base](#phase-1--volumétrie-de-base)
- [Phase 2 : Analyses descriptives transactionnelles](#phase-2--analyses-descriptives-transactionnelles)
- [Phase 3 : Investigations approfondies](#phase-3--investigations-approfondies)
- [Phase 4 : Requêtes avancées](#phase-4--requêtes-avancées)
- [Synthèse : Les 5 insights actionables](#synthèse--les-5-insights-actionables)

---

## Techniques SQL démontrées

| Catégorie | Techniques utilisées | Sections |
| :--- | :--- | :--- |
| **Bases** | SELECT, FROM, WHERE, GROUP BY, ORDER BY | partout |
| **Agrégats** | COUNT, SUM, AVG, ROUND | partout |
| **Jointures** | INNER JOIN, LEFT JOIN, triple JOIN | 2.3 → 3.7 |
| **Logique conditionnelle** | CASE WHEN, COALESCE | 3.2, 3.4, 4.3, 4.5 |
| **Tests de NULL** | IS NULL, IS NOT NULL | 1.1, 3.4 |
| **Date / temps** | EXTRACT(EPOCH FROM ...), AGE(), DATE_TRUNC, INTERVAL | 3.1, 4.1, 4.4 |
| **Window functions** | SUM() OVER (), AVG() OVER (PARTITION BY), LAG() | 3.7, 4.2, 4.3, 4.5 |
| **CTE** | WITH ... AS (simple et empilées) | 4.3, 4.5 |
| **EXISTS / sous-requête corrélée** | EXISTS(SELECT 1 FROM ... WHERE ...) | 4.4, mv_monthly_kpis |
| **FILTER** | COUNT(*) FILTER (WHERE ...) | mv_monthly_kpis |
| **Type casting** | ::numeric, ::int, ::date | partout |
| **Multi-dimensions** | GROUP BY sur 2-3 colonnes simultanément | 3.5, 3.6, 4.1 |
| **Gestion des NULL** | COALESCE, LEFT JOIN avec interprétation | 3.3, 3.4 |
| **Scoring pondéré** | CASE WHEN multi-flags + somme arithmétique | 4.5 |

---

## Vues analytiques créées

Toutes les vues ci-dessous sont déployées sur Supabase (projet `fraudsql`) et utilisées comme sources dans Looker Studio.

| Vue | Usage | Techniques clés |
| :--- | :--- | :--- |
| `mv_monthly_kpis` | Dashboard 1 : timeline transactions vs fraudes | DATE_TRUNC, COUNT FILTER, EXISTS |
| `v_fraud_by_type` | Dashboard 1 : donut répartition typologies | Window function SUM() OVER () |
| `v_fraud_by_country` | Dashboard 1 & 3 : choroplèthe | JOIN, GROUP BY multi-dim |
| `v_fraud_by_merchant` | Dashboard 1 & 2 : catégories marchands | LEFT JOIN, COALESCE |
| `v_fraud_heatmap` | Dashboard 2 : heatmap heure × jour | EXTRACT, double GROUP BY |
| `v_3ds_impact` | Dashboard 2 : impact 3DS | LEFT JOIN, multi-dim |
| `v_fraud_amount_hour` | Dashboard 2 : montant × heure | CASE WHEN buckets, EXTRACT |
| `v_customer_risk` | Dashboard 4 : profil risque client | AGE(), CASE WHEN, LEFT JOIN |
| `v_risk_score` | Dashboard 5 : scoring transaction | CTE empilées, AVG() OVER PARTITION BY |

---

## Limitations connues du dataset

Avant toute analyse, il est essentiel de noter les biais et imperfections identifiés pendant l'EDA.

- **`mobile_country_code`** : généré aléatoirement, sans corrélation avec la `nationality`. Inutilisable comme signal anti-fraude.
- **`root_cause`** : tiré uniformément au hasard, sans lien avec le type de fraude. Toute variation observée relève du bruit statistique.
- **`detection_method`** : également non corrélé à la nature de la fraude.
- **Distribution des comptes** : majoritairement courant (checking), reflète un profil de banque de détail classique.
- **Pas de fraude card-present** : 100% des fraudes sont CNP, le dataset ne modélise pas le skimming ou le vol physique.

---

## Phase 1 : Volumétrie de base

### 1.1 : Distribution des cartes par tier

```sql
SELECT cards.card_tier, COUNT(*) AS count
FROM cards
WHERE cards.card_tier IS NOT NULL
GROUP BY cards.card_tier
ORDER BY cards.card_tier;
```

| card_tier | count |
| :--- | :--- |
| business | 149 |
| classic | 3473 |
| gold | 1696 |
| platinum | 832 |
| world | 499 |

**Observation** : Classic domine à 52%, suivi de Gold (25%), Platinum (12%), World (7%), Business (2%). Cette distribution servira de référence pour comparer la distribution des victimes de fraude par tier.

---

### 1.2 : Distribution des comptes par type

```sql
SELECT account_type, COUNT(*)
FROM accounts
GROUP BY account_type
ORDER BY account_type ASC;
```

| account_type | count |
| :--- | :--- |
| checking | 5382 |
| investment | 421 |
| joint | 392 |
| savings | 1498 |

**Observation** : 70% de comptes courants, 19% d'épargne. Profil cohérent avec une banque de masse.

---

## Phase 2 : Analyses descriptives transactionnelles

### 2.1 : Origines déclarées de la fraude (`root_cause`)

```sql
SELECT fraud_reports.root_cause, COUNT(*)
FROM fraud_reports
GROUP BY fraud_reports.root_cause
ORDER BY COUNT(*) DESC;
```

| root_cause | count |
| :--- | :--- |
| data_breach | 93 |
| unknown | 76 |
| social_engineering | 68 |
| device_compromise | 68 |
| phishing | 67 |

**Observation** : Distribution anormalement uniforme, ce qui révèle que `root_cause` est généré aléatoirement. Conservé comme illustration de l'importance du contrôle qualité préalable sur chaque colonne.

---

### 2.2 : Montant moyen par sous-type de transaction

```sql
SELECT AVG(amount), transaction_subtype
FROM transactions
GROUP BY transactions.transaction_subtype
ORDER BY AVG(amount) DESC;
```

| avg | transaction_subtype |
| :--- | :--- |
| 4704.60 | wire |
| 68.81 | card_not_present |
| 65.34 | atm |
| 64.98 | card_present |

**Observation** : Le panier moyen d'un wire est ~70× supérieur à celui d'un paiement carte, à cause de la contamination par les fraudes mule/ATO/sim_swap qui génèrent des virements de plusieurs milliers d'euros.

---

### 2.3 : Méthodes de détection par présence carte

```sql
SELECT
    fr.detection_method,
    COUNT(*) AS n_dm,
    t.is_card_present
FROM fraud_reports as fr
JOIN transactions as t ON fr.transaction_id = t.transaction_id
GROUP BY fr.detection_method, t.is_card_present;
```

| detection_method | n_dm | is_card_present |
| :--- | :--- | :--- |
| anomaly_detection | 35 | false |
| customer_report | 237 | false |
| external_alert | 9 | false |
| rule_based | 91 | false |

**Observation** : 100% des fraudes confirmées ont `is_card_present = FALSE`. 64% détectées via signalement client, le client reste la dernière ligne de défense.

---

## Phase 3 : Investigations approfondies

### 3.1 : Délai entre transaction et signalement

```sql
SELECT
    fr.root_cause,
    COUNT(*) AS n_fraudes,
    ROUND(AVG(EXTRACT(EPOCH FROM (fr.reported_date - t.transaction_timestamp)) / 3600)::numeric, 1) AS avg_delay_hours,
    ROUND(AVG(EXTRACT(EPOCH FROM (fr.reported_date - t.transaction_timestamp)) / 86400)::numeric, 2) AS avg_delay_days
FROM fraud_reports AS fr
JOIN transactions AS t ON t.transaction_id = fr.transaction_id
WHERE fr.status = 'confirmed'
GROUP BY fr.root_cause
ORDER BY avg_delay_hours DESC;
```

| root_cause | n_fraudes | avg_delay_hours | avg_delay_days |
| :--- | :--- | :--- | :--- |
| social_engineering | 49 | 183.1 | 7.63 |
| data_breach | 65 | 102.7 | 4.28 |
| device_compromise | 53 | 91.2 | 3.80 |
| unknown | 46 | 87.7 | 3.66 |
| phishing | 42 | 72.4 | 3.02 |

**Observation** : Écarts non significatifs statistiquement (root_cause généré aléatoirement). Démonstration de méthode, sur donnée réelle, social engineering devrait avoir le délai le plus long.

---

### 3.2 : Utilisation du VPN

```sql
SELECT
    COUNT(*),
    CASE WHEN is_vpn = true THEN 'VPN' ELSE 'Normal' END AS vpn_flag
FROM login_sessions
GROUP BY is_vpn
ORDER BY COUNT(*) DESC;
```

| count | vpn_flag |
| :--- | :--- |
| 183200 | Normal |
| 3738 | VPN |

**Observation** : 2% des sessions via VPN. Signal faible pris isolément, puissant en combinaison (cf. score de risque Phase 4).

---

### 3.3 : Catégories de marchands ciblées

```sql
SELECT m.subcategory,
       COUNT(*) AS count,
       ROUND(SUM(fr.amount_disputed), 0) AS total_lost_eur
FROM fraud_reports fr
JOIN transactions t ON fr.transaction_id = t.transaction_id
LEFT JOIN merchants m ON t.merchant_id = m.merchant_id
GROUP BY m.subcategory
ORDER BY count DESC;
```

**Résultats clés** : wire/transfer (null) = 120 cas, 577 547€ (~97% des pertes). Sur fraude carte : direct_mkt (40), streaming (22), tickets (19), gaming (14) = 38% des cas.

---

### 3.4 : Pertes par tier de carte

```sql
SELECT
    COALESCE(c.card_tier, 'wire'),
    ROUND(AVG(fr.amount_disputed), 0),
    COUNT(*)
FROM fraud_reports fr
JOIN transactions tr ON tr.transaction_id = fr.transaction_id
LEFT JOIN cards c ON c.card_id = tr.card_id
GROUP BY c.card_tier;
```

| tier | avg_loss | count |
| :--- | :--- | :--- |
| wire | 4813 | 120 |
| classic | 289 | 182 |
| gold | 333 | 35 |
| platinum | 277 | 24 |
| world | 390 | 10 |

**Observation** : Les tiers premium sont sous-représentés dans la fraude. La fraude carte est opportuniste, pas ciblée.

---

### 3.5 : Matrice fraud_type × card_tier

```sql
SELECT fr.fraud_type,
       COUNT(*) AS count,
       c.card_tier,
       ROUND(AVG(fr.amount_disputed), 0) AS avg_amount_disputed
FROM fraud_reports fr
JOIN transactions t ON fr.transaction_id = t.transaction_id
LEFT JOIN cards c ON t.card_id = c.card_id
GROUP BY c.card_tier, fr.fraud_type
ORDER BY avg_amount_disputed DESC;
```

**Trois régimes économiques** : spray-and-pray (card_testing ~80€/cas), big-game (ATO/sim_swap/mule 4k-8k€/cas), mid-range (velocity/CNP 400-1900€/cas).

---

### 3.6 : Matrice 3DS × card_tier × fraud_type

```sql
SELECT
    ROUND(AVG(fr.amount_disputed), 0),
    t.is_3ds_authenticated,
    c.card_tier,
    fr.investigator_notes,
    COUNT(*)
FROM fraud_reports fr
JOIN transactions t ON t.transaction_id = fr.transaction_id
JOIN cards c ON c.card_id = t.card_id
GROUP BY fr.investigator_notes, c.card_tier, t.is_3ds_authenticated;
```

**Observation** : `is_3ds_authenticated = TRUE` ⟺ friendly_fraud exclusivement. 92% des fraudes carte ont bypassé 3DS.

---

### 3.7 : Pondération des typologies (window function)

```sql
SELECT
    fraud_type,
    COUNT(*) AS n_cas,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) AS pct
FROM fraud_reports
WHERE status = 'confirmed'
GROUP BY fraud_type
ORDER BY n_cas DESC;
```

| fraud_type | n_cas | pct |
| :--- | :--- | :--- |
| card_testing | 96 | 37.6% |
| mule | 62 | 24.3% |
| velocity | 59 | 23.1% |
| card_not_present | 12 | 4.7% |
| account_takeover | 12 | 4.7% |
| friendly_fraud | 10 | 3.9% |
| sim_swap | 4 | 1.6% |

**Technique** : `SUM(COUNT(*)) OVER ()`, calcule le total sans sous-requête, idiome standard pour % du total en une passe.

---

## Phase 4 : Requêtes avancées

### 4.1 : Heatmap temporelle : heure × jour de la semaine

**Question** : La fraude a-t-elle une signature temporelle ? Pic nocturne attendu (2h-4h du matin).

```sql
SELECT
    EXTRACT(DOW  FROM tr.transaction_timestamp)::int AS day_of_week,  -- 0=dim, 6=sam
    EXTRACT(HOUR FROM tr.transaction_timestamp)::int AS hour_of_day,
    COUNT(*) AS nb_fraud
FROM transactions tr
JOIN fraud_reports fr ON fr.transaction_id = tr.transaction_id
WHERE fr.status = 'confirmed'
GROUP BY day_of_week, hour_of_day
ORDER BY day_of_week, hour_of_day;
```

**Observation** : Pics visibles le vendredi (jour 5) à 3h du matin (19 cas) et 4h (10 cas). Le samedi (jour 6) est le plus calme, cohérent avec un profil de fraude automatisée qui exploite les fenêtres de faible surveillance opérationnelle. **Technique** : double `EXTRACT`, double `GROUP BY`.

---

### 4.2 : Impossibilité géographique (LAG + window function)

**Question** : Détecter deux transactions sur la même carte à plus de 500 km de distance en moins d'une heure.

```sql
WITH tx_with_prev AS (
    SELECT
        transaction_id,
        card_id,
        customer_id,
        transaction_timestamp,
        city,
        merchant_country,
        amount_eur,
        distance_from_home_km,
        LAG(transaction_timestamp) OVER (PARTITION BY card_id ORDER BY transaction_timestamp) AS prev_timestamp,
        LAG(city)                  OVER (PARTITION BY card_id ORDER BY transaction_timestamp) AS prev_city,
        LAG(merchant_country)      OVER (PARTITION BY card_id ORDER BY transaction_timestamp) AS prev_country,
        LAG(distance_from_home_km) OVER (PARTITION BY card_id ORDER BY transaction_timestamp) AS prev_distance
    FROM transactions
    WHERE status = 'approved'
)
SELECT
    transaction_id,
    card_id,
    customer_id,
    prev_city                                                           AS city_before,
    city                                                                AS city_after,
    prev_country,
    merchant_country,
    ROUND(EXTRACT(EPOCH FROM (transaction_timestamp - prev_timestamp)) / 60) AS gap_minutes,
    distance_from_home_km,
    amount_eur
FROM tx_with_prev
WHERE prev_timestamp IS NOT NULL
  AND merchant_country <> prev_country
  AND EXTRACT(EPOCH FROM (transaction_timestamp - prev_timestamp)) < 3600  -- moins d'1h
  AND distance_from_home_km > 500
ORDER BY gap_minutes ASC
LIMIT 20;
```

**Techniques** : `LAG() OVER (PARTITION BY card_id ORDER BY ...)` pour récupérer la ligne précédente, `EXTRACT(EPOCH FROM ...)` pour calculer l'écart en secondes, CTE pour séparer la logique.

---

### 4.3 : Anomalie vs baseline client (CTE + AVG OVER PARTITION BY)

**Question** : Pour chaque transaction, est-ce que le montant dépasse 3× la moyenne historique du client ?

```sql
WITH baseline AS (
    SELECT
        transaction_id,
        customer_id,
        amount_eur,
        transaction_timestamp,
        channel,
        merchant_category,
        AVG(amount_eur) OVER (PARTITION BY customer_id)    AS customer_avg,
        STDDEV(amount_eur) OVER (PARTITION BY customer_id) AS customer_stddev
    FROM transactions
    WHERE status = 'approved'
),
anomalies AS (
    SELECT
        *,
        ROUND((amount_eur / NULLIF(customer_avg, 0))::numeric, 2) AS ratio_vs_avg
    FROM baseline
    WHERE amount_eur > 3 * customer_avg
)
SELECT
    a.transaction_id,
    a.customer_id,
    a.amount_eur,
    a.customer_avg,
    a.ratio_vs_avg,
    a.channel,
    a.merchant_category,
    fr.fraud_type
FROM anomalies a
LEFT JOIN fraud_reports fr ON fr.transaction_id = a.transaction_id
                           AND fr.status = 'confirmed'
ORDER BY a.ratio_vs_avg DESC
LIMIT 20;
```

**Techniques** : CTE empilées (`baseline` → `anomalies`), `AVG() OVER (PARTITION BY customer_id)`, `NULLIF` pour éviter la division par zéro, `STDDEV()` comme bonus.

---

### 4.4 : Signature Account Takeover : changement sensible précédant la fraude

**Question** : Combien de fraudes confirmées sont précédées d'un changement d'email/phone dans les 24h ?

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
    COUNT(DISTINCT ft.fraud_id)                                              AS total_fraud,
    COUNT(DISTINCT ft.fraud_id) FILTER (WHERE EXISTS (
        SELECT 1
        FROM customer_changes cc
        WHERE cc.customer_id = ft.customer_id
          AND cc.field_name IN ('email', 'phone', 'password')
          AND cc.changed_at BETWEEN ft.fraud_time - INTERVAL '24 hours'
                                AND ft.fraud_time
    ))                                                                       AS preceded_by_change,
    ROUND(100.0 * COUNT(DISTINCT ft.fraud_id) FILTER (WHERE EXISTS (
        SELECT 1
        FROM customer_changes cc
        WHERE cc.customer_id = ft.customer_id
          AND cc.field_name IN ('email', 'phone', 'password')
          AND cc.changed_at BETWEEN ft.fraud_time - INTERVAL '24 hours'
                                AND ft.fraud_time
    )) / NULLIF(COUNT(DISTINCT ft.fraud_id), 0)::numeric, 1)                AS pct_preceded
FROM fraud_tx ft
GROUP BY ft.fraud_type
ORDER BY pct_preceded DESC;
```

**Techniques** : CTE, `COUNT FILTER (WHERE EXISTS(...))`, `INTERVAL '24 hours'`, sous-requête corrélée temporelle, `NULLIF` anti division par zéro.

---

### 4.5 : Score de risque combiné (CTE empilées + scoring pondéré)

**Question** : Construire un score 0-5 par transaction en agrégeant 5 signaux, puis classer par niveau de risque.

```sql
WITH scored AS (
    SELECT
        tr.transaction_id,
        tr.customer_id,
        tr.amount_eur,
        tr.transaction_timestamp,
        tr.merchant_country,
        tr.channel,
        CASE WHEN NOT tr.is_3ds_authenticated                                          THEN 1 ELSE 0 END AS flag_no_3ds,
        CASE WHEN ls.is_vpn OR ls.is_tor                                               THEN 1 ELSE 0 END AS flag_anomalous_ip,
        CASE WHEN tr.ip_country IS NOT NULL AND tr.ip_country <> c.country             THEN 1 ELSE 0 END AS flag_different_country,
        CASE WHEN tr.distance_from_home_km > 500                                       THEN 1 ELSE 0 END AS flag_far_away,
        CASE WHEN tr.amount_eur > 3 * AVG(tr.amount_eur) OVER (PARTITION BY tr.customer_id)
                                                                                       THEN 1 ELSE 0 END AS flag_amount_anomaly
    FROM transactions tr
    JOIN customers c ON c.customer_id = tr.customer_id
    LEFT JOIN login_sessions ls ON ls.session_id = tr.session_id
),
risked AS (
    SELECT
        *,
        (flag_no_3ds + flag_anomalous_ip + flag_different_country + flag_far_away + flag_amount_anomaly) AS risk_score
    FROM scored
)
SELECT
    transaction_id,
    customer_id,
    transaction_timestamp,
    amount_eur,
    flag_no_3ds,
    flag_anomalous_ip,
    flag_different_country,
    flag_far_away,
    flag_amount_anomaly,
    risk_score,
    CASE
        WHEN risk_score >= 4 THEN 'critical'
        WHEN risk_score  = 3 THEN 'high'
        WHEN risk_score  = 2 THEN 'medium'
        WHEN risk_score  = 1 THEN 'low'
        ELSE                      'safe'
    END AS risk_level
FROM risked
WHERE risk_score >= 2
ORDER BY risk_score DESC, amount_eur DESC
LIMIT 100;
```

**Extrait résultats** :

| transaction_id | risk_score | risk_level | amount_eur |
| :--- | :--- | :--- | :--- |
| 700191870 | 4 | critical | 1971.49 |
| 700500764 | 4 | critical | 1437.40 |
| 700500451 | 4 | critical | 1360.43 |

**Techniques** : CTE empilées (`scored` → `risked`), `AVG() OVER (PARTITION BY)`, scoring arithmétique multi-flags, CASE WHEN de classification.

---

### 4.6 : Top fraude par client (ROW_NUMBER + CTE)

**Question** : Pour les fraudes sous investigation, quelle est la plus grosse par client ?

```sql
WITH ranked AS (
    SELECT fr.*,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY amount_disputed DESC
           ) AS rn
    FROM fraud_reports fr
    WHERE status = 'under_investigation'
      AND fraud_type <> 'friendly_fraud'
)
SELECT fraud_id, customer_id, fraud_type, amount_disputed, reported_date
FROM ranked
WHERE rn = 1
ORDER BY amount_disputed DESC;
```

**Technique** : `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)`, pattern classique "top N par groupe".

---

## Synthèse : Les 5 insights actionables

1. **3DS = la mesure de mitigation la plus efficace.** ~92% des fraudes carte ont bypassé 3DS. Le rendre obligatoire au-dessus de 200€ éliminerait théoriquement 7 typologies sur 8.

2. **97% des pertes financières viennent des fraudes wire.** Les trois typologies sans carte (mule, account_takeover, sim_swap) génèrent l'essentiel du coût, alors qu'elles ne représentent que 32% des cas.

3. **La fraude carte est opportuniste, pas ciblée par tier.** La distribution des victimes suit la distribution naturelle des cartes émises. Mitigation comportementale > mitigation par segment.

4. **Trois régimes économiques distincts** demandent des défenses distinctes : spray-and-pray (velocity rules), big-game (auth renforcée sur virements), mid-range (3DS + scoring).

5. **Les biens digitaux sont la première cible de la fraude carte** : direct_mkt, streaming, tickets, gaming représentent 38% des cas carte. Surveillance accrue sur ces MCC justifiée.
