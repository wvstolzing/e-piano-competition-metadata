# e-piano-competition-metadata
Re-scraped metadata for MIDI files from the now defunct Yamaha e-piano competition

The sqlite3 database, and the json representation extracted from it, are meant to accompany the repo at https://github.com/chelate1118/piano-e-competition-midi-files which contains both a dump of the 'standard' MIDI files formerly hosted at yamahaden.com, as well as processed versions containing (presumably) additional CC data extracted from 'XP' midi files from the same site.

The aforementioned repo unfortunately doesn't contain the full metadata for composers, work titles, and performers. The other dump of the same collection available on the internet, the 'MAESTRO' dataset meant to feed some sort of ML training software, makes mention of the composers, but omits full player & work metadata.

I shall describe the steps I followed to construct `yamaha_competition.json` and `yamaha_competition.db`; so that if I've done anything stupid it should be easy to figure out where/why:

To gather the relevant metadata I scraped the relevant pages from the archive.org snapshots of the now offline competition website, using the following wget command:

```
wget --recursive --no-clobber --page-requisites --html-extension --convert-links  https://web.archive.org/web/20201001220519/https://yamahaden.com/midi-files
```

Fortunately the HTML was easy to parse using lxml, so I used the following python script to do that:

```
from lxml import html
from pathlib import Path
from xxhash import xxh3_64_hexdigest
import orjson

basepath = (Path.home() / Path("src/yamahacompetition_dnm"))
relevfiles = (basepath / "relevfiles").read_text().splitlines()

worklist = []
perflist = []
perf2worklist = []

for f in relevfiles:
    parsed = html.parse((basepath / f))
    details = parsed.xpath("//*[starts-with(@class, 'details')]")
    piecefields = details[0].xpath("//*[@class='piece_fields']")
    workinfo = piecefields[0].xpath("//*[contains(@class, 'item_subcont')]")
    perfblocks = details[0].xpath("//*[@class='perform_block']")

    workstr = ''
    composerstr = ''
    datestr = ''
    arrstr = ''

    for item in workinfo:
        textbit = item.text_content().strip()
        if textbit.lower() == 'by':
            continue
        else:
            workstr = f"{workstr}{textbit}"

    composerstr = piecefields[0].xpath("//*[@class='composer_name']")[0].text_content().strip()

    if len(piecefields) > 1:
        for moreitems in piecefields[1:]:
            moreitemtext = moreitems.text_content().strip()
            if moreitemtext.startswith('Date of Composition: '):
                datestr = moreitemtext[len('Date of Composition: '):]
            elif moreitemtext.startswith('Transcribed/Arranged By: '):
                arrstr = moreitemtext[len('Transcribed/Arranged By: '):]

    workdict = {
            "workid": xxh3_64_hexdigest(f"{workstr}{composerstr}{arrstr}{datestr}".encode()),
            "work": workstr,
            "composer": composerstr,
            "arr": arrstr,
            "date": datestr
            }

    worklist.append(workdict)

    for perfblock in perfblocks:

        perfdict = {
            "collection": "",
            "duration": "",
            "eseq_file": "",
            "performed_by": "",
            "piece_type": "",
            "recording_date": "",
            "round": "",
            "selection": "",
            "standard_midi_file": "",
            "xp_file_name": ""
                }

        for entry in perfblock.xpath(".//*[strong]"):
            k = entry.getchildren()[0].text_content()
            v = entry.text_content().strip()[len(k):]
            perfdict[k.strip(': ').replace(' ', '_').lower()] = v

        perflist.append(perfdict)
        perf2worklist.append({"standard_midi_file":perfdict["standard_midi_file"], "workid":workdict["workid"]})

with open('/tmp/worklist.json', 'wb') as workfile:
    workfile.write(orjson.dumps(worklist))
with open('/tmp/perflist.json', 'wb') as perffile:
    perffile.write(orjson.dumps(perflist))
with open('/tmp/perf2worklist.json', 'wb') as perf2workfile:
    perf2workfile.write(orjson.dumps(perf2worklist))
```

I let duckdb ingest the resulting jsons, to create three tables (work, performance, and a table relating primary keys from the two); then did a bit more post-processing with duckdb's easy to use string functions; then exported the result to a sqlite3 database and a json with the rough informal schema:

```
array of composers:
  array of works:
      work title,
      arranger/transcriber,
      composition date,
      array of performances:
        performer,
        duration,
        competition performed at,
        date of recording
        standard file name at the chelate1118/piano-e-competition-midi-files repo
        processed file name at the chelate1118/piano-e-competition-midi-files repo
```
        
