# Tech Notes

Raccolta di note tecniche scritte in Markdown, versionate in Git e convertite
automaticamente nei formati di consegna (PDF, DOCX, HTML, sito web, OneNote).

## Struttura

```
tech-notes/
├── docs/                  # sorgenti Markdown (una nota = un file)
│   ├── index.md           # home del sito
│   └── esempio-nota.md    # nota di esempio con tutti gli elementi tipici
├── docs/assets/           # immagini, schemi, PDF di riferimento
├── pandoc/
│   ├── defaults-pdf.yaml  # impostazioni comuni per PDF
│   ├── defaults-docx.yaml # impostazioni comuni per Word
│   ├── defaults-html.yaml # HTML autonomo
│   ├── defaults-onenote.yaml # HTML semplice per OneNote
│   └── *.css              # stili per HTML e OneNote
├── build/                 # output generati (non versionati)
├── mkdocs.yml             # configurazione del sito
├── Makefile               # comandi di conversione
└── .github/workflows/     # build automatica e pubblicazione su GitHub Pages
```

## Prerequisiti

- [Pandoc](https://pandoc.org/installing.html) ≥ 3.0
- Per il PDF: una distribuzione LaTeX (TeX Live / MiKTeX) **oppure** `weasyprint`
- Per il sito: `pip install mkdocs-material`
- Per i diagrammi Mermaid in PDF/DOCX: `npm i -g @mermaid-js/mermaid-cli` + filtro
  `pandoc-mermaid` (facoltativo; nel sito MkDocs funzionano già)

Su Windows funziona tutto anche da WSL2.

## Comandi

| Comando                          | Risultato                                         |
|----------------------------------|---------------------------------------------------|
| `make pdf  NOTE=esempio-nota`    | `build/esempio-nota.pdf`                          |
| `make docx NOTE=esempio-nota`    | `build/esempio-nota.docx` (per colleghi su Word)  |
| `make html NOTE=esempio-nota`    | `build/esempio-nota.html` autonomo, immagini incluse |
| `make onenote NOTE=esempio-nota` | come `html`, ottimizzato per copia-incolla in OneNote |
| `make all`                       | tutte le note in tutti i formati                  |
| `make site`                      | sito statico in `build/site/`                     |
| `make serve`                     | anteprima live del sito su http://127.0.0.1:8000  |

## Condividere con OneNote

OneNote non importa Markdown direttamente. Il metodo che conserva meglio
tabelle, titoli e blocchi di codice è:

1. `make onenote NOTE=nome-nota`
2. apri `build/nome-nota.onenote.html` nel browser
3. `Ctrl+A`, `Ctrl+C` → incolla in una pagina OneNote

Alternative:
- **Word come ponte**: `make docx`, poi copia-incolla dal DOCX, oppure
  *Inserisci → Stampa file* per un'immagine fedele ma non modificabile.
- **Automatico**: l'API Microsoft Graph crea pagine OneNote a partire da HTML;
  si può aggiungere uno script che pubblica l'HTML generato in un blocco appunti.

Il sorgente "vero" resta sempre il `.md` nel repository: OneNote è una copia
di consultazione, non il posto dove modificare.

## Convenzioni

- Nomi file in minuscolo con trattini: `imx95-jailhouse-setup.md`
- Ogni nota inizia con un blocco YAML (titolo, autore, data, tag)
- Immagini in `docs/assets/`, riferite con percorso relativo
- Diagrammi come testo (Mermaid), non come immagini esportate
