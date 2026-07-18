# 09 · Networking from the Shell

Shell scripts routinely need to talk to the network: fetching data, checking
whether a service is up, or scripting an API call. This module covers
`curl`, `wget`, and bash's lesser-known built-in socket support via
`/dev/tcp`.

## curl: the basics

```bash
curl https://example.com                       # GET request, prints body to stdout
curl -o page.html https://example.com            # save response to a file
curl -O https://example.com/file.zip              # save using the remote filename
curl -s https://example.com                        # silent: suppress the progress meter
curl -sS https://example.com                        # silent but still show errors
```

```bash
curl -I https://example.com                # HEAD request: headers only, no body
curl -w '%{http_code}\n' -o /dev/null -s https://example.com   # print just the status code
```

## curl: making requests with data

```bash
# GET with query parameters
curl -G --data-urlencode "q=shell scripting" https://example.com/search

# POST a JSON body
curl -X POST https://api.example.com/items \
    -H "Content-Type: application/json" \
    -d '{"name": "widget", "qty": 5}'

# send a custom header (e.g. an API token)
curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com/me
```

```bash
# fail (nonzero exit) on HTTP error codes instead of printing the error body as "success"
curl -fsS https://api.example.com/status
echo "exit code: $?"
```

`-f` is important in scripts: without it, curl exits 0 even on a 404 or 500,
because as far as curl is concerned the *transfer* succeeded — only `-f`
treats an HTTP error status as a curl failure.

## curl: retries and timeouts

```bash
curl --retry 3 --retry-delay 2 --max-time 10 https://api.example.com/data
# --retry 3: retry up to 3 times on transient failures
# --retry-delay 2: wait 2s between retries
# --max-time 10: give up entirely after 10s total
```

## Parsing JSON responses with jq

```bash
curl -s https://api.example.com/users/1 | jq '.name'
curl -s https://api.example.com/users | jq -r '.[] | "\(.id): \(.name)"'
```

`jq -r` outputs raw strings (no surrounding quotes) — the right choice when
piping the result into further shell commands.

## wget: an alternative for downloads

```bash
wget https://example.com/file.zip                 # download, keep the remote filename
wget -O out.zip https://example.com/file.zip         # download to a specific filename
wget -q https://example.com/file.zip                   # quiet mode
wget -c https://example.com/bigfile.iso                 # resume a partial download
wget --mirror --no-parent https://example.com/docs/       # mirror a whole directory tree
```

`curl` and `wget` overlap heavily for simple downloads; `wget` tends to be
preferred for recursive/mirroring jobs, `curl` for scripting APIs (richer
control over headers, methods, and request bodies).

## Waiting for a service to become available

A very common script pattern — polling until a port responds, e.g. before
starting a dependent process in CI or a deploy script:

```bash
wait_for_port() {
    local host="$1" port="$2" timeout="${3:-30}"
    local waited=0
    until (echo > "/dev/tcp/$host/$port") 2>/dev/null; do
        sleep 1
        waited=$((waited + 1))
        if (( waited >= timeout )); then
            echo "timed out waiting for $host:$port" >&2
            return 1
        fi
    done
    echo "$host:$port is up (after ${waited}s)"
}

wait_for_port localhost 5432 30
```

## bash's built-in `/dev/tcp` pseudo-device

Bash (not `sh`/dash) can open raw TCP connections without `curl`, `nc`, or
any external tool at all, using `/dev/tcp/HOST/PORT`:

```bash
# check if a port is open (no data sent)
if (echo > /dev/tcp/example.com/443) 2>/dev/null; then
    echo "port 443 is reachable"
else
    echo "port 443 is closed or unreachable"
fi
```

```bash
# a minimal raw HTTP GET request, no curl/wget needed
exec 3<>/dev/tcp/example.com/80
echo -e "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n" >&3
cat <&3
exec 3<&-        # close the file descriptor when done
```

This is useful for quick reachability checks in minimal environments, but
for anything beyond a raw health check, `curl` is far more robust (handles
TLS, redirects, compression, retries) — reach for `/dev/tcp` mainly when
no other tool is guaranteed to be installed.

## Cheat sheet

| Task | Command |
|------|---------|
| GET and print body | `curl https://host/path` |
| Fail script on HTTP error | `curl -fsS https://host/path` |
| POST JSON | `curl -X POST -H "Content-Type: application/json" -d '{...}' url` |
| Just the HTTP status code | `curl -w '%{http_code}' -o /dev/null -s url` |
| Download, resume-capable | `wget -c url` |
| Extract a JSON field | `curl -s url \| jq -r '.field'` |
| Check if a port is open | `(echo > /dev/tcp/host/port) 2>/dev/null` |

## Exercise

Write a script `wait-for-http.sh HOST PORT PATH` that polls
`http://HOST:PORT/PATH` every 2 seconds using `curl -fsS`, printing a dot
each attempt, until it gets a successful (2xx) response or 60 seconds pass
(whichever first) — then exits 0 on success or 1 on timeout. Test it against
a path that becomes available only after a short delay (e.g. `sleep 5 &&
python3 -m http.server 8000` in another terminal).
