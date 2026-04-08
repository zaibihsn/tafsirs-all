# Developer Implementation Guide

This guide provides technical details on how to integrate the tafsir data from this repository into your Web, Android (Flutter/Native), or Backend applications using the jsDelivr CDN.

## 1. Data Structure Overview

All tafsirs are organized into Surah-based files (`1.toon` to `114.toon`) within their respective slug directories.

### jsDelivr CDN Pattern
```text
https://cdn.jsdelivr.net/gh/zaibihsn/tafsirs-all@master/{tafsir-slug}/{surah-number}.toon
```
*Replace `{tafsir-slug}` with one from the catalog and `{surah-number}` with 1-114.*

---

## 2. The .toon Format

The `.toon` format is a lightweight, line-based format designed for efficient parsing.

### File Structure Example
```text
quran[7]{c,v,t}:
  1,1,"In the name of Allah..."
  1,2,"All praise is due to Allah..."
```

- **Header**: `label[count]{fields}:`
    - `label`: Usually `quran` or `tafsir`.
    - `count`: Total verses in this surah.
    - `fields`: Field order (e.g., `c`=Chapter, `v`=Verse, `t`=Text).
- **Entries**: Starts with two leading spaces.
- **Quotes**: Text is always double-quoted. Supporting multi-line text is required.

---

## 3. Implementation Examples

### A. JavaScript / Web (Fetch & Parse)

```javascript
/**
 * Fetches and parses a tafsir surah file.
 */
async function fetchTafsir(slug, surahNum) {
    const url = `https://cdn.jsdelivr.net/gh/zaibihsn/tafsirs-all@master/${slug}/${surahNum}.toon`;
    const response = await fetch(url);
    const content = await response.text();

    const verses = [];
    // Regex matches: key part, then everything inside the double quotes
    const entryRegex = /^\s+([^,"]+[,:][^,"]+|[^,]+,[^,]+),"(.*?)"(?=\s*^\s+\d+[,:]\d+,|\s*\Z)/gms;
    
    let match;
    while ((match = entryRegex.exec(content)) !== null) {
        const [full, key, text] = match;
        const [c, v] = key.split(/[,:]/);
        verses.push({
            surah: parseInt(c),
            verse: parseInt(v),
            text: text.trim()
        });
    }
    return verses;
}

// Usage
fetchTafsir('arabic-muyassar', 1).then(data => console.log(data));
```

### B. Dart / Flutter (Mobile Development)

```dart
import 'package:http/http.dart' as http;

class TafsirEntry {
  final int surah;
  final int verse;
  final String text;
  TafsirEntry({required this.surah, required this.verse, required this.text});
}

Future<List<TafsirEntry>> fetchTafsir(String slug, int surahNum) async {
  final url = 'https://cdn.jsdelivr.net/gh/zaibihsn/tafsirs-all@master/$slug/$surahNum.toon';
  final response = await http.get(Uri.parse(url));
  
  if (response.statusCode != 200) throw Exception('Failed to load tafsir');

  final content = response.body;
  final entries = <TafsirEntry>[];
  
  // Robust line processing
  final regExp = RegExp(
    r'^\s+([^,"]+[,:][^,"]+|[^,]+,[^,]+),"(.*?)"(?=\s*^\s+\d+[,:]\d+,|\s*\Z)',
    multiLine: true,
    dotAll: true,
  );

  for (final match in regExp.allMatches(content)) {
    final key = match.group(1)!;
    final text = match.group(2)!;
    final parts = key.split(RegExp(r'[,:]'));
    
    entries.add(TafsirEntry(
      surah: int.parse(parts[0]),
      verse: int.parse(parts[1]),
      text: text.trim(),
    ));
  }
  return entries;
}
```

### C. Python (Backend / Scripting)

```python
import re, requests

def parse_toon(content):
    # Regex to handle multi-line strings
    pattern = r'(?m)^\s+([^,"]+[,:][^,"]+|[^,]+,[^,]+),"(.*?)"(?=\s*^\s+\d+[,:]\d+,|\s*\Z)'
    matches = re.findall(pattern, content, re.DOTALL)
    
    results = []
    for key, text in matches:
        s, v = re.split(r'[,:]', key)
        results.append({
            'surah': int(s),
            'verse': int(v),
            'text': text.strip()
        })
    return results

# Usage
content = requests.get("LINK_TO_CDN_FILE").text
data = parse_toon(content)
```

---

## 4. Best Practices

### Caching
Data in this repository is stable. It is highly recommended to cache `.toon` files locally on devices (using `shared_preferences` or `sqflite` in Mobile, or `localStorage`/`IndexedDB` in Web) to reduce CDN traffic and improve offline availability.

### Verse Normalization (6,236)
Every tafsir follows the 6,236-verse standard. If a tafsir has an original count lower than 6,236, missing verses will contain an empty string `""`.
> **Recommendation**: Handle empty strings in your UI (e.g., skip the verse or show "Commentary not available for this verse" if desired).

### Character Encoding
All files are **UTF-8** encoded. Ensure your application's network client and UI are configured to handle UTF-8, especially for Arabic and other non-Latin scripts.

---

## 5. Support
For issues with data or specific tafsir requests, please [open an issue](https://github.com/zaibihsn/tafsirs-all/issues) on GitHub.
