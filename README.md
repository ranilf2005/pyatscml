0) What you’ll end up with

GitLab CE on Docker (HTTP on 8929, SSH on 2222)

Dockerized GitLab Runner (Docker executor) that launches job containers

A sample repo with .gitlab-ci.yml that runs pyats run job …, uploads JUnit results & HTML logs to GitLab

Notes:

Docker’s Ubuntu docs currently cover 24.04/24.10; 25.04 works the same. If the apt repo doesn’t yet include 25.04, use the convenience script below (dev-friendly). 
Docker Documentation

We’ll use official GitLab Docker & Runner images and collect JUnit test reports in GitLab UI. 
GitLab Docs
+3
gitlab-docs-d6a9bb.gitlab.io
+3
GitLab Docs
+3

pyATS can emit xUnit/JUnit via --xunit, which GitLab will parse.


1) Install Docker Engine + Compose v2

Option A — official apt repo (recommended when available):

sudo apt update
sudo apt remove -y docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc 2>/dev/null || true
sudo apt install -y ca-certificates curl gnupg

# Docker repo key & source
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release; echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# run docker as your user (log out/in after this)
sudo usermod -aG docker $USER
docker --version
docker compose version


(Reference: Docker’s official Ubuntu guide & Compose plugin docs.) 
Docker Documentation
+1

Option B — convenience script (safe for labs/dev if 25.04 repo lags):

curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER


(Recommended by Docker for quick dev installs.) 
GitHub

2) Create folders & a Docker network
sudo mkdir -p /srv/gitlab/{config,logs,data}
sudo mkdir -p /srv/gitlab-runner/config
sudo chown -R 1000:1000 /srv/gitlab /srv/gitlab-runner
docker network create gitlab-net

3) docker-compose.yml for GitLab + Runner

Create docker-compose.yml in an empty folder (e.g., ~/gitlab-local):

version: "3.9"

services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    restart: always
    hostname: gitlab.local
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://YOUR_HOST_IP:8929'
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
    ports:
      - "8929:8929"   # HTTP UI
      - "2222:22"     # Git over SSH
    volumes:
      - /srv/gitlab/config:/etc/gitlab
      - /srv/gitlab/logs:/var/log/gitlab
      - /srv/gitlab/data:/var/opt/gitlab
    networks: [gitlab-net]
    shm_size: '256m'

  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    restart: always
    depends_on: [gitlab]
    volumes:
      - /srv/gitlab-runner/config:/etc/gitlab-runner
      - /var/run/docker.sock:/var/run/docker.sock
    networks: [gitlab-net]

networks:
  gitlab-net:
    external: true


Replace YOUR_HOST_IP with your Ubuntu box’s LAN IP (e.g., 192.168.30.200).
GitLab Omnibus config is documented in the Docker install & config pages. 
gitlab-docs-d6a9bb.gitlab.io
+1

Bring it up:

docker compose up -d
docker ps

4) First login to GitLab

Wait until GitLab finishes booting (CPU settles; docker logs -f gitlab shows “ready”).

Get the initial root password (valid until first reconfigure):

docker exec -it gitlab bash -lc "grep 'Password:' /etc/gitlab/initial_root_password || cat /etc/gitlab/initial_root_password"


Browse to http://YOUR_HOST_IP:8929 → login as root with that password → set a new one.

(Omnibus installs store the first password in /etc/gitlab/initial_root_password.) 
gitlab-docs-d6a9bb.gitlab.io

5) Register the GitLab Runner (Docker executor)

In GitLab UI: Admin > Runners (site-wide) or Project > Settings > CI/CD > Runners (project-scoped) → copy the registration token. Then:

# interactively register (one-time)
docker run --rm -it \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  --network gitlab-net \
  gitlab/gitlab-runner:latest register

# Answer prompts:
# GitLab URL:  http://gitlab:8929   (container-to-container via gitlab-net)
# Token:       <paste token>
# Description: pyATS Docker runner
# Tags:        pyats
# Executor:    docker
# Default image: ciscotestautomation/pyats:latest


The Runner container mounts the host Docker socket so it can start job containers (Docker executor). See GitLab’s official Docker Runner docs. 
GitLab Docs
+1

6) Create a sample pyATS project repo
Repo layout
pyats-demo/
├─ .gitlab-ci.yml
├─ testbed.yaml
├─ jobs/
│  └─ smoke_job.py
└─ tests/
   └─ test_ping_and_routes.py

testbed.yaml

(Uses CI/CD variables for credentials—add them later in step 7.)

devices:
  iosv-0:
    os: ios
    type: router
    connections:
      cli:
        protocol: ssh
        ip: 192.168.30.182
        port: 22
        username: ${PYATS_USER}
        password: ${PYATS_PASS}

  csr1000v-0:
    os: iosxe
    type: router
    connections:
      cli:
        protocol: ssh
        ip: 192.168.30.183
        port: 22
        username: ${PYATS_USER}
        password: ${PYATS_PASS}

jobs/smoke_job.py
from pyats.easypy import run

def main(runtime):
    run(testscript='tests/test_ping_and_routes.py',
        runtime=runtime)

tests/test_ping_and_routes.py

Minimal example to show the flow (adjust logic/targets for your lab).

from pyats import aetest

PING_TARGET = "100.1.1.1"   # change to a reachable IP in your lab

class CommonSetup(aetest.CommonSetup):
    @aetest.subsection
    def connect(self, testbed):
        testbed.connect(log_stdout=False)

class PingTest(aetest.Testcase):
    @aetest.test
    def ping_100_percent(self, testbed):
        for name, device in testbed.devices.items():
            out = device.execute(f"ping {PING_TARGET} repeat 5")
            if "Success rate is 100 percent" not in out:
                self.failed(f"{name} ping failed (<100%)")

class RoutesCompare(aetest.Testcase):
    @aetest.test
    def compare_static_routes(self, testbed):
        # Compare 'S' routes between both devices
        devs = list(testbed.devices.values())
        rts = []
        for d in devs:
            parsed = d.parse("show ip route")
            static = set()
            for proto, routes in parsed.get('vrf', {}).get('default', {}).get('routes', {}).items():
                # pyATS/Genie parsed dict has 'source_protocol' per route
                if routes.get('source_protocol') == 'static':
                    static.add(proto)
            rts.append(static)
        if len(rts) >= 2 and rts[0] != rts[1]:
            self.failed(f"Static routes differ: {rts[0]} vs {rts[1]}")

class CommonCleanup(aetest.CommonCleanup):
    @aetest.subsection
    def disconnect(self, testbed):
        for dev in testbed.devices.values():
            if dev.is_connected():
                dev.disconnect()


You’ll run this through pyats run job …. pyATS can emit xUnit/JUnit via --xunit, which GitLab consumes as Unit test reports. 
devnet-pubhub-site.s3.amazonaws.com
+1

7) .gitlab-ci.yml to run pyATS & publish reports
stages: [test]

variables:
  # pass device creds via project/group CI/CD > Variables (masked/protected)
  PYATS_USER: "$PYATS_USER"
  PYATS_PASS: "$PYATS_PASS"

pyats_tests:
  stage: test
  image: ciscotestautomation/pyats:latest
  tags: ["pyats"]
  script:
    - pyats version check
    - mkdir -p reports logs
    - pyats run job jobs/smoke_job.py -t testbed.yaml --xunit reports/ --html-logs logs/
  artifacts:
    when: always
    paths:
      - logs/
      - archive/*.zip
      - reports/
    reports:
      junit:
        - reports/*.xml


Unit test reports appear in the pipeline/MR UI when you declare artifacts:reports:junit. 
GitLab Docs
+1

The job image uses the official pyATS Docker image. (You can also pip install "pyats[full]" into a Python base image if you prefer.) 
Docker Hub

8) Push, set variables, and run

In GitLab: create a blank project (e.g., pyats-demo).

Locally:

cd pyats-demo
git init
git remote add origin http://YOUR_HOST_IP:8929/root/pyats-demo.git
git add .
git commit -m "pyATS CI smoke test"
git push -u origin main


In Project > Settings > CI/CD > Variables, add:

PYATS_USER (masked)

PYATS_PASS (masked)

Open Build > Pipelines and watch the job go green; open Unit test report panel & artifacts for HTML logs.

9) Runner networking & options (only if needed)

By default the Docker executor NATs outbound traffic, which is fine for pyATS initiating SSH to your lab subnets.

If you need special networking (e.g., host network), you can set network_mode in the Runner’s config.toml, but be aware of trade-offs and helper-container quirks. For most labs, don’t change this. 
GitLab Docs
+1

10) Handy admin commands
# logs & status
docker logs -f gitlab
docker logs -f gitlab-runner
docker exec -it gitlab-runner gitlab-runner verify

# change root password if needed
docker exec -it gitlab bash -lc "gitlab-rake 'gitlab:password:reset' "  # then pick 'root'


(Password reset via gitlab-rake is supported with Omnibus/GitLab in Docker.) 
Computer How To

11) Extras you can add later

Enable HTTPS & a real hostname via reverse proxy (Nginx/Traefik).

Use Docker-in-Docker or socket binding to build images inside CI (set privileged only if you must). 
GitLab Docs

Publish more rich logs or artifacts (e.g., --html-logs, archives).

Build your own pyATS image with pinned libs, or use Cisco’s pyats-image-builder. 
PyPI
