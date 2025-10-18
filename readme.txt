\
Simple HTTPS Web Server on RHEL 8.1 with Python 3 and CGI support
================================================================

0) Create directories
---------------------
sudo mkdir -p /d9adm/simple_https/cgi_bin
sudo mkdir -p /d9adm/simple_https/certs
sudo chown -R "$USER":"$USER" /d9adm/simple_https
cd /d9adm/simple_https

1) Create HTML pages
--------------------
index.html
-----------
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>Number</title></head>
<body><h1 style="font-family:monospace">1234567890</h1></body>
</html>

input.html
-----------
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>Enter Number</title></head>
<body>
  <form action="/cgi_bin/submit.py" method="post">
    <label>Enter 10-digit number:</label>
    <input type="text" name="number" pattern="\\d{10}" maxlength="10" required>
    <button type="submit">Submit</button>
  </form>
</body>
</html>

2) CGI handler (submit.py)
--------------------------
#!/usr/bin/env python3
import cgi, html, re

print("Content-Type: text/html; charset=utf-8\\n")
form = cgi.FieldStorage()
num = (form.getfirst("number", "") or "").strip()
ok = bool(re.fullmatch(r"\\d{10}", num))
msg = f"You entered: {html.escape(num)}" if ok else "Error: need exactly 10 digits."
print(f"<!DOCTYPE html>\\n<html><head><meta charset='utf-8'><title>Result</title></head>"
      f"<body><h1>{msg}</h1><p><a href='/input.html'>Back</a></p></body></html>")

3) Create self-signed certificate
---------------------------------
HOSTNAME=<your-hostname-or-IP>
openssl req -x509 -newkey rsa:2048 -nodes -days 365 \\
  -keyout certs/server.key -out certs/server.crt \\
  -subj "/CN=${HOSTNAME}" \\
  -addext "subjectAltName=DNS:${HOSTNAME},IP:${HOSTNAME}"

4) HTTPS server (https_server.py)
---------------------------------
#!/usr/bin/env python3
import os, ssl
from http.server import CGIHTTPRequestHandler, ThreadingHTTPServer

PORT = 9443
BASE = os.path.dirname(os.path.abspath(__file__))
os.chdir(BASE)

class MyHandler(CGIHTTPRequestHandler):
    cgi_directories = ["/cgi_bin"]

httpd = ThreadingHTTPServer(("0.0.0.0", PORT), MyHandler)
context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
context.load_cert_chain("certs/server.crt", "certs/server.key")
httpd.socket = context.wrap_socket(httpd.socket, server_side=True)

print(f"Serving HTTPS on https://0.0.0.0:{PORT}")
httpd.serve_forever()

5) Run
------
cd /d9adm/simple_https
python3 ./https_server.py

6) Access
---------
https://<HOSTNAME>:9443/index.html
https://<HOSTNAME>:9443/input.html
