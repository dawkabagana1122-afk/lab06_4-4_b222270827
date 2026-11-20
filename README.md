# Лаб 6: Тест тохиолдол боловсруулах — Category-Partition ба PICT

**F.CSA313 — Программ хангамжийн чанарын баталгаа ба тест (2026)**

| | |
|---|---|
| **Оюутан** | Д.Алтанбагана |
| **Оюутны код** | B222270827 |

## Алхам 1: PICT-ийг эх кодоос бүтээх

macOS дээр cmake суулгаагүй тул зааврын `make` хувилбарыг ашигласан. Эх кодыг репоос гадна `~/tools/pict` руу clone хийсэн:

```bash
mkdir -p ~/tools && cd ~/tools && git clone https://github.com/microsoft/pict && cd pict
make -j4 && sudo cp pict /usr/local/bin/
```

Нотолгоо `results/pict-build.txt`:

```
pict commit ab76c25 2026-09-08
Pairwise Independent Combinatorial Testing

Usage: pict model [options]
```
