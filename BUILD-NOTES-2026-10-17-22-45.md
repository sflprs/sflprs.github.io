# How this archive was built

Notes on how the files in the SFLPRS repositories were made, so they can be updated or rebuilt later. Recovered on 2026-10-17 from the October 10–11, 2026 Claude session that did the work.

**Short version:** you don't need the original scripts to update the archive. Everything they worked out is already saved in the repositories:

- **Which photo went where** is recorded in each repository's `manifest.csv` (`original_path` → `new_path`).
- **Volume titles, page order, dates and rotations** are in `viewer-data.js`.
- **OCR text** is in each repository's `text/` folder, and copied into `viewer-text.js` for search.

So most future changes are either small hand edits, or a short new script that reads these files. A full re-run from the raw scans is only needed to redo the OCR from scratch.

> ⚠️ `viewer-data.js` has been hand-edited since it was generated (for example on 2026-10-17). Treat the current file as the master copy. Never overwrite it with a freshly generated one without carrying those edits over.

## Where things stand

- Working copy: `G:\My Drive\2026-theo-github\sflprs\` (moved from `D:\SFLPRS-Archive`, which is kept as a frozen backup).
- Source scans: the Czur folders on D: (`D:\Czur-2026-10-10`, and an identical copy in `D:\2025-10-27-czur-scans-2`). These were never changed.
- Tools used: Python 3 with Pillow and NumPy, Tesseract OCR, ImageMagick, `md5sum`. All of it ran in Claude's cloud workspace or the Cowork VM, not on Windows. Nothing here is meant to run on Theo's machine.

## The pipeline, step by step

| # | Step | What it did | Output |
|---|---|---|---|
| 1 | Hash | `md5sum` of every `.jpg` in the scan folder | `all_md5.txt` (4,099 photos) |
| 2 | Dedupe | Dropped exact duplicates: the 626 loose files in `Heritage Minutes Dau 4` (copies of its `2025_03_17_12_25_34` subfolder) and 5 top-level files (copies of Day 3 pages) | `unique.tsv`: 3,468 photos, numbered `0001`–`3468` in path order |
| 3 | Metadata | Size, pixel dimensions and modified time of each original | `unique_meta.tsv` |
| 4 | Reduce | Half-size grayscale JPEG of each photo, for OCR only | `ocrimg/NNNN.jpg` (temporary) |
| 5 | OCR | `tesseract img.jpg - --psm 1` (later `--psm 3`) on each reduced image | `txt/NNNN.txt` |
| 6 | Quality check | Scored each page by the share of its words that are common across the whole archive. Pages under 0.55 were flagged | `bad.txt`, narrowed to `bad2.txt` |
| 7 | Sideways pages | Flagged pages were rotated 90° and 270° and read again. The rotation was kept if it found at least 30% more real words (and more than 20) | `rot/NNNN.txt` and the rotation angle (378 pages) |
| 8 | Re-shots | Compared 4-word phrases between pages. A word-overlap (Jaccard) of 0.6 or more marks a possible re-shot | `dups.json`, then the `possible_reshot_of` column |
| 9 | Volumes | A hand-written table (`volumes.py`) listing each series and volume, its label, repository, and which run of photos in which scan folder belongs to it. Worked out by reading spine and cover photos and the dates in the OCR | `volumes.py` |
| 10 | Build | For each volume: assigned pages in photo order, named `<code>_<page>` (e.g. `T2_0001`), found the first date near the top of each page, and worked out the year range | `built.json`, `texts.json` |
| 11 | Emit | Wrote `manifest.csv`, `checksums-md5.txt`, `duplicates-not-copied.csv`, `text/…/*.txt`, `viewer-data.js` and `viewer-text.js` | the files in the repos |
| 12 | Copy | Copied the original photos byte-for-byte to their new names, and made the thumbnails | `images/`, `thumbs/` |
| 13 | Split | Regrouped everything into the 7 series repositories plus `sflprs.github.io`, then renamed the Trustees, Crocker and Directors repos to add year ranges | the current layout |
| 14 | Verify | Checked that every manifest row has its image, text and thumbnail, and that every page in `viewer-data.js` exists on disk | — |

## File formats

### `viewer-data.js`

One JavaScript constant. Paraphrased shape:

```js
const ARC = {
  series: [
    { id, title, note, folder,
      volumes: [
        { id: "T2", title: "Trustees Meetings 1930–1955",
          repo: "board-of-trustees-1906-1999",
          folder: "T2_Trustees-Meetings-1930-1955",
          note, years: "1930–1955",
          pages: [ { n: "T2_0001", d: "1930-01-15" },          // d = date found, "" if none
                   { n: "T2_0002", d: "", r: 90 } ] }        // r = text rotation, only when turned
      ] }
  ]
};
```

The viewer builds each image path as `<repo>/images/<folder>/<n>.jpg`, and the thumbnail and text paths the same way. **To rename a volume, fix a date or move a page, edit this file directly** (back it up first).

### `viewer-text.js`

```js
const TXT = { "<repo>/<volume folder>/<page>": "OCR text…", … };
```

The same text is also in each repo's `text/<volume folder>/<page>.txt`. If you correct OCR text by hand, change both, or rebuild this file from the `text/` folders.

### `manifest.csv` (one per repository, plus a combined one in `sflprs.github.io`)

Columns: `repository, new_path, text_path, original_path, md5, bytes, width, height, original_modified, series, volume, volume_title, part, page, ocr_date, ocr_chars, text_rotated, possible_reshot_of`.

`part` is A or B when a volume was photographed across two sessions. `text_rotated` is the rotation in degrees for sideways pages.

## Common jobs

- **Fix a volume title or a page date:** edit `viewer-data.js` by hand. Update `volume_title` or `ocr_date` in the manifests if you want them to match.
- **Add a new volume:** ask Claude to (1) hash the new photos and check them against `checksums-md5.txt` for duplicates, (2) OCR them, (3) copy them into the right repository as `<code>_NNNN.jpg` with thumbnails and text, and (4) **append** a volume entry to the existing `viewer-data.js`, rows to the manifests, and entries to `viewer-text.js`, without regenerating them. Keep each repository under 1 GB; at the moment the largest is about 720 MB.
- **Redo the OCR for some pages:** re-run Tesseract on the images listed in the manifest, then replace those entries in `text/` and `viewer-text.js`.
- **Check the files are intact:** compare each repository's `checksums-md5.txt` against its `images/` folder.

## Recovered scripts

These are the scripts as written in the session. They use the session's temporary paths (`/home/claude/ocr`, `/home/claude/build`) and the original flat layout, before the split into repositories. They are a record of the method, not something to run as-is.

**Not fully recovered:** `volumes.py` (the volume table) and the final `build.py` were edited many times in the session by search-and-replace. The method is shown below, and the result is fully captured in `manifest.csv` and `viewer-data.js`. The thumbnail and repository-split scripts were not recovered.

### Reduce images for OCR (step 4)

```python
import os,sys,time
from PIL import Image
from concurrent.futures import ProcessPoolExecutor
R=os.path.expanduser('~/mnt/Czur-2026-10-10')
OUT=R+'/_build-temp/ocrimg'
rows=[l.rstrip('\n').split('\t') for l in open(os.path.expanduser('~/work/unique.tsv'))]
def do(r):
    i,h,p=r
    o=f'{OUT}/{i}.jpg'
    if os.path.exists(o): return None
    im=Image.open(f'{R}/{p}'); W,H=im.size
    im.draft('L',(W//2,H//2)); im=im.convert('L')
    im.save(o+'.tmp','JPEG',quality=80); os.rename(o+'.tmp',o)
    return f'{i}\t{W}\t{H}'
t=time.time(); n=0
with ProcessPoolExecutor(2) as ex:
    with open(os.path.expanduser('~/work/dims.tsv'),'a') as f:
        for res in ex.map(do,rows,chunksize=4):
            if res: f.write(res+'\n'); n+=1
            if time.time()-t>150: break
print(n, time.time()-t)
```

### OCR quality check (step 6): `quality.py`

```python
import os,re,collections
os.chdir('/home/claude/ocr')
T={f[:-4]:open('txt/'+f,errors='ignore').read() for f in os.listdir('txt') if re.match(r'^\d+\.txt$',f)}
cnt=collections.Counter(w for t in T.values() for w in re.findall(r'[a-z]{3,}',t.lower()))
vocab={w for w,c in cnt.items() if c>=15}
bad=[]
for i,t in T.items():
    toks=re.findall(r'\S+',t)
    if len(toks)<15: continue
    good=sum(1 for w in toks if re.sub(r'[^a-z]','',w.lower()) in vocab)
    q=good/len(toks)
    if q<0.55: bad.append((i,round(q,2)))
bad.sort()
open('bad.txt','w').write('\n'.join(i for i,q in bad)+'\n')
print(len(bad))
print(sorted(collections.Counter(int(i)//100 for i,q in bad).items()))
```

### Re-read sideways pages (step 7): `rotocr.py`

```python
import os,re,sys,subprocess,collections
from PIL import Image
from concurrent.futures import ProcessPoolExecutor
O='/home/claude/ocr'; os.makedirs(O+'/rot',exist_ok=True)
T={f[:-4]:open(O+'/txt/'+f,errors='ignore').read() for f in os.listdir(O+'/txt') if re.match(r'^\d+\.txt$',f)}
cnt=collections.Counter(w for t in T.values() for w in re.findall(r'[a-z]{3,}',t.lower()))
V={w for w,c in cnt.items() if c>=15}
def q(t):
    toks=re.findall(r'\S+',t)
    if not toks: return 0,0
    g=sum(1 for w in toks if re.sub(r'[^a-z]','',w.lower()) in V)
    return g/len(toks), g
def run(i):
    out=f'{O}/rot/{i}.res'
    if os.path.exists(out): return
    best=(q(T.get(i,''))[1],0,T.get(i,''))
    for ang in (90,270):
        tmp=f'/tmp/r_{i}_{ang}.png'
        Image.open(f'{O}/img/{i}.jpg').rotate(ang,expand=True).save(tmp)
        t=subprocess.run(['tesseract',tmp,'-','--psm','3'],capture_output=True,text=True,env={**os.environ,'OMP_THREAD_LIMIT':'1'}).stdout
        os.remove(tmp)
        g=q(t)[1]
        if g>best[0]*1.3 and g>20: best=(g,ang,t)
    if best[1]:
        open(f'{O}/rot/{i}.txt','w').write(best[2])
    open(out,'w').write(str(best[1]))
ids=[l.strip() for l in open(O+'/bad2.txt') if l.strip()]
with ProcessPoolExecutor(2) as ex: list(ex.map(run,ids))
open(O+'/rot/DONE','w').write('1')
```

### Find likely re-shots (step 8): `dups.py`

```python
import os,re,json,collections
os.chdir('/home/claude/ocr')
meta={l.split('\t')[0]:l.split('\t')[2] for l in open('unique_meta.tsv')}
def folder(p): return os.path.dirname(p)
T={f[:-4]:open('txt/'+f,errors='ignore').read() for f in os.listdir('txt') if re.match(r'^\d+\.txt$',f)}
S={}
for i,t in T.items():
    w=re.findall(r'[a-z]{3,}',t.lower())
    sh={' '.join(w[k:k+4]) for k in range(len(w)-3)}
    if len(sh)>=40: S[i]=sh
idx=collections.defaultdict(list)
for i,sh in S.items():
    for s in sh: idx[s].append(i)
pairs=collections.Counter()
for s,ids in idx.items():
    if 2<=len(ids)<=6:
        for a in range(len(ids)):
            for b in range(a+1,len(ids)): pairs[(ids[a],ids[b])]+=1
res=[]
for (a,b),c in pairs.items():
    if c<15: continue
    j=len(S[a]&S[b])/len(S[a]|S[b])
    if j>=0.4: res.append((a,b,round(j,2)))
cross=collections.Counter((folder(meta[a]),folder(meta[b])) for a,b,j in res)
for k,v in cross.most_common(): print(v,k)
json.dump(res,open('/home/claude/build/dups.json','w'))
```

### Build volumes and dates (step 10): core of `build.py` (partial)

Each volume in `volumes.py` lists `parts`: tuples of (scan folder, first index, last index) in photo order. The first date found in the top ~700 characters of a page is used, unless it is more than 8 years from the volume's median year (30 for T1). The year range drops the first and last 1% of dated pages, to resist OCR misreads.

```python
def first_date(t):
    for m in RX.finditer(t[:700]):
        d, y = int(m.group(2)), int(m.group(3))
        if 1 <= d <= 31 and 1853 <= y <= 2025:
            return (y, MON[m.group(1)[:3].lower()], d)
    return None

def slug(s):
    s = s.replace('’', '').replace('–', '-')
    return re.sub(r'[^A-Za-z0-9]+', '-', s).strip('-')

# for each series s and volume v in volumes.SERIES:
#   ids   = photos from v['parts'], in order
#   raw   = {i: first_date(text(i))}; med = median year
#   dates = raw, but None where |year - med| > span
#   vol_folder = f"{v['code']}_{slug(v['label'])}"
#   page name  = f"{v['code']}_{seq:04d}"
#   new_path   = f"{v['repo']}/images/{vol_folder}/{name}.jpg"
#   pages.append(dict(n=name, d='YYYY-MM-DD' or '', r=rotation if any))
#   possible re-shots: pairs from dups.json with overlap >= 0.6
```

### Write the output files (step 11): `emit.py` (before the split into repos)

```python
import json, os, csv, re, collections
B='/home/claude/build'; OUT=B+'/out'
import shutil; shutil.rmtree(OUT, ignore_errors=True); os.makedirs(OUT)
d=json.load(open(B+'/built.json')); texts=json.load(open(B+'/texts.json'))
rows, arc = d['rows'], d['arc']
# text files
for path,t in texts.items():
    p=f'{OUT}/text/{path}.txt'; os.makedirs(os.path.dirname(p),exist_ok=True)
    open(p,'w',encoding='utf-8').write(t+'\n')
# manifest
cols=['new_path','text_path','original_path','md5','bytes','width','height','original_modified','series','volume','volume_title','part','page','ocr_date','ocr_chars','text_rotated','possible_reshot_of']
with open(f'{OUT}/manifest.csv','w',newline='',encoding='utf-8') as f:
    w=csv.DictWriter(f,fieldnames=cols,extrasaction='ignore'); w.writeheader()
    for r in rows:
        r['text_path']=r['new_path'].replace('images/','text/',1)[:-4]+'.txt'
        w.writerow(r)
# duplicates left out
md5_new={r['md5']:r['new_path'] for r in rows}
kept={r['original_path'] for r in rows}
with open(f'{OUT}/duplicates-not-copied.csv','w',newline='',encoding='utf-8') as f:
    w=csv.writer(f); w.writerow(['original_path','md5','identical_to'])
    n=0
    for l in open('/home/claude/ocr/all_md5.txt'):
        h,p=l.rstrip('\n').split('  ',1); p=p[2:]
        if p not in kept: w.writerow([p,h,md5_new[h]]); n+=1
print('dups',n)
with open(f'{OUT}/checksums-md5.txt','w') as f:
    for r in rows: f.write(f"{r['md5']}  {r['new_path']}\n")
# viewer data
for s in arc:
    for v in s['volumes']:
        v.pop('years_ocr',None)
open(f'{OUT}/viewer-data.js','w',encoding='utf-8').write('// Generated list of series, volumes and pages for index.html\nconst ARC = '+json.dumps(dict(series=arc),ensure_ascii=False,separators=(',',':'))+';\n')
open(f'{OUT}/viewer-text.js','w',encoding='utf-8').write('// OCR text of every page, keyed by page path (machine-read; expect errors)\nconst TXT = '+json.dumps(texts,ensure_ascii=False,separators=(',',':'))+';\n')
shutil.copy(B+'/index.html',OUT+'/index.html')
print(len(rows), sum(len(t) for t in texts.values()))
```

### Final check (step 14), after the split

```python
# for each series repo r:
for row in csv.DictReader(open(f'{r}/manifest.csv',encoding='utf-8')):
    assert row['repository']==r
    for p in (row['new_path'],row['text_path'],row['new_path'].replace('images/','thumbs/',1)):
        if not os.path.exists(f'{r}/{p}'): miss+=1
# and every page in viewer-data.js:
#   exists(f"{v['repo']}/images/{v['folder']}/{p['n']}.jpg")
```
