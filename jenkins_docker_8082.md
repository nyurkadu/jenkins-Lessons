# Lesson 22 Lab — Jenkins in Docker on port 8086

Goal: run a Jenkins server inside a Docker container and open it in the browser at **http://localhost:8086**.

Environment: Docker Desktop on Windows, commands run in **PowerShell**.

---

## Part 1 — Create the Docker environment

### Step 1.1 — Verify Docker is running

Start Docker Desktop, then check the engine responds:

```powershell
docker version
docker info --format "{{.ServerVersion}}"
```

You should see a client **and** a server version. If only the client prints, Docker Desktop is not fully started yet — wait for the whale icon to stop animating and retry.

### Step 1.2 — Create a network for Jenkins

A dedicated network keeps Jenkins isolated and lets future containers (agents, databases) reach it by name:

```powershell
docker network create jenkins
```

Check it exists:

```powershell
docker network ls | Select-String jenkins
```

### Step 1.3 — Create a volume for Jenkins data

Jenkins keeps everything (jobs, plugins, users, credentials) under `/var/jenkins_home`. A named volume makes that survive container restarts and removals:

```powershell
docker volume create jenkins_home
```

Check it exists:

```powershell
docker volume ls | Select-String jenkins_home
```

### Step 1.4 — Pull the Jenkins image

```powershell
docker pull jenkins/jenkins:lts-jdk17
```

`lts-jdk17` is the Long-Term-Support release bundled with Java 17. Confirm it is local:

```powershell
docker images jenkins/jenkins
```

---

## Part 2 — Install and run Jenkins on port 8086

### Step 2.1 — Start the container

```powershell
docker run -d `
  --name jenkins `
  --restart unless-stopped `
  --network jenkins `
  -p 8086:8080 `
  -p 50000:50000 `
  -v jenkins_home:/var/jenkins_home `
  jenkins/jenkins:lts-jdk17
```

What each flag does:

| Flag | Meaning |
|---|---|
| `-d` | run in the background |
| `--name jenkins` | container name, used by the commands below |
| `--restart unless-stopped` | come back automatically after Docker Desktop restarts |
| `--network jenkins` | join the network from Step 1.2 |
| `-p 8086:8080` | **host port 8086** → container port 8080 (Jenkins web UI) |
| `-p 50000:50000` | port for inbound build agents (optional but standard) |
| `-v jenkins_home:/var/jenkins_home` | persist data in the volume from Step 1.3 |

> Jenkins itself still listens on **8080 inside the container**. Only the host side is changed to 8086, so no Jenkins configuration is needed.

### Step 2.2 — Confirm the container is up

```powershell
docker ps --filter name=jenkins
```

The `PORTS` column must show `0.0.0.0:8086->8080/tcp`.

Follow the startup log until you see the initial password block:

```powershell
docker logs -f jenkins
```

Wait for lines like:

```
*************************************************************
Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

a1b2c3d4e5f6...
*************************************************************
```

Press `Ctrl+C` to stop following the log (the container keeps running).

### Step 2.3 — Get the initial admin password

If you missed it in the log, read it straight from the container:

```powershell
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Copy the value.

### Step 2.4 — Open Jenkins in the browser

Go to:

```
http://localhost:8086
```

1. **Unlock Jenkins** — paste the password from Step 2.3 and click **Continue**.
2. **Customize Jenkins** — click **Install suggested plugins** and wait for the progress screen to finish.
3. **Create First Admin User** — fill in username, password, full name and e-mail, then **Save and Continue**.
4. **Instance Configuration** — confirm the URL is `http://localhost:8086/` and click **Save and Finish**.
5. Click **Start using Jenkins**.

You now have a working Jenkins dashboard on port 8086.

### Step 2.5 — Verify from the command line

```powershell
Invoke-WebRequest http://localhost:8086/login -UseBasicParsing | Select-Object StatusCode
```

Expected: `StatusCode 200`.

---

## Part 3 — Install a plugin from the CLI (GitHub plugin)

Plugins can be installed from the browser (Manage Jenkins → Plugins), but the official image also ships `jenkins-plugin-cli`, which works from a shell inside the container.

### Step 3.1 — Go inside the container (PowerShell)

```powershell
docker exec -it jenkins bash
```

The prompt changes to `jenkins@<id>:/$`. Everything in the next step runs inside the container.

### Step 3.2 — Install the plugin (inside the container)

```bash
jenkins-plugin-cli --plugins github
```

The tool downloads `github` and all of its dependencies (`git`, `git-client`, `scm-api`, `token-macro`, ...). Confirm the file landed, then leave the container:

```bash
ls /var/jenkins_home/plugins/ | grep github
exit
```

### Step 3.3 — Restart Jenkins to load the plugin (PowerShell)

Jenkins only loads plugins at startup:

```powershell
docker restart jenkins
```

Wait for `docker logs jenkins` to show `Jenkins is fully up and running`, then check **Manage Jenkins → Plugins → Installed plugins** and search for *GitHub*.

> One-liner alternative without entering the container: `docker exec jenkins jenkins-plugin-cli --plugins github` followed by `docker restart jenkins`. Do not append Linux pipes such as `| tail` in PowerShell — they are not available there.

### Step 3.4 — Clone the lesson repository into the container

The image already ships `git`, so the lesson repo can live inside the persistent Jenkins home. From PowerShell:

```powershell
docker exec jenkins git clone https://github.com/nyurkadu/jenkins-Lessons /var/jenkins_home/jenkins-Lessons
```

Or from inside the container (`docker exec -it jenkins bash`):

```bash
cd /var/jenkins_home
git clone https://github.com/nyurkadu/jenkins-Lessons
```

Check it:

```powershell
docker exec jenkins git -C /var/jenkins_home/jenkins-Lessons remote -v
```

> If git prints `warning: You appear to have cloned an empty repository`, the GitHub repo has no commits yet — that is fine, the clone is set up and `git pull` will fetch content once it is pushed. Because `/var/jenkins_home` is the `jenkins_home` volume, the clone survives container restarts and removals.

---

## Part 4 — Day-to-day commands

| Task | Command |
|---|---|
| Stop Jenkins | `docker stop jenkins` |
| Start Jenkins again | `docker start jenkins` |
| View logs | `docker logs -f jenkins` |
| Shell inside the container | `docker exec -it jenkins bash` |
| Remove the container (data stays in the volume) | `docker rm -f jenkins` |
| Recreate after removal | rerun the `docker run` command from Step 2.1 |
| Remove **everything** including data | `docker rm -f jenkins; docker volume rm jenkins_home; docker network rm jenkins` |

---

## Part 5 — Resume in the next lesson (do NOT recreate)

The container, its data volume and the cloned repo are kept between lessons. Never rerun Step 2.1 or `docker rm` — that would lose nothing thanks to the volume, but it is unnecessary. Just start what already exists:

```powershell
docker ps -a --filter name=jenkins        # is it there? Up or Exited?
docker start jenkins                      # only needed if it shows Exited
docker logs -f jenkins                    # wait for "Jenkins is fully up and running", Ctrl+C
```

Then open http://localhost:8086 and log in with the admin user created in Step 2.4 (no initial password needed any more).

Pull the latest lesson code inside the container:

```powershell
docker exec jenkins git -C /var/jenkins_home/jenkins-Lessons pull
```

Because the container was started with `--restart unless-stopped`, it normally comes back on its own when Docker Desktop starts; the `docker start` above is just in case it was stopped by hand.

---

## Troubleshooting

**`Bind for 0.0.0.0:8086 failed: port is already allocated`** — another **container** already publishes 8086. (This lab originally used 8082, but `node2` from the Ansible lesson was already mapped `8082->80`, which is why it moved to 8086.) Find it, stop it, remove the half-created `jenkins` container, and rerun Step 2.1:

```powershell
docker ps --format "table {{.Names}}	{{.Ports}}" | Select-String 8086
docker stop <name>         # the container you found above
docker rm jenkins          # the failed run left it in "Created" state
```

Then repeat the `docker run` command from Step 2.1. Alternatively, keep the other container and pick a different host port by changing only the left side of `-p` (that is how this lab ended up on 8086).

**Port 8086 is held by a Windows process** (not a container) — find and stop it, or pick another host port and change only the left side of `-p`:

```powershell
netstat -ano | Select-String ":8086"
```

**Browser shows "connection refused"** — Jenkins takes 30–60 seconds to boot the first time. Run `docker logs jenkins` and wait for `Jenkins is fully up and running`.

**Password file not found** — the setup wizard has already been completed on this volume. Log in with the admin user you created, or wipe the volume (last row of the table above) to start over.

**Container keeps restarting** — check `docker logs jenkins`. The most common cause is a permissions problem on a bind-mounted host folder; that is why this lab uses a named volume instead.
