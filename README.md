# Table of Contents
- [Table of Contents](#table-of-contents)
- [Part 1: Running Jenkins in Docker](#part-1-running-jenkins-in-docker)
- [Part 2: Setting up Jenkins Wizard](#part-2-setting-up-jenkins-wizard)
- [Part 3: Running SonarQube in Docker](#part-3-running-sonarqube-in-docker)
- [Part 4: Running Nexus in Docker](#part-4-running-nexus-in-docker)
- [Part 5: Setup Jenkins Plugins and Tools](#part-5-setup-jenkins-plugins-and-tools)
- [Part 6: Setup SonarQube with Jenkins](#part-6-setup-sonarqube-with-jenkins)
- [Part 7: Setup Nexus with Jenkins](#part-7-setup-nexus-with-jenkins)
- [Part 8: Add GitHub and DockerHub credentials in Jenkins](#part-8-add-github-and-dockerhub-credentials-in-jenkins)
- [Part 9: Creation of Jenkins Pipeline](#part-9-creation-of-jenkins-pipeline)
- [Part 10: First Phase Pipeline Checks and Feedback](#part-10-first-phase-pipeline-checks-and-feedback)
    - [Jenkins Pipeline (Classic)](#jenkins-pipeline-classic)
    - [Jenkins Pipeline (Blue Ocean)](#jenkins-pipeline-blue-ocean)
    - [SonarQube Quality and Security](#sonarqube-quality-and-security)
    - [OWASP Dependency Checker](#owasp-dependency-checker)
    - [Sonatype Nexus Repository](#sonatype-nexus-repository)
- [Part 11: Setup Kubernetes Cluster](#part-11-setup-kubernetes-cluster)
- [Part 12: Install Trivy on the Jenkins Container](#part-12-install-trivy-on-the-jenkins-container)
- [Part 13: Add Kubernetes Deployment Stages to Jenkins Pipeline Script](#part-13-add-kubernetes-deployment-stages-to-jenkins-pipeline-script)

# Part 1: Running Jenkins in Docker
**Step 1:** Create a network bridge in Docker using the following command.
```shell
docker network create jenkins
```
![Alt text](pics/01_create-docker-network.png)

**Step 2:** We will run the E-Commerce Java Application using a Docker container in a Docker container (more precisely in a Blue Ocean container). This practice is called dind, aka `docker in docker`. So, please download and run `docker:dind` Docker image using the following command.
```shell
docker run \
  --name jenkins-docker \
  --detach \
  --privileged \
  --network jenkins \
  --network-alias docker \
  --env DOCKER_TLS_CERTDIR=/certs \
  --volume jenkins-docker-certs:/certs/client \
  --volume jenkins-data:/var/jenkins_home \
  --publish 2376:2376 \
  --publish 8070:8070 --publish 5000:5000 \
  --restart always \
  docker:dind \
  --storage-driver overlay2
```

![Alt text](pics/02_run-dind.png)

The following is an explanation of the command above.

- **--name:** Specifies the name of the Docker container that will be used to run the image.

- **--detach:** Runs the Docker container in the background. However, this instance can be stopped later by running the *docker stop jenkins-docker* command.

- **--privileged:** Running dind (docker in docker aka docker within docker) currently requires *privileged access* to function properly. This requirement may not be necessary with newer Linux kernel versions.

- **--network jenkins:** This corresponds to the network created in the previous step.

- **--network-alias docker:** Makes Docker within a Docker container available as a *docker* hostname within a *jenkins* network.

- **--env DOCKER_TLS_CERTDIR=/certs:** Enables the use of TLS on the Docker server. This is recommended because we are using a *privileged container*. This environment variable controls the root directory where Docker TLS certificates are managed.

- **--volume jenkins-docker-certs:/certs/client:** Maps the /certs/client directory inside the container to a Docker volume named jenkins-docker-certs.

- **--volume jenkins-data:/var/jenkins_home:** Maps the /var/jenkins_home directory inside the container to a Docker volume named jenkins-data. This will allow other Docker containers managed by this Docker container's Docker daemon to mount data from Jenkins.

- **--publish 2376:2376:** Expose the Docker daemon port on the host machine (your computer). This is useful for executing Docker commands on the host machine (your computer) to control the inner Docker daemon.

- **--publish 8070:8070--publish 5000:5000:** Exposes ports 3000 and 5000 of the dind (docker in docker) container.

- **--restart always:** ensures the container restarts and stays up not only when there is a failure but also after the server in use has also restarted.

- **docker:dind:** This is the image of *docker:dind* itself. This image can be downloaded before running using the *docker image pull docker:dind* command.

- **--storage-driver overlay2:** Storage driver for Docker volumes. See the [Docker Storage drivers](https://docs.docker.com/storage/storagedriver/select-storage-driver) page for supported options.

It also mapping the Docker volume
![Alt text](pics/03_check-volume.png)

**Step 3:** Create `Dockerfile` and copy the following contents to your Dockerfile.
```shell
FROM jenkins/jenkins:2.464-jdk17
USER root

# Install Docker
RUN apt-get update && apt-get install -y lsb-release
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
  https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli

# Install Kubectl
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
RUN mv kubectl /usr/local/bin
RUN chmod +x /usr/local/bin/kubectl

# Install Trivy
RUN apt-get install wget apt-transport-https gnupg lsb-release -y
RUN wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | tee /usr/share/keyrings/trivy.gpg > /dev/null
RUN echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | tee -a /etc/apt/sources.list.d/trivy.list
RUN apt-get update
RUN apt-get install trivy -y

# Install Jenkins Plugins
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean:1.27.13 docker-workflow:580.vc0c340686b_54" # Initial version blueocean:1.27.9 docker-workflow:572.v950f58993843
```

**Step 4:** Create a new Docker image from the previous Dockerfile and name it `myjenkins-blueocean:2.464.2-1`.
```shell
docker build -t myjenkins-blueocean:2.464.2-1 .
```

![Alt text](pics/04_jenkins-blueocean-image.png)

**Step 5:** After that, run `myjenkins-blueocean:2.464.2-1` image as a container in Docker using the following command.
```shell
docker run \
  --name jenkins-blueocean \
  --detach \
  --network jenkins \
  --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client \
  --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 \
  --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  --volume "$HOME":/home \
  --restart=on-failure \
  --env JAVA_OPTS="-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true" \
  myjenkins-blueocean:2.464.2-1 
```

![Alt text](pics/05_run-jenkins.png)

The following is an explanation of the command above.
- **--name:** Specifies the name of the Docker container that will be used to run the image.

- **--detach:** Runs the Docker container in the background.

- **--network jenkins:** Connects this container to the previously created *jenkins* network. This makes the Docker daemon from the previous step available to this container via the *docker* hostname.

- **--env DOCKER_HOST=tcp://docker:2376:** Defines the environment variables used by *docker*, *docker-compose*, and other Docker tools to connect to the Docker daemon from the previous step.

- **--publish 8080:8080:** Map (publish/expose) port 8080 of the current container to port 8080 on the host machine (your computer). The first number represents the port on the host, while the last represents the port on the container. So, if you specify *-p 49000:8080* for this option, you will access Jenkins on the host machine via port 49000.

- **--publish 50000:50000:** Maps (exposes) port 50000 of the current container to port 50000 on the host machine (your computer). This is only necessary if you have set up one or more inbound Jenkins agents on another machine that interacts with your *jenkins-blueocean* container (the Jenkins "controller"). Inbound Jenkins agents communicate with the Jenkins controller via TCP port 50000 by default. You can change this port number on the Jenkins controller via the Configure Global Security page.

- **--volume jenkins-data:/var/jenkins_home:** Maps the container's */var/jenkins_home* directory to a Docker volume named jenkins-data.

- **--volume jenkins-docker-certs:/certs/client:ro:** Maps the container's /certs/client directory to a previously created volume named jenkins-docker-certs. This creates client TLS certificates to connect to the available Docker daemon at the path specified by the DOCKER_CERT_PATH environment variable.

- **--volume "$HOME":/home:** Maps the $HOME directory on the host machine (your computer, usually the */Users/your-username* directory) to the */home* directory on the container.

- **--restart=on-failure:** Configure the Docker container restart policy to restart when a failure occurs.

- **--env JAVA_OPTS="-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true":** Allows checkout on local repositories.

- **myjenkins-blueocean:2.464.2-1:** The name of the Docker image you created in the previous step.


# Part 2: Setting up Jenkins Wizard
**Step 1:** Open your browser and run http://{{Public-IP}}:8080. Wait until the **Unlock Jenkins** page appears.

**Step 2:** As it says, you are required to copy the password from the Jenkins log to ensure that Jenkins is accessed safely by administrators. 

**Step 3:** Display the Jenkins console log with the following command in Terminal/CMD
```shell
docker logs jenkins-blueocean
```

**Step 4:** From the Terminal/CMD application, copy the password between the 2 series of asterisks.
![Alt text](pics/06_jenkins-initial-password.png)

**Step 5:** Return to the *Unlock Jenkins* page in the browser, paste the password into the **Administrator password** column and click **Continue**.

**Step 6:** After that, the Customize Jenkins page appears. Select **Install suggested plugins**. The setup wizard shows progress that Jenkins is being configured and recommended plugins are being installed. This process may take up to several minutes. <br>
**Note:** If there is a failure during the plugin installation process, you can click **Retry** (to repeat the plugin installation process until it is successful) or click **Continue** (to skip and proceed directly to the next step).

**Step 7:** Once done, Jenkins will ask you to create an administrator user. When the *Create First Admin User* page appears, fill it in according to your wishes and click **Save and Continue**.

**Step 8:** On the Instance Configuration page, select **Save and Finish**. That means, we will access Jenkins from the url http://{{localhost}}:8080/. 

**Step 9:** When the *Jenkins is ready* page appears, click the **Start using Jenkins** button.
Note: This page may indicate *Jenkins is almost ready!* If so, click **Restart**. If the page doesn't refresh automatically after a few minutes, manually click the *refresh* icon in your browser.

**Step 10:** If necessary, you can log back into Jenkins using the credentials you created earlier.
![Alt text](pics/07_jenkins-home.png)


# Part 3: Running SonarQube in Docker
**Step 1:** We will use the free community edition image called **lts-community**. There is also a paid version available called the developer version.
```shell
docker run -d --name sonarqube -p 30002:9000 sonarqube:lts-community
```

The first 30002 is the host port (feel free to use another free port on your VM) which is mapped to the second 9000 port, the Docker container port. This is because the SonarQube service is exposed on port 9000. Use docker ps to view the status of the SonarQube container it should be Up and running.

![Alt text](pics/08_sonarqube-container.png)

**Step 2:** Now let's try to access the SonarQube service running inside a Docker container on the SonarQube VM on port 30002.

**Step 3:** Enter your initial username and password as **admin** for the SonarQube service. Then, update the default password to a strong one.

![Alt text](pics/09_sonarqube-login.png)

![Alt text](pics/10_sonarqube-reset-password.png)


# Part 4: Running Nexus in Docker
**Step 1:** Nexus service runs on default port 8081 just like SonarQube service runs on default port 9000 and Jenkins service runs on default port 8080. Run the following command to run the Nexus service inside a Docker container instead of running it on the Nexus VM/node host.
```shell
docker run -d --name nexus -p 30003:8081 sonatype/nexus3
```

The first 30003 is the host port (feel free to use another free port on your VM) which is mapped to the second 8081 port, the Docker container port. This is because the Nexus service is exposed on port 8081. Use docker ps to view the status of the Nexus container it should be Up and running.

![Alt text](pics/11_nexus-container.png)

**Step 2:** Copy and paste the Nexus VM IP address on your web browser to access the Nexus service dashboard.

![Alt text](pics/12_nexus-login.png)

**Step 3:** For SonarQube, we know the default username and password is **admin**. For Nexus, to sign in, the username is also **admin** by default, but the default password is different for everyone. Just like with Jenkins, where we got the password from a file, we will also get our initial password for Nexus from a file. First, we need to access the Docker container using the following commands.
```shell
docker exec -it nexus /bin/bash
cat /nexus-data/admin.password && echo
```

![Alt text](pics/13_nexus-initial-password.png)

**Step 4:** Copy and paste the password.

![Alt text](pics/14_nexus-login-2.png)

**Step 5:** Set the new password for Nexus.

![Alt text](pics/15_nexus-login-3.png)

**Step 6:** Enable anonymous access

![Alt text](pics/16_nexus-login-4.png)


# Part 5: Setup Jenkins Plugins and Tools
**Step 1:** Let's install some plugins on the Jenkins VM. From the Dashboard, navigate to Manage Jenkins in System Configuration and select Plugins.

![Alt text](pics/17_manage-jenkins.png)

**Step 2:** Go to the Available Plugins tab on the left side. Search for **SonarQube Scanner** which will start the analysis.

**Step 3:** Search for nexus and select **Nexus Artifact Uploader**. 

**Step 4:** Search for docker and select **"Docker"**, **"docker-build step"**, and **"CloudBees Docker Build and Publish"**.

![Alt text](pics/18_install-plugins.png)

**Step 5:** After that search for owasp and select **"OWASP Dependency-Check"** and search for jdk and select **"Eclipse Temurin installer"**

![Alt text](pics/19_install-plugins-2.png)

This will help us install multiple versions of jdk whichever we want to use we can go with that specific version. Finally, click on the Install button to Install all the selected plugins on the Jenkins VM. 

**Step 6:** Now navigate to the Manage Jenkins page and select the Tools option.

![Alt text](pics/20_jenkins-tools.png)

**Step 7:** In **JDK Installations** select **Add JDK** option give it a name and select **Install automatically** under which select **Install from adoptium.net** and select jdk 17 version. repeat the same steps for jdk 11 version in case jdk 17 fails we should have jdk 11.

![Alt text](pics/21_jenkins-tools-jdk.png)

**Step 8:** Under **SonarQube Scanner Installations** select **Add SonarQube Scanner** provide a name like sonarqube-scanner. 

![Alt text](pics/22_jenkins-tools-sonar.png)

**Step 9:** For **Maven Installations** select **Add Maven** and provide a name like maven and select the version from **Install from Apache** dropdown list.

![Alt text](pics/23_jenkins-tools-maven.png)

**Step 10:** Under **Dependency-Check Installations** select **Add Dependency-Check** provide a name like dependency-check. Select **Install Automatically** radio button and click on **Add Installer** dropdown list and select **Install from github.com** and select a version.

![Alt text](pics/24_jenkins-tools-dp.png)

**Step 11:** Same for the Docker Installations click on the Add Docker button to provide a name such as docker. Select **Install Automatically** radio button and click on **Add Installer** dropdown list and select **Download from docker.com**

![Alt text](pics/25_jenkins-tools-docker.png)

**Step 12:** Click on the Apply button and then on the Save button to save the configurations. 

Now all the tools are configured with Jenkins but not yet connected the SonarQube and Nexus services with Jenkins.


# Part 6: Setup SonarQube with Jenkins

To connect SonarQube with Jenkins we need some authentication like username and password but when tools need to communicate with each other we will be using tokens.

**Step 1:** Go to the SonarQube dashboard in the **Administration** section click on the **Security** dropdown list and select **Users** and an admin user will be present that is us we signed In using an admin username and password.

**Step 2:** On admin user under **Tokens** click on the hamburger icon under **Tokens for AdministratorGenerate Tokens** 

![Alt text](pics/26_sonarqube-token.png)

**Step 3:** Give a name for that token like admin-token

![Alt text](pics/27_sonarqube-token-2.png)

**Step 4:** Click on the Generate button a random hash will be generated.

![Alt text](pics/28_sonarqube-token-3.png)

**Step 5:** Go to Jenkins dashboard and navigate to **Manage Jenkins** In the Security section select **Credentials**. Select the **global** and click on **Add Credentials**

![Alt text](pics/29_jenkins-credentials.png)

**Step 6:** In **New Credentials** under **Kind** select **Secret text** in **Scope** keep **Global** selected. In **Secret**, paste the SonarQube Admin user token and In **ID** provide a string like sonar-token same for description. Click on Create to create a credential.

**Step 7:** On the Jenkins dashboard navigate to **Manage Jenkins** and select **System**. Scroll down to the **SonarQube servers**. Click on the **Add SonarQube** button. In the **Name** enter a string like sonar. In the **Server URL** enter the SonarQube Server IP with a port number like **http://{{SonarQube-Node-IP}}:{{Host-Port}}** in the **Server authentication token** dropdown select the token we created in Credentials. Click on apply and save button.

![Alt text](pics/31_jenkins-sonar-server.png)

> **Note:** Because we are deploying SonarQube on the same VM with Jenkins, we can use the IP from the default Docker Network Bridge from the **Gateway** section. You can run following command to get the IP
```shell
docker network inspect bridge
```

![Alt text](pics/32_docker-network-bridge.png)


# Part 7: Setup Nexus with Jenkins

Now let's configure and connect Nexus with Jenkins this configuration needs to be done inside the source configuration files. We need to install a plugin for Managed Files where we create configuration credentials for servers. 

**Step 1:** Go to Jenkins dashboard and select **Manage Jenkins** and select Plugins click on **Available Plugins**. Search for **Config File Provider** plugin. Select it and install it.

![Alt text](pics/33_install-plugins-3.png)

**Step 2:** Once the **Config File Provider** plugin installation is complete, navigate to the **Manage Jenkins** and now the **Managed files** tab will be visible click on that.

![Alt text](pics/34_managed-files-plugins.png)

**Step 3:** Click on **Add a new Config** button and select **Global Maven settings.xml** scroll down in ID enter a string like **global-maven** and click on the Next button. 

![Alt text](pics/35_global-maven.png)

**Step 4:** In the **Edit Configuration File** scroll down in the Content section copy line number 119 to 123 and paste it below line 124.

![Alt text](pics/36_global-maven-2.png)

**Step 5:** Go to the Nexus dashboard and click on the **Setting** icon that is beside the search tab and from the right-hand side select **Repository** and **Repositories**. Select **maven-releases** repository and copy the **Name** of the repository which is maven-releases.

![Alt text](pics/37_maven-releases.png)

**Step 6:** paste the **maven-releases** repository name inside the `<id> </id>` block of the `<server> </server>` block in Content of Jenkins dashboard. For username and password block provide the Nexus admin username and password. For password security, we can use the **Server Credentials** option above the content tab.

![Alt text](pics/38_global-maven-3.png)

**Step 7:** Repeat the same steps for **maven-snapshots** copy the above `<server> </server>` block and copy and paste the id from the Nexus dashboard which is **maven-snapshots** and provide the username and password of the Nexus admin user. Next click on Submit button.

![Alt text](pics/39_global-maven-4.png)

What is the difference between releases and snapshots?

> Releases get deployed in the production environment and snapshots are deployed in non-production environments like development.

**Step 8:** Now we need to edit the pom.xml file which is present on the GitHub repository. 

- From the Nexus dashboard click on the copy button for maven-releases copy the URL and paste it into the `<repository> </repository>` block's `<url> </url>` block. repeat same steps for maven-snapshots.

- From the Nexus dashboard click on the copy button for maven-snapshots copy the URL and paste it into the `<snapshotRepository> </snapshotRepository>` block's `<url> </url>` block.

> **Note:** Again, because we are deploying Nexus on the same VM with Jenkins, we can use the IP from the default Docker Network Bridge from the **Gateway** section. You can run following command to get the IP
```shell
docker network inspect bridge
```

![Alt text](pics/40_pom-file.png)

If your code is on the local environment make changes and push it on github later. If your code is on GitHub then make changes on GitHub directly.


# Part 8: Add GitHub and DockerHub credentials in Jenkins

Because, Jenkins will be interact with the GitHub and DockerHub, so we need to add those credentials in Jenkins

**Step 1:** Go to Jenkins dashboard and navigate to **Manage Jenkins** In the Security section select **Credentials**. Select the **global** and click on **Add Credentials**

**Step 2:** For **Kind** keep the **Username with password** selected and keep **Scope** Global for the **Username** enter your dockerhub username and for **Password** enter your dockerhub password (you can also use personal access token) and enter a string for ID like dockerhub-credentials

![Alt text](pics/41_jenkins-credentials-dockerhub.png)

**Step 3:** Again click on Add Credentials, for **Kind** keep the **Username with password** selected and keep **Scope** Global for the **Username** enter your github username and for **Password** enter your github password (you can also use personal access token) and enter a string for ID like github-credentials

![Alt text](pics/42_jenkins-credentials-github.png)


# Part 9: Creation of Jenkins Pipeline
**Step 1:** Click on the New Item button on the Jenkins dashboard enter a name for your pipeline select Pipeline and click on the OK button.

![Alt text](pics/43_jenkins-pipeline.png)

**Step 2:** Provide a short description of the pipeline. Select **Discard old builds** and inside **Days to keep builds** enter 10 **Max # of builds to keep** tab enter 2.

![Alt text](pics/44_jenkins-pipeline-2.png)

**Step 3:** Scroll down to the Pipeline section and select Hello World from the dropdown list to get a basic groovy syntax. Remember two different stages can't have the same name. Make use of Pipeline Syntax if you are not aware of any specific syntax of Groovy.

![Alt text](pics/45_jenkins-pipeline-3.png)

> **Note:** We have not defined Java or Maven so whatever is installed on the Jenkins server that it's going to use. To make use of specific versions of tools like Java and Maven we need to define the versions in the tools{} block of the pipeline script.

**Step 4:** Install the **Pipeline Maven Integration** Plugin from the Plugins tab on the Jenkins dashboard.

![Alt text](pics/46_install-plugins-4.png)

**Step 5:** Pipeline script till Deploy Artifact to Nexus Stage.
```groovy
pipeline {
  agent any
  tools {
    maven 'maven'
    jdk 'jdk17'
  }
  environment {
    SCANNER_HOME = tool 'sonarqube-scanner'
  }
  stages {
    stage('Git Checkout') {
      steps {
        git branch: 'main', credentialsId: 'github-credentials', url: 'https://github.com/kandlagifari/E-commerce-Application-CICD.git'
      }
    }
    stage('Code Compilation') {
      steps {
        sh "mvn compile"
      }
    }
    stage('Unit Tests') {
      steps {
        sh "mvn test -DskipTests=true"
      }
    }
    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonar') {
          sh ''' 
          $SCANNER_HOME/bin/sonar-scanner \
          -Dsonar.projectKey=E-commerce-Application \
          -Dsonar.projectName=E-commerce-Application \
          -Dsonar.java.binaries=. 
          '''
        }
      }
    }
    stage('OWASP Dependency-Check') {
      steps {
        dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'dependency-check'
        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
      }
    }
    stage('Maven Build') {
      steps {
        sh "mvn package -DskipTests=true"
      }
    }
    stage('Deploy Artifact to Nexus') {
      steps {
        withMaven(globalMavenSettingsConfig: 'global-maven', jdk: 'jdk17', maven: 'maven', mavenSettingsConfig: '', traceability: true) {
          sh "mvn deploy -DskipTests=true"
        }
      }
    }
  }
}
```

**Step 6:** You can also refer to Jenkinsfile on the GitHub repository by specifying the **Repository URL** to GitHub HTTPS URL. If you are using private repository, you also will need to specify the **Credentials**, which we already setup on the previous section.

![Alt text](pics/47_jenkins-pipeline-4.png)


# Part 10: First Phase Pipeline Checks and Feedback

Since this is a DevSecOps pipeline, security analysis results from SonarQube and OWASP Dependency Checker provide valuable insights for improving code quality and mitigating vulnerabilities. Let's review these results and provide actionable feedback to the developer.

### Jenkins Pipeline (Classic)
Access the results in Jenkins under Dashboard > E-commerce-Application-CICD

![Alt text](pics/48_jenkins-classic-build.png)

### Jenkins Pipeline (Blue Ocean)
Access the results in Jenkins under Dashboard > E-commerce-Application-CICD > Open Blue Ocean > Choose the Build

![Alt text](pics/49_jenkins-blueocean-build.png)

### SonarQube Quality and Security
Access the SonarQube dashboard at `<IP-address>:30002` (default port is 9000)

![Alt text](pics/50_sonarqube-result.png)

Findings:
- 0 Vulnerabilities (positive)
- 25 Bugs
- 2 Security Hotspots

### OWASP Dependency Checker
Access the results in Jenkins under Dashboard > E-commerce-Application-CICD > #6 > Dependency-Check.

![Alt text](pics/51_dependency-check.png)

Identified vulnerabilities: 70 Critical, 141 High, 173 Medium, and 8 Low severity.

**Action:** Collaborate with the developer to address these vulnerabilities as soon as possible.

### Sonatype Nexus Repository
Access the SonarQube dashboard at `<IP-address>:30003` (default port is 8081)

![Alt text](pics/52_nexus-artifact.png)


# Part 11: Setup Kubernetes Cluster

Now we need to set up a Kubernetes cluster. In this demo, I will use KinD to create simple Kubernetes Cluster with only one Control Plane node

**Step 1:** Install **KinD** from release binaries, you can always refer to the [official documentation](https://kind.sigs.k8s.io/docs/user/quick-start/)
```shell
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
# For ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

**Step 2:** Install **kubectl** by running the following command
```shell
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo mv kubectl /usr/local/bin
sudo chmod +x /usr/local/bin/kubectl
kubectl version --client
```

**Step 3:** Create a cluster configuration file called **cluster.yaml** as follows 
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```

**Step 4:** Create a cluster by running the following command
```shell
kind create cluster --config=cluster.yaml
kubectl cluster-info --context kind-kind
```

You will have simple Kubernetes Cluster with one nodes

![Alt text](pics/53_setup-k8s-cluster.png)

Once the Kubernetes cluster is set we need to create a service account, and a role assign that role to the service account, create a secret for that service account, and create a token. We also need to create a namespace inside a Kubernetes cluster.

**Step 5:** On the control plane or master node of the Kubernetes cluster run the below commands
```shell
kubectl create namespace ecommerce
```

**Step 6:** Create a service account with the name jenkins so that using this service account Jenkins can access the Kubernetes cluster. Create a Kubernetes manifest file named as **serviceaccount.yaml** using Vim editor (you can also create new directory and store YAML file on the repository) and copy the below content in that file
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: ecommerce
```

**Step 7:** Execute the below command to apply the changes
```shell
kubectl apply -f serviceaccount.yaml
```

**Step 8:** Now create a role to define what kind of access a service account will have that will be attached to this role. Create a k8s manifest file named **role.yaml**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: ecommerce
rules:
- apiGroups:
  - ""
  - apps
  - autoscaling
  - batch
  - extensions
  - policy
  - rbac.authorization.k8s.io
  resources:
  - pods
  - componentstatuses
  - configmaps
  - daemonstes
  - deployments
  - events
  - endpoints
  - horizontalpodautoscalers
  - ingress
  - jobs
  - limitranges
  - namespaces
  - nodes
  - pods
  - persistentvolumes
  - persistentvolumeclaims
  - resourcequotas
  - replicasets
  - replicationcontrollers
  - serviceaccounts
  - services
  verbs: ["get", "list", "watch", "create", "updadte", "patch", "delete"]
```

**Step 9:** Run the below command to apply the changes
```shell
kubectl apply -f role.yaml
```

**Step 10:** Next, we need to assign this role to the service account that we created for that Create a k8s manifest file named **assign.yaml**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: ecommerce
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: ecommerce
```

**Step 11:** Run the below command to apply the changes
```shell
kubectl apply -f assign.yaml
```

Generally, the root account is not used to perform deployments using CI/CD tools. Instead, service accounts are used.

**Step 12:** Now we need to generate a token for the service account so that Jenkins can access the service account using the token. Create a k8s manifest file named **serviceaccounttoken.yaml**
```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: jenkins-token
  namespace: ecommerce
  annotations:
    kubernetes.io/service-account.name: jenkins
```

**Step 13:** Run the below command to apply the changes in webapps namespace
```shell
kubectl apply -f serviceaccounttoken.yaml
```

**Step 14:** To view the token that is created inside the secret we need to run the following command. Remember that we need to get/retrieve the decoded token and not the base64 encoded token.
```shell
kubectl get secret jenkins-token -n ecommerce -o jsonpath="{.data.token}" | base64 --decode && echo
```

![Alt text](pics/54_jenkins-sa-token.png)

Now we need to find the config file which is inside the .kube hidden folder that we created earlier. The config file contains complete information about the k8s cluster.


# Part 12: Install Trivy on the Jenkins Container
> This part is optional, because we already installed trivy along with the docker image build process on the earlier. If we build the docker images, the Jenkins setup such as Plugins, Tools, Credentials will be persistent because we are using Docker Volume

**Step 1:** As there is no official trivy plugin available on Jenkins we manually have to install trivy on Jenkins container. 
```shell
docker exec -it jenkins-blueocean bash
```

**Step 2:** Run the below commands on Jenkins container to install trivy.
```shell
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```


# Part 13: Add Kubernetes Deployment Stages to Jenkins Pipeline Script

**Step 1:** Now let's install Kubernetes and Kubernetes CLI plugins from the Jenkins dashboard in the Plugins section under Available Plugins. Click on the Install button.

![Alt text](pics/55_install-plugins-5.png)

**Step 2:** Create a new credential in Jenkins so that Jenkins can access the Kubernetes cluster as a service account user.

![Alt text](pics/56_jenkins-credentials-k8s.png)

To view the secret service account token that we created earlier execute the following command and paste it into the Secret tab while creating a credential.
```shell
kubectl get secret jenkins-token -n ecommerce -o jsonpath="{.data.token}" | base64 --decode && echo
```

**Step 3:** For the Kubernetes API endpoint from the **.kube/config** file output copy the IP of the server and paste it on Jenkins' **Kubernetes API endpoint**.

![Alt text](pics/57_k8s-sample-pipeline.png)

> `kind get kubeconfig --internal` will give you the contents of a KUBECONFIG you can use within the `kind` docker network instead of on the host. Because of how docker handles networking you'll still need your container to be on that network (e.g. with `docker network connect`).

```shell
# Get the API server endpoint
kind get kubeconfig --internal

# Connect Jenkins container into KinD network
docker network connect kind jenkins-blueocean
```

**Step 4:** Now update the image name and container port number inside the kubernetes manifest file named **deploymentservice.yaml**

**Step 5:** Complete Jenkins Pipeline script looks like this in which docker, trivy and kubernetes deployment stages are added and it's a final pipeline script (If you are using Jenkinsfile, you can adjust it and push to GitHub repository)

```groovy
pipeline {
  agent any
  tools {
    maven 'maven'
    jdk 'jdk17'
  }
  environment {
    SCANNER_HOME = tool 'sonarqube-scanner'
  }
  stages {
    stage('Git Checkout') {
      steps {
        git branch: 'main', credentialsId: 'github-credentials', url: 'https://github.com/kandlagifari/E-commerce-Application-CICD.git'
      }
    }
    stage('Code Compilation') {
      steps {
        sh "mvn compile"
      }
    }
    stage('Unit Tests') {
      steps {
        sh "mvn test -DskipTests=true"
      }
    }
    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonar') {
          sh ''' 
          $SCANNER_HOME/bin/sonar-scanner \
          -Dsonar.projectKey=E-commerce-Application \
          -Dsonar.projectName=E-commerce-Application \
          -Dsonar.java.binaries=. 
          '''
        }
      }
    }
    stage('OWASP Dependency-Check') {
      steps {
        dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'dependency-check'
        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
      }
    }
    stage('Maven Build') {
      steps {
        sh "mvn package -DskipTests=true"
      }
    }
    stage('Deploy Artifact to Nexus') {
      steps {
        withMaven(globalMavenSettingsConfig: 'global-maven', jdk: 'jdk17', maven: 'maven', mavenSettingsConfig: '', traceability: true) {
          sh "mvn deploy -DskipTests=true"
        }
      }
    }
    stage('Docker Build and Tag'){
      steps{
        script{
          withDockerRegistry(credentialsId: 'dockerhub-credentials', toolName: 'docker') {
            sh "docker build -t kandlagifari/ecommerce-application:latest -f docker/Dockerfile ."
          }
        } 
      }
    }
    stage('Trivy Image Scan'){
      steps{
        sh "trivy image kandlagifari/ecommerce-application:latest > trivy-report.txt"
      }
    }
    stage('Push Image to Dockerhub'){
      steps{
        script{
          withDockerRegistry(credentialsId: 'dockerhub-credentials', toolName: 'docker') {
            sh "docker push kandlagifari/ecommerce-application:latest"
          }
        } 
      }
    }
    stage('Deploy to Kubernetes'){
      steps{
        script{
          withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s-service-account-token', namespace: 'ecommerce', serverUrl: 'https://kind-control-plane:6443']]) {
            sh "kubectl apply -f deploymentservice.yaml -n ecommerce --validate=false"
            sh "kubectl get svc -n ecommerce"
          }
        } 
      }
    }
  }
}
```
