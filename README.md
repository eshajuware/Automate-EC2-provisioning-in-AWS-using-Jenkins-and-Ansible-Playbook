# Automate-EC2-provisioning-in-AWS-using-Jenkins-and-Ansible-Playbook


A record of everything you set up and fixed while building this project on AWS (24 Sep 2026). The pipeline creates a new EC2 instance with Ansible, and that instance runs the 2048 game in Docker.

Based on the MrCloudBook tutorial "Automate EC2 provisioning in AWS using Jenkins and Ansible Playbook" and the original repo `Aj7Ay/ANSIBLE`.

---

## 1. Jenkins Server (EC2)
- Region: **us-east-1 (N. Virginia)**, instance name `CI-CD`.
- Instance type: **c7i-flex.large** (2 vCPU, 4 GB) — the tutorial's t2.medium wasn't available in the free-tier list.
- OS: **Ubuntu 26.04 (Resolute Raccoon)** instead of the tutorial's 22.04.
- Key pair used: `zomato`.
- Security group: SSH (22), HTTP (80), HTTPS (443), plus **8080** for Jenkins.

## 2. Saved Command History (so it isn't lost again)
Problem: after a reconnect, `history` showed almost nothing because bash only writes history when a shell exits cleanly, and `ubuntu` and `root` each have their own history file.

Fix added to `~/.bashrc`:
```bash
shopt -s histappend
export HISTSIZE=10000
export HISTFILESIZE=20000
export HISTTIMEFORMAT="%F %T "
export PROMPT_COMMAND="history -a; $PROMPT_COMMAND"
```
Every command is now saved as it is typed, with a date and time. (Optional extras: `tmux` for sessions that survive a network drop, and `script -a ~/project-log.txt` to record output.)

## 3. Installed Java + Jenkins
The tutorial's `jenkins.sh` failed with `Malformed entry 1 in list file /etc/apt/sources.list.d/adoptium.list`. Three causes:
- The blog's `[` was copied as the HTML code `&#91;`, which apt can't read.
- The Adoptium repo has no `resolute` (26.04) section.
- The tutorial's 2023 Jenkins key had expired.

Clean script used (`vi jenkins.sh`, then `i`, paste, `Esc`, `:wq`):
```bash
#!/bin/bash
set -e

rm -f /etc/apt/sources.list.d/adoptium.list /etc/apt/sources.list.d/jenkins.list
rm -f /etc/apt/keyrings/adoptium.asc /usr/share/keyrings/jenkins-keyring.asc

apt update -y
apt install -y fontconfig openjdk-21-jdk
java --version

mkdir -p /etc/apt/keyrings
wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | tee /etc/apt/sources.list.d/jenkins.list > /dev/null

apt-get update -y
apt-get install -y jenkins
systemctl enable --now jenkins
systemctl status jenkins --no-pager
```
Then:
```bash
chmod +x jenkins.sh
sudo ./jenkins.sh
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Opened `http://<public-ip>:8080`, unlocked Jenkins with that password, and installed the suggested plugins.

## 4. Installed Trivy
Same `&#91;` problem in the tutorial's `trivy.sh`. Also used `generic` in the repo line instead of the Ubuntu codename, because `resolute` isn't published there.
```bash
#!/bin/bash
set -e

apt-get install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | tee /etc/apt/sources.list.d/trivy.list
apt-get update
apt-get install -y trivy
trivy --version
```
Result: **Trivy 0.74.0** installed.

## 5. Installed Ansible + AWS libraries
```bash
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
ansible --version
which ansible        # /usr/bin/ansible
```
Result: **ansible core 2.21.4** on Python 3.14.4. The Ansible PPA works on 26.04.

AWS SDK for Ansible:
```bash
sudo apt install -y python3-boto3 python3-botocore
ansible-galaxy collection install amazon.aws
```
Result: `boto3` and `botocore` **1.40.72**; the `amazon.aws` collection was already installed.

Errors you saw that are **expected and can be ignored**:
- `pip3 install boto boto3` → `externally-managed-environment` (PEP 668). Ubuntu blocks system-wide pip installs. Not needed, since boto3 came from apt. Don't use `--break-system-packages`.
- `python3-boto` → "no installation candidate". That's the old boto 2 library, and nothing in this project uses it.
- `python3-xyz` → "Unable to locate package". `xyz` was only a placeholder in an error message.

## 6. AWS Setup
- **IAM role** `CI-CD` with the `AmazonEC2FullAccess` policy, attached to the Jenkins instance (Actions → Security → Modify IAM role). This lets Ansible create EC2 instances.
- **AMI ID** for Ubuntu 26.04 in us-east-1, from Canonical's SSM parameter (run in CloudShell):
```bash
aws ssm get-parameters --region us-east-1 \
  --names /aws/service/canonical/ubuntu/server/26.04/stable/current/amd64/hvm/ebs-gp3/ami-id \
  --query 'Parameters[0].[Value]' --output text
```
Result: `ami-09b09d2491cd88154`
- **Key pair:** `zomato` (already existed in us-east-1).

## 7. GitHub Fork and Playbook Fixes
Forked `Aj7Ay/ANSIBLE` to **`https://github.com/eshajuware/ANSIBLE`** so Jenkins could pull your edited playbook.

Changes made to `ec2.yaml` from the tutorial version:

| Item | Tutorial | Yours |
|---|---|---|
| Region | ap-south-1 (Mumbai) | us-east-1 |
| Key pair | Mumbai | zomato |
| AMI | ami-0f5ee92e2d63afc18 | ami-09b09d2491cd88154 (Ubuntu 26.04) |
| Instance type | t2.micro | t3.small |
| Python interpreter | /usr/bin/python3.10 | /usr/bin/python3 |
| "Install dependencies" pip task | present | removed (fails on 26.04; boto3 already installed) |
| Port 8080 rule | open | removed (only the game on 3000 is needed) |
| `sudo` in user_data | used | removed (user data already runs as root) |

Also removed the **Git merge conflict markers** (`<<<<<<< HEAD`, `=======`, `>>>>>>>`) that were left in the forked file, since they made it invalid YAML.

Final `vars` section:
```yaml
  vars:
    ansible_python_interpreter: /usr/bin/python3
    keypair: zomato
    instance_type: t3.small
    image_id: ami-09b09d2491cd88154
    wait: yes
    group: webserver
    count: 1
    region: us-east-1
    security_group: ec2-security-group
    tag_name:
      Name: Aj-ec2
```

## 8. Jenkins Configuration
- Installed the **Ansible** plugin (Manage Jenkins → Plugins → Available plugins).
- Added the tool under **Manage Jenkins → Tools → Ansible installations**: name `ansible`, path **`/usr/bin`** (the *directory*, not `/usr/bin/ansible`).
- Created a **Pipeline** job and pasted the script into the **Script** box under Pipeline → Definition → "Pipeline script".

Final pipeline:
```groovy
pipeline {
    agent any
    tools {
        ansible 'ansible'
    }
    stages {
        stage('cleanws') {
            steps {
                cleanWs()
            }
        }
        stage('checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/eshajuware/ANSIBLE.git'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage('ansible provision') {
            steps {
                ansiblePlaybook playbook: 'ec2.yaml'
            }
        }
    }
}
```
Changes from the tutorial pipeline: the repo URL points to your fork, and the `pip install --upgrade requests==2.20.1` line was removed (it fails on Ubuntu 26.04's Python 3.14).

## 9. Result
- Build ran the four stages; Ansible created the security group and launched a new EC2 instance tagged **`Aj-ec2`**.
- The user-data script installed Docker and started the `sevenajay/2048` container on port **3000**.
- Opened `http://<new-instance-public-ip>:3000` (plain **http**, after waiting a few minutes for Docker to start) and the **2048 game loaded and works**.
- To play: click the board once, then use the arrow keys.

If the page hadn't loaded, the checks were: use `http://` not `https://`, wait for cloud-init to finish, confirm port 3000 in the instance's security group, and inspect the instance with `sudo cloud-init status`, `sudo docker ps` and `curl -I localhost:3000`.

---

## Still Open / Worth Following Up
- **Terminate the `Aj-ec2` instance** when done. Every pipeline run creates another instance (`count: 1`), so check the EC2 console before re-running.
- **Stop (not terminate) the Jenkins instance** to keep Jenkins jobs and history on the disk. The public IP changes after a restart.
- **Check the disk size** with `df -h`. During `apt install` the free space showed only about 3 GB, which is small for Jenkins, Trivy and Docker. If it's low, enlarge the EBS volume (Volumes → Modify Volume), then run `sudo growpart` and `sudo resize2fs`.
- **Add the `Jenkinsfile` to your GitHub fork** (a file named `Jenkinsfile`) plus a short README, so the whole project can be rebuilt.
- **Back up the server:** Actions → Image and templates → Create image (AMI).
- Optionally add `archiveArtifacts artifacts: 'trivyfs.txt', allowEmptyArchive: true` after the Trivy stage to keep the scan report with the build history.
