# Lab Setup Challenges & Troubleshooting Log

This document captures the real-world issues I ran into while setting up and running
the `week1-labs` Docker Compose environment, and how I diagnosed and resolved each one.
The goal is to show the actual process of getting from a broken environment to a
working one — not just the final clean commands.

---

## Challenge 1: `docker compose up -d` — "no configuration file provided: not found"

**What happened:**
Running `docker compose up -d` from my project folder on Windows returned:
```
no configuration file provided: not found
```

**Diagnosis:**
Docker Compose looks for a file named `docker-compose.yml`, `docker-compose.yaml`,
`compose.yml`, or `compose.yaml` in the current directory. This error means it
couldn't find one — either it wasn't there, the filename didn't match exactly, or I
was in the wrong directory.

**Fix:**
Confirmed the correct folder and filename, and re-ran the command once I was `cd`'d
into the directory that actually contained `docker-compose.yml`.

---

## Challenge 2: "failed to connect to the docker API" (Windows / Docker Desktop)

**What happened:**
After the compose file was found, the next error was:
```
unable to get image 'redis:7': failed to connect to the docker API at
npipe:////./pipe/dockerDesktopLinuxEngine; ... The system cannot find the file specified.
```

**Diagnosis:**
The Docker CLI was installed and working, but the Docker Desktop background engine
wasn't running. The CLI can exist independently of the engine actually being started.

**Fix:**
This pointed to a deeper decision: rather than keep depending on Docker Desktop and
WSL on Windows, I moved the whole workflow to a native Kali Linux environment, where
Docker runs as a direct Linux service — no virtualization layer or separate desktop
app required.

---

## Challenge 3: Setting up Docker natively on Kali Linux

**What happened:**
Needed Docker Engine and the Compose plugin installed directly on Kali, without
Docker Desktop.

**Fix:**
- Installed Docker Engine via Docker's official APT repository (using the Debian
  `bookworm` package set, since Kali is Debian-based).
- Installed the `docker-compose-plugin` so `docker compose` (space, not hyphen)
  worked as a native subcommand.
- Enabled and started the Docker service with `systemctl enable --now docker`.
- Added my user to the `docker` group so I didn't need `sudo` for every command.

---

## Challenge 4: `apt update` blocked by a broken Trivy repo signature

**What happened:**
Running `sudo apt update` failed partway through:
```
Err:2 https://aquasecurity.github.io/trivy-repo/deb generic InRelease
Sub-process /usr/bin/sqv returned an error code (1) ... Missing key ...
The repository '...' is not signed.
```

**Diagnosis:**
A third-party APT source (Trivy, a security scanner) had a GPG signing key that
wasn't properly registered, so APT refused to trust or update from it — by design,
to prevent tampered packages from being installed silently.

**Fix:**
Since Trivy wasn't needed for this specific lab, I removed its repo source file
(`/etc/apt/sources.list.d/trivy.list`) to unblock `apt update` immediately, rather
than spending time re-importing its key for a tool I wasn't using yet. This is a
good example of triaging: fix what's blocking the actual task first, revisit
optional tooling later.

---

## Challenge 5: GitHub authentication failures when pushing

**What happened:**
Several rounds of `git push` failures, each with a different underlying cause:

1. **`Password authentication is not supported for Git operations`**
   — I had entered my email address at the Username prompt, and/or tried to
   authenticate with my actual GitHub account password. GitHub disabled
   password-based Git authentication years ago; only a Personal Access Token (PAT)
   or SSH key works now.

2. **Malformed username** — At one point I pasted a full profile URL
   (`https://github.com/Ene-Otalu`) into the Username prompt instead of just the
   username itself (`Ene-Otalu`). Git took the whole string literally, and GitHub
   rejected it as invalid.

3. **`remote: Repository not found`** — Even after fixing the username and using a
   token correctly, the push still failed because the remote URL pointed to a
   repository (`otaluene/week1-labs`) that didn't actually exist under my real
   GitHub account. My username and the email-derived name I assumed weren't the same
   thing.

**Fix:**
- Generated a Personal Access Token from GitHub (Settings → Developer settings →
  Personal access tokens) with `repo` scope, and used that as the password.
- Corrected the Username prompt to use just the plain GitHub handle (`Ene-Otalu`),
  no URL, no email.
- Verified my real username by checking my GitHub profile directly rather than
  assuming it matched my email address.
- Created a new, empty repository under my actual account and pointed my local
  remote at it with `git remote set-url origin https://github.com/Ene-Otalu/<repo>.git`.
- Re-ran `git push origin main`, which succeeded once the username, token, and
  remote URL all lined up correctly.

---

## Key Takeaways

- **Read the actual error text carefully** — each failure here had a specific,
  literal explanation in its output (missing file, missing engine, missing key,
  wrong credential type, wrong repo). None of them needed guesswork once I slowed
  down and read what the tool was actually telling me.
- **Authentication errors are rarely about "wrong password"** — they're usually
  about using the *wrong kind* of credential (password vs. token) or the *wrong
  identifier* (email vs. username vs. URL).
- **Moving to native Kali eliminated an entire class of Windows/WSL/Docker Desktop
  virtualization issues** by running Docker as a plain Linux service instead of
  through an extra abstraction layer.
- **Not every broken thing needs fixing immediately** — removing the unrelated
  Trivy repo instead of debugging its key let me stay focused on the actual goal.
