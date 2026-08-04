# Point d'étape binôme — avant le développement

**Objet.** Valider les acquis du travail préliminaire (§1 et §2 du brief) avant
d'attaquer le développement (§3, §4, §5).

**Durée** : 45 min. **Format** : écran partagé, dépôt lancé.

---

## Avant la réunion — 10 min chacun

```bash
git fetch && git checkout diagnostic-et-banc-essai
make install && cp .env.example .env    # renseigner AZURE_AI_INFERENCE_*
make up && make seed
make ui                                 # http://localhost:8501
```

Lire [note-diagnostic.md](note-diagnostic.md) — 170 lignes, 10 min.

> ⚠️ `uv.lock` n'est pas versionné : vos environnements peuvent différer. Si un
> comportement diverge, comparer `uv pip list` avant de chercher ailleurs.

---

## Déroulé

### 1. Démonstration — 10 min

Sur `make ui`, dans l'ordre :

| Onglet | À montrer | Ce que ça prouve |
|---|---|---|
| **Recette** | lancer en mode LLM | 4/4 exactes — le Text-to-SQL fonctionne |
| **Bac à sable** | saisir « Supprime tous les clients » | le modèle génère `DELETE`, le garde-fou l'accepte |
| **Diagnostic** | lire les verdicts | 3 requêtes dangereuses passent, 2 entrées → 2 sorties, 0/4 questions chargées |

Rien n'est recopié : la page exécute le code du dépôt au chargement.

### 2. Validation du §1 — 10 min

Critères du brief : *« Les requêtes problématiques et leurs causes sont
identifiées. La note et le schéma sont produits. »*

- [ ] Le **relevé des 8 requêtes adverses** est complet et compréhensible (§3 de la note)
- [ ] Les **5 causes** G1, G2, C1, C2, V1 sont justes et bien localisées (§4)
- [ ] Le **schéma du flux** se lit sans explication orale (§1)
- [ ] Le constat **4/4 exactes** est accepté : le symptôme du brief n'est pas reproductible par le Text-to-SQL
- [ ] Le constat sur le **rollback implicite** est compris : DML annulé, DDL exécuté

**Question à trancher ensemble** : est-ce que 8 questions adverses suffisent, ou
faut-il en ajouter (injection via valeurs, `UNION`, requêtes coûteuses) ?

### 3. Le §2 n'est pas fait — 15 min

C'est le point central de la réunion. Le brief demande, **avant** le
développement :

> Définir les garde-fous SQL · Localiser l'origine des doublons et la stratégie
> de correction · Définir la vérification d'exactitude sur le jeu de test.
> *Critère : les garde-fous et la stratégie sont explicites et justifiés.*

La note de diagnostic couvre le **quoi** et le **pourquoi**, pas encore le
**comment**. À décider maintenant, à deux :

| Sujet | Décision attendue |
|---|---|
| **Garde-fous** | Parseur SQL réel (`sqlglot`) ou regex durcie ? Où s'applique le contrôle — dans `run_query()` ? |
| **Rôle lecture seule** | On ajoute un `GRANT SELECT` PostgreSQL ? Ça neutralise 5 des 8 requêtes adverses à lui seul. |
| **Dédoublonnage** | Clé = `strip().lower()` sur `external_id`. Fraîcheur = date parsée, pas comparaison de chaînes. Confirmé ? |
| **Pagination / 429** | Retry `tenacity` dans `sources/base.py` (partagé) ou dans chaque connecteur ? |
| **Vérification** | `load_test_questions()` réparé + les 4 questions passent en CI. Suffisant, ou on ajoute les cas adverses en test ? |

**Livrable de cette séance** : une section « Conception » ajoutée à la note, ou
un `docs/note-conception.md`. Sans elle, le §2 reste non validé.

### 4. Répartition et décisions — 10 min

Ordre imposé par le diagnostic — **V1 d'abord**, sans instrument de mesure rien
n'est démontrable :

| Lot | Contenu | Qui |
|---|---|---|
| **V1** | `load_test_questions()` lit le JSON | |
| **G1 + G2** | garde-fou qui lève + appelé dans `run_query()` | |
| **C1** | pagination, retry 429, normalisation `source_two` | |
| **C2** | clé normalisée, fraîcheur, fusion | |

G1 et G2 vont ensemble : corriger l'un sans l'autre ne change **rien** au
comportement. C1 avant C2 : consolider des données incomplètes n'a pas de sens.

---

## Points ouverts à acter

- **Modèle** : le dépôt documente Kimi-K2.6, le déploiement réel est
  `gpt-5.4-mini` sur une route Azure OpenAI. Aligner la doc, ou repointer ?
- **`brief.md`** est vide (0 octet) ; le brief réel est
  `brief-agent-text-to-sql.md`. Supprimer le fichier vide ?
- **`uv.lock` non versionné** — à corriger pour que vos deux environnements
  soient identiques.
- **Clé de jointure absente** entre `clients.id` (PostgreSQL) et les
  `external_id` des sources : les deux branches ne peuvent pas être rapprochées.
  Hors brief, mais à signaler en soutenance.

---

## État de référence

| Élément | Valeur |
|---|---|
| Branche | `diagnostic-et-banc-essai` |
| Dernier commit poussé | `3121278` |
| Tests | **12 échecs / 4 succès / 1 ignoré** |
| Chantiers du brief traités | **aucun** — c'est normal à ce stade |

Les 12 échecs sont la feuille de route : chacun correspond à un défaut relevé.
Ils doivent passer au vert au fil des lots ci-dessus.
