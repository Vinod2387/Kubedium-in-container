## How to run Kubedium master + Worker, Jenkins with Docker CLI installed three container:

```bash
    docker-compose up -d

```

## Step 1: Go inside Jenkins container: 

```bash
docker exec -u root -it jenkins bash
```

## Step 2: Install Docker CLI in Jenkins container:

```bash
apt-get update
apt-get install -y docker.io
docker –version
```

## Add Jenkins in docker group:
```bash
groupadd docker || true
usermod -aG docker jenkins
```

## Verify group membership: 
```bash
getent group docker
```

Result should be:  docker:x:999:jenkins

## Restart container or re-login the user:
```bash
docker restart jenkins
```
## To Access Jenkin: 
```bash
http://localhost:8080
```
## Initial Admin Password: 
```bash
exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```
## To fix Docker permission inside the Jenkins container:
```bash
docker exec -u root -it jenkins bash
ls -l /var/run/docker.sock
id jenkins
usermod -aG docker jenkins

groupmod -g 124 docker
usermod -aG docker jenkins
```
Then restart the Jenkins container.

# Now Set-up Kubedium in Kubedium's Container:

## Execute on Both "Master" & "Worker" Nodes

1. **Disable Swap**: Required for Kubernetes to function correctly.
    ```bash
    sudo swapoff -a
    ```

2. **Load Necessary Kernel Modules**: Required for Kubernetes networking.
    ```bash
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF

    sudo modprobe overlay
    sudo modprobe br_netfilter
    ```

3. **Set Sysctl Parameters**: Helps with networking.
    ```bash
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF

    sudo sysctl --system
    lsmod | grep br_netfilter
    lsmod | grep overlay
    ```

4. **Install Containerd**:
    ```bash
    sudo apt-get update
    sudo apt-get install -y ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt-get update
    sudo apt-get install -y containerd.io

    containerd config default | sed -e 's/SystemdCgroup = false/SystemdCgroup = true/' -e 's/sandbox_image = "registry.k8s.io\/pause:3.6"/sandbox_image = "registry.k8s.io\/pause:3.9"/' | sudo tee /etc/containerd/config.toml

    sudo systemctl restart containerd
    sudo systemctl status containerd
    ```

5. **Install Kubernetes Components**:
    ```bash
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl gpg

    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```

## Execute ONLY on the "Master" Node

1. **Initialize the Cluster**:
    ```bash
    sudo kubeadm init
    ```

2. **Set Up Local kubeconfig**:
    ```bash
    mkdir -p "$HOME"/.kube
    sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
    sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config
    ```

3. **Install a Network Plugin (Calico)**:
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
    ```

4. **Generate Join Command**:
    ```bash
    kubeadm token create --print-join-command
    ```

> Copy this generated token for next command.

---

### Execute on ALL of your Worker Nodes

1. Perform pre-flight checks:
    ```bash
    sudo kubeadm reset pre-flight checks
    ```

2. Paste the join command you got from the master node and append `--v=5` at the end:
    ```bash
    sudo kubeadm join <private-ip-of-control-plane>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --cri-socket 
    "unix:///run/containerd/containerd.sock" --v=5
    ```

    > **Note**: When pasting the join command from the master node:
    > 1. Add `sudo` at the beginning of the command
    > 2. Add `--v=5` at the end
    >
    > Example format:
    > ```bash
    > sudo <paste-join-command-here> --v=5
    > ```

---

### Verify Cluster Connection

**On Master Node:**

```bash
kubectl get nodes

```

   <img src="https://raw.githubusercontent.com/faizan35/kubernetes_cluster_with_kubeadm/main/Img/nodes-connected.png" width="70%">

---

### Verify Container Status on Worker Node
<img src="https://github-production-user-asset-6210df.s3.amazonaws.com/106548243/356852724-c3d3732f-5c99-4a27-a574-86bc7ae5a933.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAVCODYLSA53PQK4ZA%2F20241217%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20241217T113912Z&X-Amz-Expires=300&X-Amz-Signature=7a5a38a006bb504d47ccd2c35a0f6799c03ea7974e9794661a744f866bf3a6ca&X-Amz-SignedHeaders=host" width="70%">
<img src="https://github.com/user-attachments/assets/c3d3732f-5c99-4a27-a574-86bc7ae5a933" width="70%">

##### Check environment variable in the Project:
```bash
ls -la
```


## Create Namespace:
```bash
kubectl create namespace walrust
```

## Make walrust namespace as Default namespace:
```bash
kubectl config set-context --current --namespace walrust
``` 

## Enable DNS resolution on kubernetes cluster (Don't try this)

 - Check coredns pods in kube-system namespace and you will find both the pods are running on master nodes.

```bash
kubectl get pods -n kube-system -o wide | grep -i core
```
 - Following step will run core DNS pods in Master and worker both. (Don't try this)

 ```bash
 kubectl edit deploy coredns -n kube-system -o yaml
 ```

 - Make replica count 2 to 4 here.

 ### Check now: (Don't try)
 ```bash 
kubectl get pods -n kube-system -o wide | grep -i core
```

## Install Docker on Master and Worker Nodes:
```bash
sudo apt install docker.io -y

sudo chmod 777 /var/run/docker.sock

sudo usermod -aG docker ubuntu

newgrp docker

sudo chmod 777 /var/run/docker.sock
```
## Edit .env.docker file and change the public ip address with your worker node public ip:(For Walrus Project Only)
```bash
vim .env.docker
```















## Trivy Install: 
```bash
docker exec -it -u root jenkins bash
```

# Update & Install Required packages:
```bash
apt update
apt install -y wget apt-transport-https gnupg lsb-release curl
```
## For Trivy add public key & APT repo:
```bash
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -cs) main" > /etc/apt/sources.list.d/trivy.list
```
## Update & install Trivy:
```bash
apt update
apt install -y trivy
```
## verify installation:
```bash
trivy --version
```
Include trivy step after docker image build.

Convert Trivy Json report in HTML & Publish:

1) Plugin: HTML Publisher Plugin

2) Add a script in Github in root folder:

generate-trivy-html.js
         script:

```bash
const fs = require('fs');
const input = process.argv[2];
const output = process.argv[3];

if (!input || !output) {
  console.error("Usage: node generate-trivy-html.js input.json output.html");
  process.exit(1);
}

try {
  const data = fs.readFileSync(input, 'utf8');
  const json = JSON.parse(data);

  const html = `
    <html>
      <head><title>Trivy Report</title></head>
      <body>
        <h1>Trivy Vulnerability Report</h1>
        <pre>${JSON.stringify(json, null, 2)}</pre>
      </body>
    </html>
  `;

  fs.writeFileSync(output, html);
  console.log(`HTML report written to ${output}`);
} catch (err) {
  console.error("Error:", err.message);
  process.exit(1);
}
```

## Update GitHub repository:
```bash
git add generate-trivy-html.js
git commit -m "Add Trivy HTML generator script"
git push origin master
```
## OWASP Dependency-Check:

1) Plugin: OWASP Dependency-Check

2) Tool Configuration in Jenkins:
        Jenkins → Manage Jenkins → Global Tool Configuration
        Dependency-Check section → Add Dependency Check
        Give a name (e.g., Dc) 
3) After OWASP check Report location: 
```bash
/var/jenkins_home/workspace/Fintrack/./dependency-check-report.html
```
## Email Notification:

1) Go to Gmail, Set two step varifiacation, Copy the passoword.

2) In Jenkins, In Global Credentials, Create credentials, Username- vinodkaware6151@gmail.com & PassWord - App-PassWord, ID mail-cred, Description mail-cred.

3) Go to Ec2 and allow port 465, Protocol TCP, Type SMTPS

4) Go in Jenkins- Manage Jenkins- System- Extended Email Notification
SMPT Server = smtp.gmail.com
SMTP port = 465
Click on Advance
Click on none -Set email-cred which was set in Global Credentials
Tick on "Use SSL"

5) Scroll down to Email Notification
SMTP Server = smtp.gmail.com
Click on Advance Option
Tick on "Use SSL"
Tick on "Use SMTP Authentication", UserName vinodkaware6151@gmail.com, PassWord  App-Password
SMTP Port = 465
Click on Apply

6) Click on "Test configuration by sending test e-mail"
Test e-mail recipient = vinodkaware6151@gmail.com
Click on "Test configuration" You will get a message- Email was successfully sent.
CliCK on "Apply"

7) Important Note - Paste your email script outside the stages, means at the last leave on brackate and paste.

# Best Email Block to add in Jenkins Pipeline:
```bash
 post {
    always {
        script {
            def jobName = env.JOB_NAME
            def buildNumber = env.BUILD_NUMBER
            def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

            def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                        <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${env.BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
            """

            emailext(
                subject: "${jobName} - Build ${buildNumber} ${pipelineStatus.toUpperCase()}",
                body: body,
                to: 'vinodkaware6151@gmail.com',
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy-img-report.html'
            )
        }

        cleanWs()
    }
}
```

## Sonar Container:
```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:lts-community
```
## To Access Sonar: 
```bash
http://localhost:9000
```
SonarQube Container’s name: sonarqube

## To know Sonar-Containers ip:
```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <sonarqube-container-name>
```
## Password less Authentication:
```bash
ssh-copy-id -f "-o IdentityFile /home/vinod/kwid/.pem" myamazon.pem ubuntu@public_ip
```
```bash
ssh ubuntu@public_ip
```
## Download Jenkins :
```bash
sudo apt update
sudo apt upgrade -y
sudo apt install openjdk-17-jdk -y
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
## Download Docker :
```bash
sudo apt install docker.io -y

sudo chmod 777 /var/run/docker.sock

sudo usermod -aG docker ubuntu

newgrp docker

sudo chmod 777 /var/run/docker.sock
```
## Install Node.js:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl -y
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
```
## MongoDB in Container
```bash
sudo docker run -d \
  --name mongo \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  ```
