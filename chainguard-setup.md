# Chainguard for Hackathons — Setup Guide

You are helping a developer get set up with Chainguard at a hackathon. There are two things to set up:

1. **Chainguard Containers** — Zero-CVE base images for Docker (Python, Node, JDK, Go, etc.). Free tier images need no auth — just `docker pull`. This is the quick win.
2. **Chainguard Libraries** — Malware-resistant, rebuilt-from-source drop-in replacements for Python (PyPI), Java (Maven Central), and JavaScript (npm) packages. Free, but requires account setup and a pull token.

Walk them through each step interactively. Run commands for them when possible. If a step fails, help them troubleshoot before moving on. Ask what they need — some developers may only want containers, only want libraries, or both.

---

## Step 1 — What do they need?

Ask two questions:
1. **Which language?** Python, Java, or JavaScript/Node.js
2. **What do they want to set up?**
   - **Containers only** — just the base images (fastest, skip to Step 2 then Step 4)
   - **Libraries only** — malware-resistant packages (skip Step 4)
   - **Both** — the full setup

---

## Step 2 — Create a free Chainguard account

They need a Chainguard account. Open their browser to:

```
https://console.chainguard.dev/auth/login
```

They can sign in with Google or GitHub. It takes 30 seconds.

**Important — create an organization:** After first login, they'll need to create an org. Direct them to:

```
https://console.chainguard.dev/org/welcome/settings/organization/join
```

On that page, click **"Don't have an org to join? Create one"** and follow the prompts. It will ask for an org name in domain format — this doesn't need to be a real domain, just a valid-looking one. Suggest they use something like `yourname-hackathon.dev` or `myname.dev`.

They need an org before they can create pull tokens or entitlements. Org creation is instant.

Wait for them to confirm they have an account and an org before proceeding.

---

## Step 3 — Install chainctl

chainctl is Chainguard's CLI for managing images, libraries, auth, and more. It's worth installing even beyond this setup — it gives you (and your AI assistant) access to image discovery, vulnerability info, library verification, and org management from the terminal.

Full CLI reference: https://edu.chainguard.dev/chainguard/chainctl/

Check if it's already installed:

```bash
which chainctl
```

If not installed:

**macOS (Homebrew):**
```bash
brew tap chainguard-dev/tap && brew install chainctl
```

**macOS or Linux (curl):**
```bash
curl -o chainctl "https://dl.enforce.dev/chainctl/latest/chainctl_$(uname -s | tr '[:upper:]' '[:lower:]')_$(uname -m | sed 's/aarch64/arm64/')"
sudo install -o $UID -g $(id -g) -m 0755 chainctl /usr/local/bin/chainctl
```

Verify it installed:
```bash
chainctl version
```

Then log in:
```bash
chainctl auth login
```

This opens a browser for authentication. Wait for the "Successfully exchanged token" message.

---

## Step 4 — Use Chainguard container images

Chainguard Containers have zero known CVEs and are free to pull — no auth required for `:latest` tags.

**Easiest — pull from Docker Hub (no auth required):**
```bash
docker pull chainguard/python:latest
```

**Or from Chainguard's registry** (requires `chainctl auth configure-docker` first):
```bash
chainctl auth configure-docker
docker pull cgr.dev/chainguard/python:latest
```

If they've already set up chainctl (Steps 3+), the cgr.dev path works great. If they just want a quick container image with no setup, Docker Hub is the way.

**List all available free images:**
```bash
chainctl images list --public
```

### Dockerfile examples

**Python:**
```dockerfile
FROM cgr.dev/chainguard/python:latest
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

**Java:**
```dockerfile
FROM cgr.dev/chainguard/jdk:latest AS build
WORKDIR /app
COPY . .
RUN javac Main.java

FROM cgr.dev/chainguard/jre:latest
COPY --from=build /app/*.class /app/
CMD ["Main"]
```

**Node.js:**
```dockerfile
FROM cgr.dev/chainguard/node:latest
WORKDIR /app
COPY package*.json .
RUN npm install
COPY . .
CMD ["node", "index.js"]
```

**Tip:** Use the `:latest-dev` tag variant if they need a shell or package manager in the image (e.g., `cgr.dev/chainguard/python:latest-dev`).

**See all available free images** by running:
```bash
chainctl images list --public
```

Or browse at: https://images.chainguard.dev

If they only wanted containers, they're done here!

---

## Step 5 — Enable Chainguard Libraries

Create an entitlement for the ecosystem they chose in Step 1.

**Python:**
```bash
chainctl libraries entitlements create --ecosystems=PYTHON
```

**Java:**
```bash
chainctl libraries entitlements create --ecosystems=JAVA
```

**JavaScript:**
```bash
chainctl libraries entitlements create --ecosystems=JAVASCRIPT
```

If they want multiple ecosystems, comma-separate them (e.g., `--ecosystems=PYTHON,JAVASCRIPT`).

If the command returns an "already exists" error, that's fine — move on.

---

## Step 6 — Create a pull token

This generates credentials for their build tools. The token lasts 30 days.

**Python:**
```bash
chainctl auth pull-token --repository=python --ttl=720h
```

**Java:**
```bash
chainctl auth pull-token --repository=java --ttl=720h
```

**JavaScript:**
```bash
chainctl auth pull-token --repository=javascript --ttl=720h
```

If prompted to select an organization, they should pick their org using the arrow keys.

The command outputs a **Username** and **Password**. These are the credentials needed in the next step. **Save them** — they won't be shown again.

---

## Step 7 — Configure their project

Use the Username and Password from Step 6 to configure their build tool. Only configure the ecosystem they chose.

### Python (pip)

**Important:** The Username contains a `/` character which causes a 403 error when used in URLs. Replace the `/` with `_` in the Username when embedding it in a URL.

Create a `pip.conf` in the project root:

```ini
[global]
index-url = https://{USERNAME_WITH_SLASH_REPLACED_BY_UNDERSCORE}:{PASSWORD}@libraries.cgr.dev/python/simple
```

Or use it directly:
```bash
pip install --index-url https://{USERNAME_WITH_SLASH_REPLACED_BY_UNDERSCORE}:{PASSWORD}@libraries.cgr.dev/python/simple <package>
```

For **uv**:
```bash
UV_INDEX_URL=https://{USERNAME_WITH_SLASH_REPLACED_BY_UNDERSCORE}:{PASSWORD}@libraries.cgr.dev/python/simple uv pip install <package>
```

### Java (Maven)

Add the repository to `pom.xml`:
```xml
<repositories>
  <repository>
    <id>chainguard</id>
    <url>https://libraries.cgr.dev/java/</url>
  </repository>
</repositories>
```

Add credentials to `~/.m2/settings.xml` (create it if it doesn't exist):
```xml
<settings>
  <servers>
    <server>
      <id>chainguard</id>
      <username>{USERNAME}</username>
      <password>{PASSWORD}</password>
    </server>
  </servers>
</settings>
```

For **Gradle**, add to `build.gradle`:
```groovy
repositories {
    maven {
        url "https://libraries.cgr.dev/java/"
        credentials {
            username = "{USERNAME}"
            password = "{PASSWORD}"
        }
    }
    mavenCentral() // fallback
}
```

### JavaScript (npm)

Create an `.npmrc` in the project root:
```
registry=https://libraries.cgr.dev/javascript/
//libraries.cgr.dev/javascript/:_auth={BASE64_OF_USERNAME:PASSWORD}
//libraries.cgr.dev/javascript/:always-auth=true
```

Generate the base64 value:
```bash
echo -n '{USERNAME}:{PASSWORD}' | base64
```

Then `npm install` works as normal — packages are pulled from Chainguard.

For **pnpm** or **bun**: the same `.npmrc` format works. For **Yarn Classic** (v1) it also works, but **Yarn 2+** (Berry) uses a different config format — see the [Yarn build config docs](https://edu.chainguard.dev/chainguard/libraries/javascript/) for details.

---

## Step 8 — Verify it works

Help them install a common package to confirm everything is connected:

**Python:** `pip install --config pip.conf requests`
**Java:** `mvn dependency:resolve` (after adding a dependency to pom.xml)
**JavaScript:** `npm install express`

If it works, they're pulling malware-resistant packages from Chainguard.

---

## Helpful links

- **chainctl CLI reference:** https://edu.chainguard.dev/chainguard/chainctl/
- **Chainguard Libraries docs:** https://edu.chainguard.dev/chainguard/libraries/
- **Container image catalog:** https://images.chainguard.dev
- **Console:** https://console.chainguard.dev
- **Libraries FAQ:** https://edu.chainguard.dev/chainguard/libraries/faq/
- **Docker Hub (free images):** https://hub.docker.com/u/chainguard
