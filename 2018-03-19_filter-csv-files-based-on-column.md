---
categories: notes
---

# How to filter big csv files based on values occuring in specific columns

**tl;dr**
1. create a txt-file with the patterns in it, one per line
2. grep for the patterns in this file via `grep -fF ids.txt bigcsv.csv`

Recently, [a colleague of mine](https://twitter.com/notknut) found nice data
that was very useful for an [investigation we are working
on](https://correctiv.org/recherchen/wohnen/wem-gehoert-hamburg/).

It was an update for [German Zensus 2011 Datasets](https://www.zensus2011.de/SharedDocs/Aktuelles/Ergebnisse/DemografischeGrunddaten.html?nn=3065474).

For the investigation I was interested in the tables "Wohnungen und Gebäude im
100 Meter-Gitter". These are huge `csv`-files about ownerships of flats (and
more info) in Germany – each line represents a specific *100 meter grid square*
within germany (no, this little blog post is NOT about turning this into useful
geodata).

**Wohnungen100m.csv**

```csv
Gitter_ID_100m,Gitter_ID_100m_neu,Merkmal,Auspraegung_Code,Auspraegung_Text,Anzahl,Anzahl_q
100mN26865E43357,CRS3035RES100mN2686500E4335700,INSGESAMT,0,Einheiten insgesamt,3,0
100mN26865E43357,CRS3035RES100mN2686500E4335700,HEIZTYP,4,Zentralheizung,3,0
100mN26865E43357,CRS3035RES100mN2686500E4335700,WOHNEIGENTUM,99,Trifft nicht zu (da keine Eigentumswohnung),3,0
100mN26865E43357,CRS3035RES100mN2686500E4335700,ZAHLWOHNGN_HHG,1,1 Wohnung,3,0
```

The file is `4.3G` for total Germany, but I need only the data for the city of
Hamburg.

As you may have already recognized, apart from the `Gitter_ID_100m`-column
there is no further information about the geographical location for a data row.

Luckily, there is another `3.2G` big `csv`-file that maps Germany into these
*100 meter grid squares*, found at the
[Geodatenzentrum](https://www.geodatenzentrum.de/geodaten/gdz_rahmen.gdz_div?gdz_spr=deu&gdz_akt_zeile=5&gdz_anz_zeile=1&gdz_unt_zeile=31&gdz_user_id=0)
from the "Bundesamt für Katographie und Geodäsie" of Germany.

**DE_Grid_ETRS89-LAEA_100m.csv**

```csv
100mN26840E43341;4334100;2684000;4334150;2684050;227;227;0;2,27;2,27;0,0;09780133
100mN26840E43342;4334200;2684000;4334250;2684050;1402;1402;0;14,02;14,02;0,0;09780133
100mN26840E43343;4334300;2684000;4334350;2684050;1844;1844;0;18,44;18,44;0,0;09780133
100mN26840E43344;4334400;2684000;4334450;2684050;2287;2287;0;22,87;22,87;0,0;09780133
100mN26840E43345;4334500;2684000;4334550;2684050;488;488;0;4,88;4,88;0,0;09780133
```

As you see, we have the *grid id* in the first column, and – according to the
[Datensatzbeschreibung](https://www.geodatenzentrum.de/auftrag1/archiv/vektor/geogitter/last/geogitter.pdf)
– the *AGS* (a unique ID for each city in Germany) in the last column.

Usually I would now fire up a [Jupyter Notebook](https://jupyter.org/) and
solve the matching problem with the python-based data wrangling library
[pandas](https://pandas.pydata.org/)

But, I am also a huge fan of the unix *command-line*, so...

## 1. Obtain the grid squares that belong to Hamburg

Hamburg has the *AGS* `02000000`, and as long as this information is in the
last column of the second mentioned `csv`-file, we can safely filter it via the
[grep](https://www.gnu.org/software/grep/)-pattern `02000000$` (for those who
are not familiar with regex: the `$` indicates the end of line)

So, to filter the grid file for only the grids belonging to Hamburg, we could do this:

    grep 02000000$ DE_Grid_ETRS89-LAEA_100m.csv

This gives us back all the records that end with the *AGS* from the city of Hamburg.

But I only need a list of the *grid ids*, that is only the first column of this file.

That is where the command-line tool
[cut](https://en.wikipedia.org/wiki/Cut_(Unix)) comes in – we simply specify a
delimiter (`;` in this case) and the 1-based index of the columns we want (just
`1` in our case).

To get just every *grid id* that belongs to Hamburg, one per line:

    grep 02000000$ DE_Grid_ETRS89-LAEA_100m.csv | cut -d";" -f1

The output of this command looks then like this:

```
100mN33653E43354
100mN33653E43355
100mN33653E43356
100mN33653E43357
100mN33653E43358
100mN33653E43359
100mN33653E43360
100mN33653E43361
100mN33653E43362
100mN33653E43363
```

We need this as a file (as you will see in step 2), so we pipe the output to a
new file like this:

    grep 02000000$ DE_Grid_ETRS89-LAEA_100m.csv | cut -d";" -f1 > ids.txt


## Grep a csv file by a list of patterns

We now take our initial `csv`-File, and filter it only for the *grid ids* that
belong to Hamburg:

    grep -Ff ids.txt Wohnungen100m.csv > Wohnungen100m_HH.csv

According to the grep documentation, `-f` indicates to obtain the patterns to
grep for from a specified file, one per line. `-F` indicates to interpret the
patterns as `fixed strings` instead of regex.

To make this more stable, we could force `grep` to only search against the
beginning of the rows (currently it looks for the occurences of the *grid ids*
anywhere), we could modify the creation of `ids.txt` (step 1) a bit:

    grep 02000000$ DE_Grid_ETRS89-LAEA_100m.csv | cut -d";" -f1 | sed s/^/\^/g > ids.txt

(this `sed` pattern just adds a `^`-sign to the beginning of every line)

Then we would just drop the `-F` option. Because `grep` is now evaluating
regular expressions, the actual command from above takes a lot (really a lot!)
more time.

## Keep the csv column headers

If you followed this step by step on your own, you may have noticed that at the
last step we lost the column headers from the `Wohnungen100m.csv` file.

To prevent this, we could first obtain the headers from the origin source file
via the `head` command:

    head -1 Wohnungen100m.csv > Wohnungen100m_HH.csv

And then append the output from step 2 via the `>>` operator (not `>`) to the
already existing `Wohnungen100m_HH.csv`.
