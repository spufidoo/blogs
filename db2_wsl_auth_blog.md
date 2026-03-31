# The bit that broke Db2
## The One Linux Permission That Silently Breaks Remote Authentication

*Db2 runs flawlessly inside WSL. Every remote client fails. Here's why — and the two commands that fix it.*

---
### The Beginning

Recently, I installed Db2 (LUW) 12.1.4 on Ubuntu 24.04 running under WSL (Windows Subsystem for Linux) on my Windows laptop. All went well. I could access it via the db2cli and via home-grown Python code (no authentication necessary, as I was connecting locally). I was happy. Slightly disappointed that IBM had dropped support for the DB2Connect extension for Visual Studio Code. Imagine my delight when I heard that IBM were replacing it with the IBM Db2 Developer Extension! I tried it out immediately. And was immediately disappointed. Not with the extension itself, but the fact that I couldn’t get it to talk to Db2 in the first place. This is my story.

---
### The Middle

I wouldn’t have guessed that I’d fall at the first hurdle - Trying to register my local Db2 instance! It returned:

```
Connection test failed
Database connection failed. Please verify connection parameters. ERRORCODE=-4499, SQLSTATE=08001
```
My username was correct, my password was correct, my hostname was correct. Hmmm… What was the port number ? 50000? No. 25000? No. OK. Check the port number.
```bash
db2 get dbm cfg | grep -i svcename
db2set -all | grep -i comm
```
…Nothing. Aha! Let’s fix that.
```bash
db2set DB2COMM=TCPIP
db2 update dbm cfg using SVCENAME 25000
db2stop
db2start
```
Confirm it is listening:
```bash
ss -lntp | grep 25000
LISTEN 0      4096              0.0.0.0:25000      0.0.0.0:*    users:(("db2sysc",pid=13958,fd=26))
```
Yay! Now check that the dbm authentication is correct, with ``db2 get dbm cfg | grep -i authentication``
```
 Server Connection Authentication      (SRVCON_AUTH) = NOT_SPECIFIED
 Database manager authentication    (AUTHENTICATION) = SERVER
 Alternate authentication       (ALTERNATE_AUTH_ENC) = NOT_SPECIFIED
 Trusted client authentication      (TRUST_CLNTAUTH) = CLIENT
```
Let’s try connecting again… Nope. Same error. After this, I kind of gave up, having reached the end of my LUW sysadm knowledge. Over to you AI (Another Idiot)… After much to-ing and fro-ing, and many false recommendations (Hallucinations? Lies?) from Copilot, I ended up creating another WSL Linux user, ``db2user`` – which, we will find, was unnecessary.
```bash
sudo adduser db2user
sudo passwd db2user
db2 connect to SAMPLE 
db2 grant connect on database to user db2user
```
I could connect to my instance nicely, but could I connect to my instance from Windows itself? So I tried it with the PowerShell command, ``Test-NetConnection localhost -Port 25000``
```
WARNING: TCP connect to (::1 : 25000) failed
ComputerName     : localhost
RemoteAddress    : 127.0.0.1
RemotePort       : 25000
InterfaceAlias   : Loopback Pseudo-Interface 1
SourceAddress    : 127.0.0.1
TcpTestSucceeded : True
```
Yay! Windows recognised the open port, so could I connect to Db2? First of all, I catalogued the WSL node, then catalogued the SAMPLE database:
```bat
❯ db2 catalog tcpip node WSL remote localhost server 25000
DB20000I  The CATALOG TCPIP NODE command completed successfully.
> db2 catalog db SAMPLE at node WSL
DB20000I  The CATALOG DATABASE command completed successfully.
```
Then, the moment of truth...
```bat
❯ db2 connect to SAMPLE
SQL30082N  Security processing failed with reason "3" ("PASSWORD MISSING").
SQLSTATE=08001
❯ db2 connect to SAMPLE user db2user using password
SQL1639N  The database server was unable to perform authentication
because security-related database manager files on the server do not
have the required operating system permissions.  SQLSTATE=08001
```
This seemed significant, so I copied and pasted it into Copilot, who responded with "BINGO! That was it!" On Linux (including WSL), Db2 performs password authentication via setuid helper binaries, most importantly:

``~/sqllib/security/db2ckpw``

This binary must be:
- owned by the instance owner (or root)
- group-owned by db2iadm1 (or the instance group, or root)
- setuid-root
- executable

If any of that is wrong, Db2 cannot authenticate remote users → SQL1639N.

So I set the correct ownership:
```bash
sudo chown root:root ~/sqllib/security/db2ckpw
```
I set the correct permissions:
```bash
sudo chmod 4755 ~/sqllib/security/db2ckpw
```
Checked the output:
```bash
ls -l ~/sqllib/security/db2ckpw
-rwsr-xr-x 1 root root 3033512 Mar 12 17:55
```
Restarted Db2, and tried again…
```bat
❯ db2 connect to SAMPLE user db2user using password
   Database Connection Information
 Database server        = DB2/LINUXX8664 12.1.4.0
 SQL authorization ID   = DB2USER
 Local database alias   = SAMPLE
```

Yay! 🎉Result. It worked! I could connect to my WSL Db2 instance from Windows!

I then tried registering the database using the new IBM Db2 Developer Extension in VS Code - Perfect!

<img width="419" height="497" alt="image" src="https://github.com/user-attachments/assets/4f0ee09f-6c66-4243-a61b-f72b25822ab5" />

So far, I am suitably impressed with the Extension. I expect another blog post will ensue.

---

### The End

#### My Environment
- IBM Db2 12.1 LUW on Ubuntu 22.04 under WSL2
- Db2 instance running, TCP/IP enabled, listener confirmed active
- Default WSL user (matching my Windows username) — coincidentally (or not), the instance owner
- Clients: Db2 CLP, VS Code IBM Db2 Developer Extension

#### What Also Mattered

Before finding the actual cause, I had to find out the hard way that Db2 didn't automatically install TCPIP connectivity:

- **`DB2COMM=TCPIP`** — was already set, but
- **`SVCENAME`** — port was not listening; starting it and issuing `ss -lntp | grep 25000` confirmed it
- **`AUTHENTICATION=SERVER`** — correct for Db2 12.1
- **TRUST settings** — `TRUST_ALLCLNTS=YES` is enforced when `AUTHENTICATION=SERVER`; you can't usefully change this
- **Windows firewall / WSL networking** — `Test-NetConnection` succeeded
- **VS Code configuration** — not the issue
- **Creating a separate `db2user` OS account** — unnecessary

If Windows can reach the port and Db2 is working locally, none of those are your problem.


#### What *Really* Mattered: `db2ckpw`

On Linux, Db2 doesn't validate remote passwords itself. It delegates that work to a privileged helper binary:

```
~/sqllib/security/db2ckpw
```

When a remote client sends credentials, Db2 forks this binary to perform PAM authentication and check against `/etc/shadow`. This is why remote and local auth are handled differently — local CLI commands run in your session context; remote connections have no such privilege, so they need the helper.

For `db2ckpw` to do its job, Linux requires it to be:

- **Owned by `root`**
- **Setuid-root** (the `s` bit in `-rwsr-xr-x`)
- **Executable**
- **On a native Linux filesystem** (not `/mnt/c` — the NTFS mount drops setuid entirely)

The setuid bit is the critical part. When Linux executes a setuid-root binary, the kernel temporarily elevates the process to run as root, regardless of who invoked it. That's what gives `db2ckpw` the access it needs to read `/etc/shadow`. Without it, the binary runs as the Db2 instance user — which doesn't have that access — and authentication fails.

When the bit is missing or ownership is wrong, this is what you get:

```
SQL1639N  The database server was unable to perform authentication
because security-related database manager files on the server do not
have the required operating system permissions.  SQLSTATE=08001
```

`SQL1639N` is actually a useful error once you know what it's pointing at. The cryptic `ERRORCODE=-4499` you'll see from JDBC is the same underlying failure, just reported at the driver level.


#### Why WSL Is Especially Prone to This

WSL fully supports Linux setuid semantics — but it doesn't protect them from you. Any of the following can silently strip the bit:

- Installing Db2 without root (fairly common in WSL environments)
- Running `chmod -R` on your home directory
- Restoring a home directory from a backup or tar archive
- Migrating between WSL distributions
- Any file copy operation that doesn't explicitly preserve special bits

What makes this particularly hard to catch: local connections keep working regardless. The setuid bit is only exercised during remote authentication. You can run Db2 locally for weeks without noticing it's broken.


---
## The Fix

Inside WSL:

```bash
sudo chown root:root ~/sqllib/security/db2ckpw
sudo chmod 4755 ~/sqllib/security/db2ckpw
```

Verify it took:

```bash
ls -l ~/sqllib/security/db2ckpw
```

You should see:

```
-rwsr-xr-x 1 root root ... db2ckpw
```

The `s` where `x` would normally be in the owner field is the setuid bit. That's what you're looking for.

Restart Db2:

```bash
db2stop
db2start
```

Then from Windows (assuming that your node and database are already catalogued):

```bat
db2 connect to <DBNAME> user <your_wsl_username> using <password>
```

---

## Quick Checklist

Before touching anything else, verify these five things:

| Check | Command / verification |
|---|---|
| TCP/IP listener active | `ss -lntp \| grep 25000` |
| Not connecting as instance owner | `db2 get instance` — compare with your login |
| Login user has a Linux password | `sudo passwd <username>` |
| WSL node catalogued on Windows side | `db2 list node directory` |
| Database catalogued on Windows side | `db2 list database directory` |
| `db2ckpw` is setuid-root | `ls -l ~/sqllib/security/db2ckpw` — look for `-rwsr-xr-x` |

If the sixth row isn't right, nothing else matters.

---

## Takeaway

WSL is a genuinely excellent environment for running IBM Db2. It behaves like Linux because it is Linux. The flip side is that it inherits Linux's security model — including the expectation that certain privileged binaries are protected correctly.

`db2ckpw` is the one file that makes remote authentication work. If its permissions are wrong, Db2 will give you nothing useful beyond `SQL1639N` — and that error doesn't exactly shout "check your setuid bits."

Now you know where to look first.
