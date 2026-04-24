# Bandit Level 1 → Level 2

## 🔗 Level Link
https://overthewire.org/wargames/bandit/bandit2.html

---

## 🧠 Level Goal
The password for the next level is stored in a file called `-` located in the home directory.

The goal is to read that file and use the password inside it to log in as `bandit2`.

---

## 📌 Given
- Current user: `bandit1`
- Target user for the next level: `bandit2`
- Server: `bandit.labs.overthewire.org`
- SSH port: `2220`
- Password location: a file named `-` in `/home/bandit1`

---

## 💻 Solution

First, connect as `bandit1`:

```bash
ssh bandit1@bandit.labs.overthewire.org -p 2220
```

After logging in, list the files in the home directory:

```bash
ls -la
```

You should see a file named exactly `-`:

```text
-rw-r----- 1 bandit2 bandit1 33 ... -
```

Read the file using an explicit path:

```bash
cat /home/bandit1/-
```

Alternative working method:

```bash
cat < -
```

The output is the password for `bandit2`.

Then log out:

```bash
exit
```

And connect as `bandit2`:

```bash
ssh bandit2@bandit.labs.overthewire.org -p 2220
```

Enter the password that was printed from the file.

---

## 📖 Explanation

This level is about a special filename: `-`.

In Linux and many Unix-like command-line tools, the dash character `-` often has a special meaning. It is commonly used to represent standard input (`stdin`) or standard output (`stdout`), depending on the command.

For example:

```bash
cat -
```

does **not** mean “read a file named `-`”. Instead, `cat` interprets `-` as “read from standard input”. As a result, the command waits for you to type input manually in the terminal.

That is why this does not solve the level:

```bash
cat -
```

The file exists, but the command is not treating `-` as a normal filename.

---

## ✅ Why `cat /home/bandit1/-` works

Command:

```bash
cat /home/bandit1/-
```

This works because `/home/bandit1/-` is an absolute file path.

Breakdown:

- `/home/bandit1/` — the home directory of user `bandit1`
- `-` — the actual filename inside that directory

When the dash is part of a full path, it is no longer interpreted as the special stdin marker. The command clearly receives a path to a file, so `cat` reads the file normally.

This is one of the safest and clearest ways to solve this level.

---

## ✅ Why `cat < -` works

Command:

```bash
cat < -
```

This works because of shell input redirection.

The `<` symbol tells the shell:

> Use the file after `<` as standard input for the command.

So the shell opens the file named `-` and passes its contents into `cat` through standard input.

In this command:

- `cat` is run without a filename argument
- `< -` tells the shell to read input from the file named `-`
- `cat` receives the file content through stdin and prints it

Important detail:

The shell handles the redirection before `cat` runs. Because of that, `cat` does not need to interpret `-` as a filename at all.

---

## ⚠️ Why `cat ./-` usually works

Another common solution is:

```bash
cat ./-
```

Here, `./-` means “the file named `-` in the current directory”.

Breakdown:

- `.` — current directory
- `/` — path separator
- `-` — filename

Because the dash is part of a path, `cat` should treat it as a regular file. In many Linux environments, this works correctly.

However, in your session, this variant did not work, while these two did:

```bash
cat /home/bandit1/-
cat < -
```

That is fine. The important point is that both working commands correctly avoid the ambiguity of plain `cat -`.

---

## ✅ Another valid method: `cat -- -`

A common general-purpose solution is:

```bash
cat -- -
```

The `--` argument means:

> Stop parsing options. Everything after this is a positional argument.

This is useful when a filename starts with `-`, because many commands treat values beginning with `-` as options.

In this command:

- `cat` is the command
- `--` ends option parsing
- `-` is then treated as a filename argument

This method is especially useful when working with filenames that look like command options.

---

## 🧩 Key Concept

The problem is not file permissions. The file is readable by the `bandit1` group:

```text
-rw-r----- 1 bandit2 bandit1 33 ... -
```

Meaning:

- Owner: `bandit2`
- Group: `bandit1`
- The group has read permission
- User `bandit1` can read the file

The real issue is the filename itself: `-` has special meaning in command-line tools.

---

## 📚 Commands Used

- `ssh` — connect to the remote Bandit server
- `ls -la` — list files, including hidden files and detailed permissions
- `cat` — print file contents
- `<` — redirect file contents into standard input
- `exit` — leave the current SSH session

---

## 🧠 What To Remember

- A filename can be almost any character, including `-`
- Plain `cat -` reads from standard input, not from a file named `-`
- Use a full path to remove ambiguity:

```bash
cat /home/bandit1/-
```

- Or use input redirection:

```bash
cat < -
```

- Or stop option parsing with `--`:

```bash
cat -- -
```

---

## 🧩 Result

The file named `-` was read successfully, and the password for `bandit2` was obtained.
