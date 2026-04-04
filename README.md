# stgraph-engineer — Skill per Claude Code

Skill specializzata per Claude Code che trasforma il modello in un ingegnere esperto di **STGraph** e del linguaggio di espressione **STEL**.

## Cosa fa

Quando attivata, la skill abilita Claude Code a:

- **Analizzare modelli `.stg`** — inventario nodi, grafo delle dipendenze, header di simulazione
- **Fare debug di espressioni STEL** — parsing, precedenza operatori, variabili, contesto
- **Correggere modelli** — fix precisi con sintassi verificata sul codice sorgente STGraph
- **Spiegare concetti** — come funziona il motore di simulazione, i tipi di nodo, il ciclo di calcolo
- **Confrontare e validare espressioni** — equivalenza semantica, polimorfismo Tensor

Opera con **disciplina ingegneristica stretta**: non inventa sintassi, usa sempre il codice sorgente come fonte di verità.

## File inclusi

| File | Descrizione |
|------|-------------|
| `SKILL.md` | La skill da importare in Claude Code |
| `COME_IMPORTARE_LA_SKILL.txt` | Guida passo per passo all'importazione |

## Installazione rapida

```bash
mkdir -p ~/.claude/skills/stgraph-engineer
curl -o ~/.claude/skills/stgraph-engineer/SKILL.md \
  https://raw.githubusercontent.com/TUO-USERNAME/TUA-REPO/main/SKILL.md
```

> Sostituisci `TUO-USERNAME/TUA-REPO` con il percorso della tua repo GitHub (o scarica `SKILL.md` manualmente e copialo nella cartella).

Riavvia Claude Code. La skill sarà disponibile come `/stgraph-engineer`.

## Utilizzo

```
/stgraph-engineer analizza il modello main.stg
/stgraph-engineer debug dell'espressione: integral(x) nel nodo O2
/stgraph-engineer correggi l'errore di divisione per zero in food_factor_raw
```

## Note

La skill fa riferimento a file di regole interni al progetto STGraph:
- `rules/STGRAPH_MASTER_RULES.md`
- `rules/STGRAPH_REVIEW_CHECKLIST.md`
- `rules/COMMON_STGRAPH_ERRORS.md`
- `rules/STGRAPH_SOURCE_CODE_MAP.md`
- `CLAUDE.md`

Per il funzionamento completo, Claude Code deve essere aperto nella cartella del progetto STGraph dove questi file sono presenti. Senza di essi la skill funziona comunque, ma usa solo la conoscenza interna.

---

Creata con [Claude Code](https://claude.ai/code)
