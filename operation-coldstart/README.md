<img src="assets/ctf-logo.png" width="700">

# Operation Coldstart - Writeup

Operation Coldstart is a Linux-based room focused on chaining multiple small weaknesses into full system compromise. The attack path involves anonymous FTP access, source code disclosure, access to internal resources through a URL preview functionality, and a privilege escalation caused by an insecure cron job using `tar` with an unquoted wildcard.

## Reconnaissance

I started with an Nmap scan to identify the exposed services running on the target.

<img src="assets/nmap-scan.png" width="700">

The scan revealed three open ports:

+ 21/tcp — FTP
+ 22/tcp — SSH
+ 80/tcp — HTTP

Two details immediately stood out during enumeration:

+ The FTP service allowed anonymous authentication (Anonymous FTP login allowed)
+ The HTTP title identified the application as a URL Preview Service (Strongly hinting a possible SSRF attack)

## Web Enumeration

After identifying the web service, I began enumerating the application directories using Gobuster.

<img src="assets/gobuster-enum.png" width="700">

During the scan, Gobuster revealed an interesting endpoint:

`/admin/`

Attempting to access the directory directly resulted in an access denied response.

<img src="assets/forbidden-access.png" width="700">

## FTP Enumeration

Since direct access to the /admin directory was not possible, I decided to further investigate the FTP service.

While enumerating the available files, I discovered a compressed backup archive named:

`backup.tar.gz`

<img src="assets/ftp-file.png" width="700">

## Source Code Analysis

After downloading the backup archive, I extracted its contents and began reviewing the application files.

Inside the backup, I found the main Flask application source code in a file named:

`app.py`

While reviewing the code, I identified the following configuration:

ALLOWED_HOSTS = {"kestrel.thm"}

<img src="assets/allowed-host.png" width="700">

Further down in the source code, I found an interesting administrative route:

<img src="assets/code-block.png" width="700">

The source code revealed two critical details:

+ Local-Only Access: The `/admin` endpoint enforces a restriction (`127.x.x.x`) to ensure it can only be accessed locally.
+ Sensitive Target: Accessing the `/admin/notes` path triggers the server to read and display the contents of `admin_notes.txt`.

Since the application processes server-side requests via its URL preview functionality, this behavior strongly suggested that the internal administrative endpoint could be accessed indirectly.

## Exploiting SSRF

Because the application implicitly trusts internal routing, we can abuse Server-Side Request Forgery (SSRF) by forcing the server to issue an HTTP request to its own loopback interface. 

By manipulating the URL preview input to target:

`http://kestrel.thm/admin/notes`

The server processes the request internally. This causes the application to satisfy its own local IP validation, effectively bypassing the external access restrictions.

Since the request originates from the server itself, the application successfully reads and renders the `admin_notes.txt` file in the response. This file disclosed highly sensitive administrative credentials, specifically:

+ A valid SSH username
+ The corresponding SSH password

<img src="assets/admin-notes.png" width="700">


