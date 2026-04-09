# 📘 Hack The Box: Dancing

> 🔗 https://app.hackthebox.com/machines/Dancing

---

## 🎯 Goal

Access the SMB share and retrieve the flag.

---

## 🧠 Key Idea

This lab demonstrates a classic SMB misconfiguration:
- the SMB service is exposed over the network;
- one share allows access without valid credentials;
- this lets us enumerate directories and download files.

The core lesson is not exploitation in the memory-corruption sense, but **enumeration and abuse of weak access control**.

---

## 🔍 Walkthrough

### Step 1 — Scan the target

```bash
nmap -sV <TARGET_IP>
```

### Why this works

`nmap` sends network probes to the target and identifies:
- open ports;
- services bound to those ports;
- sometimes service versions.

The `-sV` flag enables service/version detection. This matters because knowing that a port is open is not enough; we need to understand **what service is actually running there**.

### What we learn

The result shows:
- `445/tcp open`
- service `microsoft-ds`

### Why this matters

Port **445** is the standard TCP port for SMB.  
If it is open, the host is exposing a Windows/Samba file-sharing service, which makes SMB enumeration the next logical step.

---

### Step 2 — Enumerate SMB shares

```bash
smbclient -L //<TARGET_IP> -N
```

### Why this works

`smbclient` is a client utility for interacting with SMB resources.

Command breakdown:
- `-L` — list available SMB shares;
- `//<TARGET_IP>` — specify the remote SMB host;
- `-N` — suppress the password prompt and attempt a null/anonymous session.

If the server allows guest or anonymous access, it will return the share list even without valid credentials.

### What we get

We usually see:
- `ADMIN$`
- `C$`
- `IPC$`
- `WorkShares`

### Why this matters

The first three are standard administrative/system shares:
- `ADMIN$` — administrative access share;
- `C$` — hidden share for the system drive;
- `IPC$` — inter-process communication share.

These normally require elevated permissions.  
**`WorkShares`**, however, looks like a custom user-created share, and custom shares are much more likely to be misconfigured.

---

### Step 3 — Connect to the accessible share

```bash
smbclient //<TARGET_IP>/WorkShares -N
```

### Why this works

Now we are no longer just listing shares — we are connecting directly to the `WorkShares` share.

If anonymous access is allowed on that share, the server grants access and gives us an interactive SMB prompt:

```text
smb: \>
```

### Why we choose WorkShares

Because:
- `ADMIN$` and `C$` are typically restricted;
- `IPC$` is not intended for normal file browsing;
- `WorkShares` is the only realistic target for useful user data.

---

### Step 4 — List the contents

```bash
ls
```

### Why this works

After connecting successfully, we are effectively interacting with the remote share like a filesystem.

The `ls` command lists the contents of the current remote directory.  
Typical output shows directories such as:

- `Amy.J`
- `James.P`

### Why this matters

This strongly suggests the share is being used as a real working file repository for users.  
That means it may contain:
- notes;
- configuration files;
- credentials;
- flags or other sensitive data.

This is why enumeration should not stop at share discovery — you must also **walk the directory structure**.

---

### Step 5 — Move through directories and download files

```bash
cd Amy.J
get worknotes.txt
cd ..
cd James.P
get flag.txt
```

### Why this works

The SMB interactive shell supports commands similar to a regular shell:

- `cd` — change directory;
- `get` — download a remote file to the local machine;
- `cd ..` — move back up one level.

### Why we inspect `Amy.J` first

Because in a real assessment, you do not only collect the final objective — you also collect **contextual artifacts**.  
A file like `worknotes.txt` may contain:
- hints about additional services;
- usernames;
- operational notes;
- evidence of weak security practices.

Even if it is not required for the current flag, this is the right methodology: **collect interesting artifacts, not just the end target**.

### Why we then inspect `James.P`

Because once one user directory exists, checking the others is the natural next step.  
Inside, we find `flag.txt`, which is the target artifact for this lab.

---

### Step 6 — Read the flag locally

```bash
cat flag.txt
```

### Why this works

The `get flag.txt` command already downloaded the file to the local machine.  
After that, `cat` simply prints the contents of the local file to the terminal.


## 📋 Task Answers

<details>
<summary>Show answers</summary>

1. **Server Message Block**  
2. **445**  
3. **microsoft-ds**  
4. **-L**  
5. **4**  
6. **WorkShares**  
7. **get**

</details>

---

## 🔐 Flag

<details>
<summary>Show flag</summary>

`5f61c10dffbc77a704d76016a22f1664`

</details>

---

## 💥 Vulnerability

This lab is not about memory corruption or code execution.  
It is about an **access control misconfiguration**:

- SMB is exposed over the network;
- a custom share permits access without proper authentication;
- users store sensitive files inside that exposed share.

This is a realistic internal pentest scenario: a service that looks ordinary becomes a data exposure vector because of weak permissions.

---

## 🛡️ Recommendations

- disable anonymous/guest access on SMB shares;
- restrict access to authorized users and groups only;
- avoid storing sensitive data in broadly accessible shares;
- audit SMB shares and permissions regularly;
- log and monitor access to network file resources.

---

## 🛠️ Checklist

- [ ] Scan the target  
- [ ] Confirm SMB on 445/tcp  
- [ ] Enumerate available shares  
- [ ] Identify a likely misconfigured custom share  
- [ ] Connect without a password  
- [ ] Enumerate directories inside the share  
- [ ] Download interesting files  
- [ ] Retrieve the flag
