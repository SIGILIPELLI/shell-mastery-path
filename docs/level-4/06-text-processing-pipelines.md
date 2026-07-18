# 06 · Advanced Text Processing Pipelines

`grep`, `awk`, `sed`, and `jq` are each capable on their own, but production
data-processing scripts get their power from chaining them: filter with
`grep`, extract/transform with `awk`, reshape with `sed`, and query/reshape
JSON with `jq`. This module builds several real multi-stage pipelines,
including converting between plain text, CSV, and JSON.

## Thinking in pipeline stages

Most text pipelines follow the same shape: **filter → extract → transform →
aggregate**. Naming the stage you're on makes a long pipeline easy to debug
one `|` at a time.

```bash
# filter (grep) -> extract (awk) -> aggregate (sort | uniq -c) -> rank (sort -rn)
grep 'ERROR' app.log | awk '{print $1, $2}' | sort | uniq -c | sort -rn | head -5
```

Build pipelines incrementally: run the first stage alone, confirm the
output looks right, then pipe in the next stage — trying to write a five-
stage pipeline in one shot is how silent mistakes creep in.

## grep to filter, awk to extract

```bash
# pull just the client IP and status code for 5xx responses in an access log
grep ' 5[0-9][0-9] ' access.log | awk '{print $1, $9}'
```

```bash
# keep only lines mentioning a given user, then extract the action column
grep 'user=alice' audit.log | awk -F'action=' '{print $2}' | awk '{print $1}'
```

## awk to extract, sed to reshape

```bash
# turn "name=alice age=30 city=nyc" lines into "alice,30,nyc"
echo "name=alice age=30 city=nyc" \
    | awk '{print $1, $2, $3}' \
    | sed -E 's/[a-z]+=//g; s/ /,/g'
# alice,30,nyc
```

## Introducing jq for JSON

```bash
echo '{"name":"alice","age":30}' | jq '.name'
# "alice"

echo '{"name":"alice","age":30}' | jq -r '.name'    # -r strips the quotes
# alice
```

`jq` filters chain with `|` just like shell pipes, but inside a single
`jq` argument — `.foo.bar`, `.[0]`, and `select(...)` are the building
blocks for everything below.

## jq: navigating and filtering arrays

```bash
cat users.json
# [{"name":"alice","active":true},{"name":"bob","active":false}]

jq '.[] | select(.active == true) | .name' users.json
# "alice"

jq -r '.[] | select(.active) | .name' users.json
# alice
```

## jq: reshaping objects with map

```bash
jq '[.[] | {user: .name, status: (if .active then "active" else "inactive" end)}]' users.json
# [{"user":"alice","status":"active"},{"user":"bob","status":"inactive"}]
```

## Combining jq with curl, awk, and grep

```bash
# fetch a JSON API, keep only active users, uppercase their names
curl -s https://api.example.com/users \
    | jq -r '.[] | select(.active) | .name' \
    | awk '{ print toupper($0) }'
```

## A full pipeline: log analysis to a ranked JSON report

```bash
# access.log fields: $1=IP ... $9=status code
awk '{ print $1, $9 }' access.log \
    | sort \
    | uniq -c \
    | awk '{ printf "{\"ip\":\"%s\",\"status\":\"%s\",\"count\":%d}\n", $3, $2, $1 }' \
    | jq -s 'sort_by(-.count)'
```

Reading it stage by stage: `awk` extracts IP + status, `sort | uniq -c`
counts identical pairs, the second `awk` emits one JSON object per line,
and `jq -s` ("slurp") reads all of those separate JSON lines into a single
sorted array.

## Converting CSV to JSON and back with jq

```bash
# CSV -> JSON (first row is the header)
awk -F, 'NR==1 { split($0, h, ","); next }
         { printf "{"; for (i=1; i<=NF; i++)
               printf "\"%s\":\"%s\"%s", h[i], $i, (i<NF ? "," : "");
           print "}" }' data.csv | jq -s '.'
```

```bash
# JSON -> CSV (array of flat objects, using the first object's keys as the header)
jq -r '(.[0] | keys_unsorted) as $keys | $keys, (.[] | [.[$keys[]]]) | @csv' data.json
```

## Cheat sheet

| Tool | Best for |
|------|----------|
| `grep` | filtering lines by pattern |
| `awk` | field extraction, math, reports |
| `sed` | line-based text substitution/reshaping |
| `jq` | querying/transforming JSON |
| `jq -r` | raw (unquoted) string output |
| `jq -s` | slurp multiple JSON lines/values into one array |
| `sort \| uniq -c` | count duplicate lines |
| `@csv` (in jq) | render an array as a CSV row |

## Exercise

Given an `access.log` in Common Log Format, write a single pipeline that
extracts the requested path and status code, keeps only 4xx/5xx responses,
counts occurrences per path, and emits the result as a JSON array of
`{"path": ..., "count": ...}` objects sorted by count descending. Verify
the output is valid JSON by piping it through `jq length`.
