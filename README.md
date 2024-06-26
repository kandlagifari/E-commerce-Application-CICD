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
FROM jenkins/jenkins:2.463-jdk17
USER root
RUN apt-get update && apt-get install -y lsb-release
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
  https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean:1.27.9 docker-workflow:572.v950f58993843" # New version blueocean:1.27.13 docker-workflow:580.vc0c340686b_54
```

**Step 4:** Create a new Docker image from the previous Dockerfile and name it `myjenkins-blueocean:2.463.2-1`.
```shell
docker build -t myjenkins-blueocean:2.463.2-1 .
```

![Alt text](pics/04_jenkins-blueocean-image.png)

**Step 5:** After that, run `myjenkins-blueocean:2.463.2-1` image as a container in Docker using the following command.
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
  myjenkins-blueocean:2.463.2-1 
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

- **myjenkins-blueocean:2.463.2-1:** The name of the Docker image you created in the previous step.


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


# Part 6: Set up SonarQube with Jenkins

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


# Part 7: Set up Nexus with Jenkins

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

