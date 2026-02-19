# Tesi triennale UniTo informatica - Digitalizzazione ECGs su carta

Repository contenente i sorgenti LaTeX della tesi triennale di Matteo Sardi.

---

## Struttura del progetto

```
.
├── main.tex
├── Bibliography.bib
├── README.md
├── contents/
│   ├── TitlePage.tex
│   ├── Abstract.tex
│   ├── CoolQuote.tex
│   ├── SampleChapter.tex
│   ├── ResponsabilityDeclaration.tex
│   └── Thanks.tex
├── images/
│   ├── unito_logo.pdf
└── .gitignore
```

---

## Compilazione

La tesi è stata realizzata con LaTeX e utilizza i seguenti package:

- minted per l’evidenziazione del codice
- babel e babelbib per la gestione delle lingue e della bibliografia
- graphicx, caption, subcaption per immagini e figure
- fancyhdr per gli header personalizzati


### Procedura

1. Pulire eventuali file temporanei:

*.aux *.bbl *.blg *.toc *.fls *.fdb_latexmk *.synctex.gz _minted/

2. Compilare con latexmk abilitando -shell-escape:

latexmk -pdf -shell-escape main.tex

3. Il PDF finale sarà generato come main.pdf.

---

## Submodule

Questo progetto include una versione modificata del seguente submodule:
  [ECG-Image-Kit](https://github.com/alphanumericslab/ecg-image-kit), distribuito sotto la licenza *BSD 3-Clause License*.

  Nel submodule è presente il codice relativo al lavoro svolto.

---

## Licenza del progetto

Questo repository è rilasciato sotto [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/).  
I submodule esterni mantengono la loro licenza originale.

---

> Author: [Matteo Sardi](https://github.com/matte2712s)