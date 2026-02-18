# miis2025-spm
Course on Structured Probabilistic Models as part of MIIS

## Pyenv

`pyenv activate miis2025-spm`


## Pandoc

```sh
pandoc hw/l1/hw1.md -o hw/l1/rcs-hw1-ml2025.pdf \
  --pdf-engine=xelatex \
  -V geometry:landscape \
  -V mainfont="TeX Gyre Pagella" \
  -V mathfont="TeX Gyre Pagella Math" \
  --embed-resources --resource-path=hw/l1
```