# gh-trawler-wordlist

Credential wordlists built from default passwords, left-in-place placeholders and
hardcoded secrets found in config files in public GitHub repositories.

This is the output of [gh-trawler](https://github.com/BokkaPig/gh-trawler). See
that repo for how the data is collected, classified and filtered.

**Scope note.** Every value here was found in a public repository. Seeing a value
in this list tells you what to *try* against a target. Checking whether it still
works on that target is separate work, and you need authorisation to do it.

---

## Files

| File | Entries | Contents |
|---|---|---|
| `passwords.txt` | 1,888 | Secrets, ranked by how many distinct repos ship them. |
| `users.txt` | 981 | Usernames, same ranking. |
| `combos.txt` | 1,832 | `user:pass` pairs seen in the same file. Feed to `hydra -C`. |
| `placeholders.txt` | 850 | Only the left-in-place prose (`changeme`, `your_api_key_here`). Usually worth trying first. |
| `defaults.tsv` | 2,869 | Every entry with repo count, kind, class and the most common key name. |
| `excluded.tsv` | 1,778 | Every value filtered out of the wordlists, with the reason. |

Lists are ordered by how many repos ship each value. A value shipped by 20 repos
is almost certainly a default. A value seen once is probably specific to one
project.

## Usage

```bash
hydra -L users.txt -P passwords.txt <target> http-post-form ...
hydra -C combos.txt <target> ssh
ffuf -w placeholders.txt ...
```

## `defaults.tsv`

Columns: `repos`, `kind` (`user` / `secret`), `class`, `value`, `top_key`.

| Class | Count | Meaning |
|---|---|---|
| `identity` | 981 | A username rather than a secret. |
| `placeholder_default` | 850 | Text the deployer was told to replace, and often didn't. |
| `medium_default` | 446 | Ordinary literal (`rootpass`, `Password123`). |
| `weak_default` | 326 | Short or low-entropy literal (`admin`, `minioadmin`). |
| `high_entropy_default` | 266 | Long, random-looking secret committed to the repo. |

Use the `class` column to split findings in a report. A `JWT_SECRET` or
`SESSION_SECRET` left at its default lets an attacker forge tokens offline, which
is a different finding from "admin:admin still works". `top_key` shows what the
value was configured as.

```bash
# placeholder secrets only
awk -F'\t' '$3=="placeholder_default"{print $4}' defaults.tsv

# signing keys and session secrets
awk -F'\t' 'tolower($5) ~ /jwt|session|signing/{print $4}' defaults.tsv
```

## Filtering

Every wordlist entry is meant to be sprayed exactly as written. A value was
excluded if it:

- was shorter than 4 characters
- was longer than 128 characters
- was non-ASCII
- contained whitespace
- had no alphanumeric characters
- looked like a path or URL
- was a JSON or template fragment

Interpolated values (`${JWT_SECRET}`, `{{ .Values.pw }}`) are dropped entirely.
Everything else that was filtered out is listed in `excluded.tsv` with its reason,
so you can pull any of it back.

To build a different cut, such as `--min-length 8` or a single class, run
[gh-trawler](https://github.com/BokkaPig/gh-trawler) yourself.
