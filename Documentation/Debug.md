# Real-Time WebSocket Chat: Debugging

---

### Issue 1: Nginx served its default welcome page instead of the frontend

**Symptom:**
After running `docker-compose up -d --build`, both containers showed `Up` status, but visiting the public IP displayed Nginx's default "Welcome to nginx!" page instead of the chat frontend.

**Root Cause:**
In `docker-compose.yml`, the volume mount that maps the frontend directory into Nginx's web root was commented out:
```yaml
volumes:
  # TODO: Candidates need to map the frontend directory correctly to serve the HTML
  # - ./frontend:/usr/share/nginx/html:ro
  - ./nginx.conf:/etc/nginx/nginx.conf:ro
```
Without this line, `/usr/share/nginx/html` inside the container stayed empty, so Nginx fell back to serving its own built-in default page.

**Fix:**
Uncommented and enabled the frontend mount:
```yaml
volumes:
  - ./frontend:/usr/share/nginx/html:ro
  - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

**Verification:**
```bash
docker-compose down
docker-compose up -d --build
docker exec -it chat-nginx ls /usr/share/nginx/html
#list index.html and other frontend files, not the default nginx page
```
Then reloaded the public IP in the browser and confirmed the actual chat UI loaded instead of the Nginx default page.

### Issue 2: Backend unreachable from Nginx — Connection Refused

**Symptom:**
Frontend loaded correctly after fixing the volume mount, but any chat/WebSocket action failed. Both containers were confirmed running on the same default Docker Compose network, so networking itself wasn't expected to be the problem.

**Debugging Steps:**
```bash
# Exec into the nginx container and try reaching the backend directly
docker exec chat-nginx wget -qO- http://backend:8000
```

**Exact error observed:**
```
ubuntu@ip-172-31-6-197:~/Real-Time-WebSocket-Application-$ docker exec chat-nginx wget -qO- http://backend:8000
wget: can't connect to remote host (172.18.0.2): Connection refused
```

DNS resolution for `backend` succeeded (resolved to `172.18.0.2`), ruling out a networking/DNS problem. `Connection refused` specifically pointed to the backend process rejecting the connection at the socket level.

**Root Cause:**
In `Dockerfile`, uvicorn was started with:
```dockerfile
CMD ["uvicorn", "main:app", "--host", "127.0.0.1", "--port", "8000"]
```
Binding to `127.0.0.1` makes the app only listen for connections from **inside its own container**. Since Nginx runs in a separate container, its requests to `backend:8000` were refused outright — the process wasn't listening on the container's external network interface at all.

**Fix:**
```dockerfile
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Verification:**
```bash
docker exec chat-nginx wget -qO- http://backend:8000
# now returns an HTTP 500 instead of "Connection refused" — confirms the backend is reachable and listening
```
(The remaining 500 error was investigated separately as Issue 3, since it pointed to a different misconfiguration.)

---

### Issue 3: 502 Bad Gateway on `/ws` — Wrong upstream host AND missing WebSocket upgrade headers

**Symptom:**
After fixing the backend's bind address (Issue 2), `wget` from Nginx to the backend returned an HTTP response (progress), but the browser still failed to establish a WebSocket connection on `/ws`, returning `502 Bad Gateway`.

**Confirming the backend was reachable:**
```
ubuntu@ip-172-31-6-197:~/Real-Time-WebSocket-Application-$ docker exec chat-nginx wget -qO- http://backend:8000
wget: server returned error: HTTP/1.1 500 Internal Server Error
```
Getting back an actual HTTP status code (even a 500) confirms the TCP connection succeeded and the backend application processed the request — a `Connection refused` or timeout would mean the network layer itself was broken, whereas a 500 only happens *after* a successful handshake and request/response cycle at the HTTP level.

**Current `nginx.conf` at this point:**
```nginx
location /ws {
    proxy_pass http://localhost:8000/ws;
    proxy_http_version 1.1;
    # proxy_set_header Upgrade $http_upgrade;
    # proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 86400s;
    proxy_send_timeout 86400s;
}
```

**Root Cause:**
Two separate problems in the same block:

1. `proxy_pass http://localhost:8000/ws` pointed Nginx at itself rather than the backend container — fixed by changing to the Docker Compose service name `backend`.
2. Even after that fix, the `Upgrade` and `Connection` headers were **commented out**:
   ```nginx
   # proxy_set_header Upgrade $http_upgrade;
   # proxy_set_header Connection "upgrade";
   ```
   A WebSocket connection starts life as a normal HTTP request that asks the server to "upgrade" the protocol from HTTP to WebSocket, using the `Upgrade: websocket` and `Connection: Upgrade` headers. By default, when Nginx proxies a request upstream, it does **not** forward these headers automatically — it treats the connection as a standard `Connection: close` HTTP proxy request. Without explicitly re-adding these two headers in the `proxy_set_header` directives, the backend never receives a proper upgrade request, so the WebSocket handshake never completes, and Nginx has no valid persistent connection to route traffic through — resulting in `502 Bad Gateway`.

**Fix:**
Uncommented both headers so the upgrade request is correctly forwarded to the backend:
```nginx
location /ws {
    proxy_pass http://backend:8000/ws;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 86400s;
    proxy_send_timeout 86400s;
}
```

**Verification:**
```bash
docker-compose down
docker-compose up -d --build

docker logs chat-nginx
# no more "connect() failed" or "no live upstreams" errors on /ws
```
Reloaded the app in the browser's Network tab and confirmed the `/ws` request returned `101 Switching Protocols` instead of `502`, with chat messages sending and receiving in real time across multiple browser tabs.

---


