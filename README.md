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

## Хэсэг A: Category-Partition — `POST /transfer`

Тестлэгдэх функц: `POST /transfer {from, to, amount, currency}`. Код байхгүй, зөвхөн тодорхойлолт дээр дизайн хийнэ (shift-left).

### Олон алдааны давуу эрэмбэ

`ERROR_NO_FROM → ERROR_FROZEN → ERROR_NO_TO → ERROR_SAME_ACCOUNT → ERROR_BAD_AMOUNT → ERROR_INSUFFICIENT → ERROR_LIMIT`

Үндэслэл: эхлээд «хэнээс», дараа нь «хэнд», хамгийн сүүлд «хэдийг» шалгана, учир нь үлдэгдэл ба валютын хөрвүүлэлт нь from данс тогтоогдсоны дараа л тооцоологдох боломжтой.

### A1 — Сонголт ба төлөөлөх утгууд

| Параметр | Сонголт | Утга | Төлөөлөх утга (жишээ) | Хязгаарлалт |
|---|---|---|---|---|
| **from** | from данс байгаа, царцаагдаагүй эсэх | F1 хүчинтэй | `ACC1` (идэвхтэй, үлдэгдэл 1 000 000₮) | `[property fromExists]` |
| | | F2 царцаагдсан | `ACC_FROZEN` | `[ERROR]` `[property fromExists]` |
| | | F3 байхгүй | `ACC_NONE` | `[ERROR]` |
| **to** | to данс байгаа, from-оос өөр эсэх | T1 хүчинтэй | `ACC2` | |
| | | T2 байхгүй | `ACC_NONE2` | `[ERROR]` |
| | | T3 from-той ижил | `to = from` | `[ERROR]` `[IF fromExists]` |
| **amount** | дүн ба үлдэгдлийн харьцаа | A1 хэвийн (0 < дүн < үлдэгдэл) | `100 000` | |
| | | A2 яг үлдэгдэлтэй тэнцүү | `1 000 000` | `[SINGLE]` `[IF fromExists]` |
| | | A3 ≤ 0 | `0`, `-500` | `[ERROR]` |
| | | A4 үлдэгдлээс их | `1 000 001` | `[ERROR]` `[IF fromExists]` |
| **currency** | валют данснаас өөр эсэх | C1 данстай ижил (MNT) | `MNT` | |
| | | C2 өөр (хөрвүүлнэ) | `USD` | `[property converted]` |
| **Далд орчин: өдрийн лимит** | өнөөдөр лимитийн хэр хэсэг ашигласан бэ | L1 хоосон (дүн лимитэд багтана) | лимит 5 000 000₮, ашигласан 0 | |
| | | L2 энэ шилжүүлгээр яг лимитэд хүрнэ | ашигласан 4 900 000₮, дүн 100 000₮ | `[SINGLE]` `[IF fromExists]` |
| | | L3 энэ шилжүүлгээр лимит хэтэрнэ | ашигласан 4 950 000₮, дүн 100 000₮ | `[ERROR]` `[IF fromExists]` |

Далд орчны хоёр сонголт бий: from дансны **үлдэгдэл** (amount-тэй харьцуулж A1–A4 ангид шингээсэн) ба **өдрийн лимитийн ашиглалт** (тусдаа L ангилал).

### A2 — Хязгаарлалтууд ба үндэслэл

- **[ERROR] — F2, F3, T2, T3, A3, A4, L3:** утга бүр тодорхой нэг `ERROR_*` үүсгэдэг тул бусад утгатай хослуулах шаардлагагүй, нэг удаа шалгахад хангалттай. Хослуулбал давуу эрэмбээр нэг алдаа нөгөөгөө нууна (error masking).
- **[SINGLE] — A2, L2:** хязгаарын утга (`дүн = үлдэгдэл`, `ашигласан + дүн = лимит`) зөв боловч онцгой тул нэг удаа шалгахад хангалттай. Энд `>` ба `>=`-ийн андуурал илэрнэ.
- **[property fromExists] / [IF fromExists] — T3, A2, A4, L2, L3:** «from-той ижил», «үлдэгдэл», «өдрийн лимит» гэдэг ойлголтууд нь from данс байгаа үед л утгатай, from байхгүй бол эдгээр утгыг бүтээх боломжгүй.
- **[property converted] — C2:** валют өөр үед үлдэгдэл, лимитийг **хөрвүүлсний дараах** дүнгээр шалгадаг тул C2 нь A ба L-ийн утгатай хосолж сонирхолтой тохиолдол үүсгэнэ.

**Тооцоо (N → M):**

- Бүрэн олонлог: from 3 × to 3 × amount 4 × currency 2 × лимит 3 = **216**
- Хязгаарлалтын дараа:
  - [ERROR] утга бүр 1 удаа: F2, F3, T2, T3, A3, A4, L3 → **7**
  - [SINGLE] утга бүр 1 удаа: A2, L2 → **2**
  - Үлдсэн хэвийн утгуудын хослол: F1 × T1 × A1 × (C1, C2) × L1 → **2**
- **216 → 11**

### A3 — Спецификацийн хүснэгт

| # | from | to | amount | currency | өдрийн лимит | Хүлээгдэх үр дүн |
|---|---|---|---|---|---|---|
| T1 | F1 хүчинтэй | T1 хүчинтэй | A1 хэвийн | C1 ижил | L1 хоосон | **201** `OK` (happy path) |
| T2 | F1 | T1 | A1 | C2 өөр (USD → MNT) | L1 | **201** `OK` |
| T3 | F3 байхгүй | T1 | A1 | C1 | — | 200 `ERROR_NO_FROM` |
| T4 | F2 царцаагдсан | T1 | A1 | C1 | L1 | 200 `ERROR_FROZEN` |
| T5 | F1 | T2 байхгүй | A1 | C1 | L1 | 200 `ERROR_NO_TO` |
| T6 | F1 | T3 from-той ижил | A1 | C1 | L1 | 200 `ERROR_SAME_ACCOUNT` |
| T7 | F1 | T1 | A3 = 0 | C1 | L1 | 200 `ERROR_BAD_AMOUNT` |
| T8 | F1 | T1 | A4 үлдэгдлээс их | C1 | L1 | 200 `ERROR_INSUFFICIENT` |
| T9 | F1 | T1 | A1 | C1 | L3 хэтэрнэ | 200 `ERROR_LIMIT` |
| T10 | F1 | T1 | A2 яг үлдэгдэл | C1 | L1 | **201** `OK` (SINGLE) |
| T11 | F1 | T1 | A1 | C1 | L2 яг лимит | **201** `OK` (SINGLE) |
| T12 | F2 царцаагдсан | T2 байхгүй | A3 = 0 | C1 | L1 | 200 `ERROR_FROZEN` (давхар нөхцөл: эрэмбээр from түрүүлнэ) |
| T13 | F1 | T3 from-той ижил | A4 үлдэгдлээс их | C1 | L3 | 200 `ERROR_SAME_ACCOUNT` (давхар нөхцөл: to нь amount-аас түрүүлнэ) |
| T14 | F1 (үлдэгдэл 1 000 000₮) | T1 | 300 USD (тоогоор үлдэгдлээс бага, хөрвүүлбэл ≈1 050 000₮) | C2 өөр | L1 | 200 `ERROR_INSUFFICIENT` (property converted: хөрвүүлсний дараа хэтэрнэ) |

14 спецификаци: happy path (T1), алдааны анги бүр (T3–T9, 7 алдаа бүгд), SINGLE (T10, T11), давхар нөхцөл (T12, T13), хөрвүүлэлтийн сонирхолтой тохиолдол (T2, T14).

## Хэсэг B: PICT — pairwise covering array

### B1 — Лекцийн браузерын жишээ

`browser.model` (3 × 3 × 2 × 2 × 2 × 2 = **144** бүрэн олонлог):

```bash
pict browser.model | tee results/browser-2way.txt
pict browser.model /o:3 > results/browser-3way.txt
tail -n +2 results/browser-2way.txt | wc -l   # 10
tail -n +2 results/browser-3way.txt | wc -l   # 20
```

| Хувилбар | Мөр | Онолын доод хязгаар | Бүрэн олонлог |
|---|---|---|---|
| 2-way (`results/browser-2way.txt`) | **10** | 9 (Content × Cookies = 3 × 3) | 144 |
| 3-way (`results/browser-3way.txt`) | **20** | 18 (3 × 3 × 2) | 144 |

**Яагаад PICT 9 биш 10 өгсөн бэ:** PICT нь greedy алгоритм. Мөр бүрийг тухайн үед хамгийн олон хамраагүй хосыг хамарсан байхаар нэг нэгээр нь сонгодог, бүх хүснэгтийг нэг дор оновчлохгүй. Тиймээс хамгийн цөөн мөрийг заавал олохгүй. Лекцэд гараар 9 мөрөнд багтааж чадсан, харин PICT 1 мөр илүү гаргасан. Гэхдээ Content × Cookies-ийн 9 хос бүгд хамрагдаж, хос бүр дор хаяж нэг удаа гарсан нь хамгийн гол нь. 3-way нь 144-ийн **14%** буюу 20 мөр бөгөөд гурвалсан бүх хослолыг хамардаг.

### B2 — Шилжүүлгийн модель PICT-ээр

Хэсэг A-ийн сонголтуудыг PICT-ийн хэл рүү хөрвүүлсэн (`DailyLimit` бол далд орчны ангилал). Гурван хувилбар:

- `transfer-nc.model` — хязгаарлалтгүй
- `transfer.model` — A2-ын `[IF fromExists]` хязгаарлалттай:
  `IF [From] = "Missing" THEN [To] <> "SameAsFrom" AND [Amount] IN {"Normal", "Zero"} AND [DailyLimit] = "Empty";`
- `transfer-neg.model` — `[ERROR]` утгууд `~`-тэй (сөрөг), мөн IF хязгаарлалттай. PICT-ийн constraint дотор утгын нэрийг `~`-гүй бичдэг: `"Missing"`.

```bash
pict transfer-nc.model  > results/transfer-nc.txt
pict transfer.model     > results/transfer.txt
pict transfer-neg.model > results/transfer-neg.txt
```

| Хувилбар | Мөр | Алдаагүй мөр | Happy path мөр (Valid, Valid, Normal, Empty) | Нэг мөр дэх хамгийн их алдаа | Олон алдаатай мөр |
|---|---|---|---|---|---|
| `transfer-nc` | **13** | 0 | 0 | 4 | 10 |
| `transfer` | **15** | 1 (`Valid Valid ExactBalance Different Empty`) | 0 | 3 | 12 |
| `transfer-neg` | **19** | 5 | 2 | 1 | 0 |

(«Алдаа» гэдэгт Frozen, Missing, SameAsFrom, Zero, OverBalance, Exceeded утгуудыг тооцсон.)

**Ажиглалт:**
- Хязгаарлалтгүй ба хязгаарлалттай хувилбарт **happy path огт гараагүй**. Мөр бүрд дунджаар 2–3 алдаа зэрэг орсон, жишээ нь `Missing SameAsFrom OverBalance Different ExactLimit` мөрөнд 3 алдаа бий. Ийм мөр давуу эрэмбийн дагуу зөвхөн эхний алдааг (`ERROR_NO_FROM`) шалгах бөгөөд бусад нь нуугдана (error masking). Тиймээс олон алдаатай мөр юуг шалгаж байгаа нь тодорхойгүй.
- `transfer-nc`-д from байхгүй атлаа «ижил данс», «яг үлдэгдэл», «лимит» гэсэн утгагүй хослолтой 4 мөр гарсан. `transfer.model`-ийн хязгаарлалт тэдгээрийг арилгасан тул мөрийн тоо 13 → 15 болж өссөн. Хязгаарлалт нэмэхэд PICT үлдсэн хосуудыг хамрахын тулд өөр мөр нэмэх хэрэгтэй болдог.
- `~` хэрэглэхэд мөр бүр **хамгийн ихдээ 1 алдаатай** болж, 5 алдаагүй мөр гарсан. Мөрийн тоо 19 болж өссөн, учир нь сөрөг утга бүр хүчинтэй утга бүртэй тусдаа мөрөнд хосолдог.
- **`~` хэрэглэхэд A2-ын хязгаарлалт хэрэгтэй хэвээр үү?** Хэсэгчлэн хэрэгтэй. `~` нь алдаа × алдааг (жишээ нь `Missing × SameAsFrom`, `Missing × Exceeded`) өөрөө хаадаг тул хязгаарлалтын тэр хэсэг илүүц болно. Гэхдээ `~Missing × ExactBalance` эсвэл `~Missing × ExactLimit` хоёулаа «хүчинтэй» утгатай хослол тул `~` тэднийг хаахгүй. from байхгүй бол үлдэгдэл, лимит гэж байхгүй, тиймээс IF хязгаарлалт хэрэгтэй хэвээр.

**PICT мөрөөс бодит тест кейс (Лаб 5-ын хэв маяг):**

Нийтлэг setup: `ACC1` (MNT, үлдэгдэл 1 000 000₮, өдрийн лимит 5 000 000₮), `ACC2` (MNT). Ханш 1 USD = 3 500₮.

| # | Эх мөр | Setup | Оролт (`POST /transfer`) | Oracle |
|---|---|---|---|---|
| P1 happy path | `transfer-neg` 3-р мөр: `Valid Valid Normal Same Empty` | ACC1 идэвхтэй, өнөөдөр 0₮ шилжүүлсэн; ACC2 байгаа | `{"from":"ACC1","to":"ACC2","amount":100000,"currency":"MNT"}` | **201**, `result = "OK"`; ACC1 үлдэгдэл 900 000₮ болсон |
| P2 нэг алдаа | `transfer-neg` 9-р мөр: `Valid Valid ~Zero Different Empty` | ACC1 идэвхтэй, өнөөдөр 0₮; ACC2 байгаа | `{"from":"ACC1","to":"ACC2","amount":0,"currency":"USD"}` | **200**, `result = "ERROR_BAD_AMOUNT"`; үлдэгдэл өөрчлөгдөөгүй |
| P3 нэг алдаа | `transfer-neg` 15-р мөр: `~Frozen Valid Normal Different ExactLimit` | `ACC_FROZEN` царцаагдсан, өнөөдөр 4 650 000₮ шилжүүлсэн (100 USD = 350 000₮-өөр яг лимитэд хүрнэ); ACC2 байгаа | `{"from":"ACC_FROZEN","to":"ACC2","amount":100,"currency":"USD"}` | **200**, `result = "ERROR_FROZEN"` |
| P4 олон алдаа | `transfer` 6-р мөр: `Frozen SameAsFrom Zero Different Empty` | `ACC_FROZEN` царцаагдсан, өнөөдөр 0₮ | `{"from":"ACC_FROZEN","to":"ACC_FROZEN","amount":0,"currency":"USD"}` | **200**, `result = "ERROR_FROZEN"`. Давуу эрэмбээр from-ын шалгалт to (SAME_ACCOUNT) ба amount (BAD_AMOUNT)-аас түрүүлнэ. Нөгөө 2 алдаа энэ тестээр шалгагдахгүй. |
