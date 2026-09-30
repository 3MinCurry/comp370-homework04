# COMP 370 Homework 4 - Exploring the My Little Pony dataset

All commands were run on an Ubuntu 24.04 EC2 instance from inside the repo, with
`clean_dialog.csv` in the working directory. `csvtool` was installed with
`sudo apt install csvtool`.

## Task 3: Dataset properties

### How big is the dataset?

```bash
ls -lh clean_dialog.csv          # 4.7M
wc -l clean_dialog.csv           # 36860 lines
csvtool height clean_dialog.csv  # 36860 rows
csvtool width clean_dialog.csv   # 4 columns
tail -n +2 clean_dialog.csv | wc -l   # 36859 (header removed)
```

The file is **4.7 MB** with **36,860 lines**: one header row plus **36,859 lines
of dialogue**, across **4 columns**. `wc -l` and `csvtool height` agree, so no
field contains an embedded newline and every line of the file is exactly one
record.

### What's the structure of the data?

```bash
head -5 clean_dialog.csv
csvtool head 1 clean_dialog.csv
csvtool col 1 clean_dialog.csv | tail -n +2 | sort -u | wc -l   # 197
csvtool col 2 clean_dialog.csv | tail -n +2 | sort -u | wc -l   # 66
csvtool col 3 clean_dialog.csv | tail -n +2 | sort -u | wc -l   # 842
csvtool col 3 clean_dialog.csv | tail -n +2 | sort | uniq -c | sort -rn | head -20
```

The header is `"title","writer","pony","dialog"`, and every field is wrapped in
double quotes. Each row is one line of dialogue.

| Field | Meaning | Values |
|-------|---------|--------|
| `title` | Episode title | 197 distinct, e.g. `Friendship is Magic, part 1`, `Stare Master` |
| `writer` | Credited writer(s) of that episode | 66 distinct free-text credits, e.g. `Lauren Faust`, `Dave Rapp; story by Meghan McCarthy` |
| `pony` | Character speaking the line | 842 distinct, e.g. `Twilight Sparkle`, `Spike`, `Others`, `Flim and Flam` |
| `dialog` | The spoken line itself | Free text, ranging from `Huh?` to full paragraphs of narration |

The most frequent speakers are Twilight Sparkle (4,745 lines), Rainbow Dash
(3,072), Pinkie Pie (2,833), Applejack (2,748), Rarity (2,660), Spike (2,268)
and Fluttershy (2,109).

### How many episodes does it cover?

```bash
csvtool col 1 clean_dialog.csv | tail -n +2 | sort -u | wc -l   # 197
csvtool col 1 clean_dialog.csv | tail -n +2 | uniq | wc -l      # 197
csvtool col 1 clean_dialog.csv | tail -n +2 | sort | uniq -c | sort -rn | head -5
csvtool col 1 clean_dialog.csv | tail -n +2 | sort -u | grep -vc -E 'The Movie|Best Gift Ever'   # 195
```

There are **197 distinct titles**. Two of them are not TV episodes:
`My Little Pony The Movie` (748 lines) and the special `My Little Pony Best Gift
Ever` (458 lines). That leaves **195 TV episodes**. Counting unique titles in file
order with `uniq` also gives 197, which means each episode's lines are stored
together in one contiguous block. Two-part episodes are counted as two episodes
because each part has its own title.

### Unexpected aspects of the dataset

**1. The `pony` (speaker) field is not a clean list of characters.** This is the
issue most likely to cause problems in later analysis.

```bash
csvtool col 3 clean_dialog.csv | tail -n +2 | grep -c ' and '             # 294 rows
csvtool col 3 clean_dialog.csv | tail -n +2 | grep ' and ' | sort -u | wc -l   # 123 distinct values
csvtool col 3 clean_dialog.csv | tail -n +2 | grep -iE 'twilight|rarity|pinkie|rainbow|fluttershy' | sort | uniq -c | sort -rn
csvtool col 3 clean_dialog.csv | tail -n +2 | grep -xE 'Others|All|Main cast|Crowd|Ponies' | sort | uniq -c
```

- **Lines spoken by several characters together** are stored as a single speaker,
  e.g. `Fluttershy and Rainbow Dash` (11 lines), `Pinkie Pie and Rarity` (8) and
  `Twilight Sparkle and Spike` (7). There are 294 such rows across 123 different
  combinations. Their lines are not credited to any individual character.
- **One character appears under many names**, e.g. `Young Rainbow Dash`,
  `Mean Twilight Sparkle`, `Future Twilight Sparkle`, `Pinkie Pie 2`,
  `Pinkie Pie Duplicate`, `Illusion Rarity`, and `Twilight` as well as
  `Twilight Sparkle`.
- **Group labels** stand in for a speaker: `Others` (979 lines), `Ponies` (43),
  `All` (36), `Crowd` (20), `Main cast` (10), and even `All sans Twilight Sparkle`.

As a result, any per-character count depends on how these cases are handled.

**2. Character names also appear inside the dialogue.** A plain `grep` for a name
counts every line that *mentions* the character, not just the lines they *speak*:

```bash
grep -c 'Rarity' clean_dialog.csv            # 3517 (any mention)
grep -c '","Rarity","' clean_dialog.csv      # 2660 (Rarity is the speaker)
```

**3. The file uses Windows line endings (`\r\n`).**

```bash
head -2 clean_dialog.csv | cat -A    # every line ends in ^M$
file clean_dialog.csv
```

A pattern anchored to the end of the line, such as `grep 'Rarity"$'`, silently
matches nothing because of the hidden `\r`.

**4. Missing dialogue is stored as an unquoted `NA`.**

```bash
csvtool col 4 clean_dialog.csv | grep -cx 'NA'    # 21
grep -n ',NA' clean_dialog.csv | head -3
```

21 rows have `NA` as their dialogue, e.g. line 12081:
`"Magical Mystery Cure","M. A. Larson","Others",NA`. These are the only
unquoted fields in the file, and they would be counted as real lines of speech.

**5. Titles and writers are formatted inconsistently.**

```bash
csvtool col 1 clean_dialog.csv | tail -n +2 | sort -u | grep -i 'part'
csvtool col 2 clean_dialog.csv | tail -n +2 | sort | uniq -c | sort -rn
```

- Two-part titles use three different styles: `Friendship is Magic, part 1`,
  `The Return of Harmony Part 1` and `A Canterlot Wedding - Part 1`.
- The `writer` field mixes names with `story by` credits, uses both `&` and
  `and`, and one value still contains a leftover Wikipedia footnote:
  `...story by Meghan McCarthy & Joe Ballarini[25]`.
- The movie and a holiday special are mixed in with the TV episodes.

## Task 4: Speaker frequency

### How often does each main pony speak?

Each line was counted with `grep`, anchored to the speaker column. In the raw
file the speaker field always sits between the `writer` and `dialog` fields, so
the pattern `","<Name>","` matches only rows where that name is the entire
speaker field. It skips mentions inside the dialogue, as well as combined
speakers like `Fluttershy and Rainbow Dash`.

```bash
grep -c '","Twilight Sparkle","' clean_dialog.csv   # 4745
grep -c '","Rarity","' clean_dialog.csv             # 2660
grep -c '","Pinkie Pie","' clean_dialog.csv         # 2833
grep -c '","Rainbow Dash","' clean_dialog.csv       # 3072
grep -c '","Fluttershy","' clean_dialog.csv         # 2109
```

To check these counts, each was compared with an exact match on the speaker
column (`csvtool col 3 clean_dialog.csv | grep -cx 'Rarity'`), and all five
numbers were identical.

Only lines where the pony speaks alone, under her standard name, were counted.
Combined speakers (`Pinkie Pie and Rarity`) and variants (`Young Rarity`,
`Mean Rarity`) were left out, since they are either not her alone or not the
same character.

### Percent of all lines

The denominator is every line of dialogue in the dataset, from all characters,
with the header excluded: **36,859**.

```bash
total=$(tail -n +2 clean_dialog.csv | wc -l)
echo "pony_name,total_line_count,percent_all_lines" > Line_percentages.csv
for p in "Twilight Sparkle" "Rarity" "Pinkie Pie" "Rainbow Dash" "Fluttershy"; do
  n=$(grep -c "\",\"$p\",\"" clean_dialog.csv)
  awk -v p="$p" -v n="$n" -v t="$total" 'BEGIN{printf "%s,%d,%.2f\n", p, n, 100*n/t}' >> Line_percentages.csv
done
cat Line_percentages.csv
```

| pony_name | total_line_count | percent_all_lines |
|-----------|------------------|-------------------|
| Twilight Sparkle | 4745 | 12.87 |
| Rarity | 2660 | 7.22 |
| Pinkie Pie | 2833 | 7.69 |
| Rainbow Dash | 3072 | 8.33 |
| Fluttershy | 2109 | 5.72 |

Together the five main ponies speak 15,419 of the 36,859 lines, or **41.83%** of
all dialogue. Twilight Sparkle, the show's lead, has the most lines by a clear
margin.
