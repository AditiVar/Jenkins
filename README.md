1> Create an instance

2> Install java :
    sudo dnf install java-11-amazon-corretto -y
    java -version

3> Install jenkins:
    sudo wget -O /etc/yum.repos.d/jenkins.repo     https://pkg.jenkins.io/redhat-stable/jenkins.repo
    sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
    sudo yum install jenkins -y

4> check for the jenkins enablement:
    sudo systemctl status jenkins
    sudo systemctl enable jenkins
    sudo systemctl start jenkins
    sudo systemctl status jenkins

    Issues faced: jenkins is running on the server but not in the UI.
        We added the security group with inbound rules: 
        SSH     22      0.0.0.0/0
        HTTP    80      0.0.0.0/0
        HTTPS   443     0.0.0.0/0
        TCP     8080    0.0.0.0/0

5> login in jenkins, set the password and installed the plugins.

6> we are having jenkins -> having multiple nodes to reduce the burden on one master node.
    So we create multiple instances like windows, linux, etc on that our applications are running.
    But for assumption, windows node is not using much so that means it is wasting the resources. 
    Like it occupying the instance but executing or running the applications once or a twice in a week.
    So with the advancement of the mcroservices, we can use docker as an node. So whenever the requirement arises docker image is generated and executed the build.

7> install docker:
    # Install Docker
    sudo dnf install -y docker
    # Start Docker
    sudo systemctl start docker
    # Enable Docker on boot
    sudo systemctl enable docker
    # Check status
    sudo systemctl status docker
    # Verify installation
    docker --version

8> Allow jenkins user to use docker without sudo command. So, we add jenkins to docker group:
    getent group docker
    id jenkins
    sudo usermod -aG docker jenkins
    sudo systemctl restart docker

    Issue: When we try to switch the user to jenkins so it shows it is logged in but not opening the interactive shell
    "sudo su - jenkins
    Last login: Thu Sep  3 18:27:11 UTC 2026 on pts/3
    [ec2-user@ip-172-31-9-138 ~]$ whoami
    ec2-user
    "
    getent passwd jenkins
    You'll likely see something like:

    jenkins:x:997:995:Jenkins Automation Server:/var/lib/jenkins:/bin/false

    The Jenkins user has /bin/false as its login shell, meaning it's a service account and is not intended for interactive login.

    We can execute commands as Jenkins without changing its shell:
    sudo -u jenkins whoami


