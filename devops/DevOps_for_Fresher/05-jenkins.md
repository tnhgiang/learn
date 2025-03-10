# Jenkins

## Tại sao cần sử dụng Jenkins?

- Biết thêm kiến thức sẽ có lợi thế hơn.
- Jenkins hỗ trợ CI/CD cho các nền tảng `VCS` khác nhau.
- Jenkins không chỉ để làm mỗi CI/CD mà còn làm được nhiều thứ khác.

## Jenkins là gì?

Jenkins là công cụ giúp tự động hoá bất kỳ dự án nào.

Jenkins tích hợp được rất nhiều plugins, nên nó rất mạnh mẽ.

## Jenkins chuyên sâu?

Để đi được chuyên sâu vào Jenkins cần phải biết code. Cụ thể ngôn ngữ ở đây là `groovy`.

## Cài đặt Jenkins

1. Tạo `server` để host `jenkins`.
2. Cài đặt `jenkins`:
    - Tạo thư mục để chứa cài đặt `jenkins`:

        ```bash
        mkdir -p /tools/jenkins/
        cd /tools/jenkins/
        ```

    - Tạo script để cài jenkins

        ```bash
        #!/bin/bas
        apt install openjdk-17-jdk openjdk-17-jre -y
        java --version
        wget -p -O - <https://pkg.jenkins.io/debian/jenkins.io.key> | apt-key add -
        sh -c 'echo deb <http://pkg.jenkins.io/debian-stable> binary/ > /etc/apt/sources.list.d/jenkins.list'
        apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 5BA31D57EF5975CA
        apt-get update
        apt install jenkins -y
        systemctl start jenkins

        # Enable Jenkins to start on boot
        systemctl enable jenkins
        # This command enables the Jenkins service to start automatically when the system boots up
        # It ensures that Jenkins will be up and running even after a system restart

        # Allow incoming traffic on port 8080 through the firewall
        ufw allow 8080
        # This command modifies the system's firewall rules to allow incoming network traffic on port 8080
        # which is the default port for accessing Jenkins' web interface
        # This ensures that users can access Jenkins via the browser
        ```

    - Tiến hành install

        ```bash
        chmod +x jenkins-install.sh
        sh jenkins-install.sh
        ```

    - Kiểm tra `jenkins`

        ```bash
        systemctl status jenkins
        ```

    - Dùng `nginx` để thiết lập `reversed proxy` để tự động `foward` `port:8080` thành `port:80`:
        - Install `nginx`:

            ```bash
            apt install nginx -y
            ```

        - Cấu hình:

            ```bash
            vi /etc/nginx/conf.d/<domain>.conf
            # Ví dụ: vi /etc/nginx/conf.d/jenkins-server.remote.conf
            ```

            ```ini
            # Define a server block that will handle HTTP traffic

            server {
                # Listen on port 80 (standard for HTTP traffic)
                listen 80;

                # Define the domain or IP address that this server block will respond to
                # In this case, it will respond to requests for "jenkins-server.remote"
                server_name jenkins-server.remote;

                # Define the location block for the root path "/"
                # This means that any request to the root of the site will be handled by this block
                location / {
                    # Proxy the incoming HTTP requests to Jenkins on port 8080
                    # This means requests to http://jenkins-server.remote/ will be forwarded to http://jenkins-server.remote:8080
                    proxy_pass http://jenkins-server.remote:8080;

                    # Ensure the correct HTTP version is used for the proxy connection
                    proxy_http_version 1.1;

                    # Set the 'Upgrade' header for WebSocket support (in case WebSocket is used in the proxied app)
                    proxy_set_header Upgrade $http_upgrade;

                    # Set the 'Connection' header to 'keep-alive' to maintain a persistent connection
                    proxy_set_header Connection keep-alive;

                    # Set the 'Host' header to the original host in the incoming request
                    # This ensures that the proxied server knows the original domain that the request was sent to
                    proxy_set_header Host $host;

                    # Disable caching of the proxied request to ensure the request is forwarded every time
                    proxy_cache_bypass $http_upgrade;

                    # Set the 'X-Forwarded-For' header to pass the client’s original IP address
                    # This is important for tracking the real client IP (useful for logging and security)
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

                    # Set the 'X-Forwarded-Proto' header to pass the protocol used (http or https) in the original request
                    proxy_set_header X-Forwarded-Proto $scheme;
                }
            }

            ```

        - Khởi động lại `nginx`: `systemctl restart nginx`.

    - Add `host` cho `jenkins` ở các `server` cần kết nối đến `jenkins`
    - Tiên hành thiết lập trên `UI`.
        - Nên chọn `Install suggested plugins` nếu chưa rành nên cài những `plugins` gì cho jenkins.
        - `Jenkins URL` trong video được thiết lập trực tiếp đến `port:8080` chứ không thông qua
        `reversed proxy`.

## Tìm hiểu Jenkins

### System

`Manage Jenkins > System`

`Jenkins` cũng tạo ra một `user` cho `jenkins`.

### Plugins

`Manage Jenkins > Plugins`

### Node

`Manage Jenkins > Nodes`

Dùng `Jenkins agent` để thêm các `server` triển khai dự án vào trong `jenkins server`. `Jenkins
server` là nơi điều phối các pipeline, chứ không phải là nơi trực tiếp triển khai dự án.

**Chú ý**: theo kinh nghiệm của giảng viên, nhưng phần nào có thể tối thiểu sử dụng `ssh` thì nên
hạn chế . Vì bản chất `ssh` là `kết nối ngang hàng` sẽ có những rủi ro. Nếu có dùng `ssh`, ví dụ như
`Ansible` thì dùng xong đóng lại luôn.

### Security

`Mange Jenkins > Security`

Mặc định `Jenkins` sẽ không có phân quyền, để sử dụng được tính năng phân quyền, cần phải cài thêm plugins.

### Credentials

`Mange Jenkins > Credentials`: nơi lưu trữ thông tin đăng nhập như: `tài khoản và mật khẩu`,
`secret file`, `private key`, ...

### System Log

`Manage Jenkins > System Log`: nơi lưu trữ `log` của hệ thống.

### Jenkins CLI

Một phần rất mạnh giúp hoàn toàn tự động mọi thứ, được áp dụng rất nhiều trong thực tế.

## Jenkins CI/CD (Continuos Deployment)

Nếu dùng `jenkins` thì nên dùng `jenkins agent` thay vì dùng `ssh` để triển khai dự án:

- Vì `jenkins agent` do chính `jenkins` ban hành và phát triển, nên được hỗ trợ tốt hơn đồng
thời cũng có nhiều tính năng mà `ssh` không có: quản lý trạng thái `server`, xem `log`, cấu hình
và cài đặt, và an toàn hơn.
- Còn `ssh`, bản chất là kết nối ngang hàng nên có nguy cơ làm chết hệ thống với hàng loạt các
server nếu không cẩn thận.

Để dùng `jenkins agent`, thì trên bất cứ một `server` nào để triển khai dự án đều phải cài đặt `agent`.

### Các bước tiến hành

1. Thiết lập ban đầu trên mỗi `server` để triển khai dự án
    - Cài đặt `java` với `version` giống như trên `jenkins server`. Ví dụ trên `jenkins server` dùng
    `java-17` thì trên `server` cũng phải cài `java-17`:

        ```bash
        apt install openjdk-17-jdk
        # Choose default version when server has various java versions.
        update-alternatives --config java
        ```

    - Tạo `user` cho `jenkins`:

        ```bash
        adduser jenkins
        ```

    - Tạo `workdir` cho `jenkins`:

        ```bash
        mkdir -p /var/lib/jenkins
        chown jenkins. /var/lib/jenkins
        ```

2. Cài đặt `jenkins agent` trên `jenkins server`:

    - Mở `port` trên `jenkins server` cho các `agent` liên kết đến: `Manage Jenkins > Security > Fixed`.

    - Trên `UI`, `Manage Jenkins > Nodes > New Node`. Điền thông tin được yêu cầu:
        - `Name`: Name that uniquely identifies an agent within this Jenkins installation.
        - `Number of executors`: The maximum number of concurrent builds that Jenkins may perform on
        this node.
        - `Remote root directory`: The working directory for `jenkins` on the `remote serrver`.
        - `Labels`: Labels (or tags) are used to group multiple agents into one logical group.

    - Truy cập vào các `agent` đã được tạo, lấy `command line` dùng để kết nối các `agent` đến
    `jenkins server`.

3. Kết nối `agent` đến `jenkins server`:
    - Không sử dụng `service` cho `jenkins agent`:

        ```bash
        cd /var/lib/jenkins
        su jenkins
        # Chạy các comment line từ bước 2
        echo 8344ce58af67842150392618fcf7e9a4a2248619867befa61b2ff7e434d5cb47 > secret-file
        curl -sO http://jenkins-server.remote:8080/jnlpJars/agent.jar
        java -jar agent.jar -url http://jenkins-server.remote:8080/ -secret @secret-file -name "dev-server" -webSocket -workDir "/var/lib/jenkins" > nohup.out 2>&1 &
        ```

        - Khi `slave server` bị tắt hoặc `agent` gặp lỗi thì kết nối từ `agent` đến `jenkins` sẽ bị
        ngắt đi. Để kết nội lại, sẽ phải thực hiện lại những câu lệnh ở trên. Nên, sẽ **mất thời gian
        và công sức**.

    - Sử dụng `service` cho `jenkins agent`:

        ```bash
            vi /etc/systemd/system/jenkins-agent.service
        ```

        ```ini
        # The [Unit] section describes the service and its relationship to other system components
        [Unit]

        # Description: A brief, human-readable name of the service
        Description=Jenkins Agent Service

        # After: This specifies the ordering of services. The Jenkins agent will start after
        # the network is initialized and available (network.target)
        After=network.target

        # The [Service] section defines the behavior of the service itself
        [Service]

        # Type: Defines how the service is managed. 'simple' means the service is considered
        # started immediately after the ExecStart command is executed
        Type=simple

        # WorkingDirectory: Specifies the directory where the service will run. This is where
        # the Jenkins agent will execute, and it is usually set to Jenkins' working directory
        WorkingDirectory=/var/lib/jenkins

        # ExecStart: The command that starts the service. This one launches the Jenkins agent
        # using a specific Java command and connects it to the Jenkins server
        ExecStart=/bin/bash -c 'java -jar agent.jar -url <http://jenkins-server.remote:8080/> \
            -secret @secret-file -name "dev-server" -webSocket -workDir "/var/lib/jenkins"'

        # User: Specifies the user under which the service should run. Here, it runs under
        # the 'jenkins' user account for security reasons
        User=jenkins

        # Restart: Defines the behavior when the service exits. 'always' ensures the service
        # is automatically restarted if it crashes or stops
        Restart=always

        # The [Install] section defines how the service should be installed and enabled to
        # start on boot
        [Install]

        # WantedBy: Specifies the target under which the service should be started
        # 'multi-user.target' indicates the service will be started during the boot process
        # when the system reaches multi-user mode (a common target for general system services)
        WantedBy=multi-user.target

        ```

        ```bash
        systemctl daemon-reload
        systemctl start jenkins-agent
        systemctl status jenkins-agent
        ```

        - `Service` sẽ tự động được khởi tạo lại khi gặp lỗi hoặc reboot.

4. Kết nối `jenkins` đến `gitlab`, để khi có `commit` hoặc `PR` trên `gitlab` thì `pipeline` trên
`jenkins` sẽ được triển khai:

    - Cài đặt `Plugins` trên `jenkins`: Gitlab và Blue Ocean. Và tick chọn: Restart Jenkins when
    installation is completed and no jobs are running.

    - Tạo `user:jenkins` với quyền `admin` trên `Gitlab` và lấy `Access Token`:
        - `User` với quyền `admin` có thể kéo bất kỳ `repo` nào trên `gitlab` để triển khai.
        - Chọn `scope: api` cho `Access Token`.

    - Trên `jenkins`, vô `Manage Jenkins > System > Gitlab`, tiến hành thiết lập:
        - `Connection name`: Name that will be used in Jenkins to refer to this GitLab connection.
        - `Gitlab host URL`: URL of the GitLab server.
        - Tạo `Credentials` với `Kind: Gitlab API Token và nhập`Access Token` ở bước trên. Dùng
        `credentails` mới vừa được tạo.

    - `Test Connection` thử xem `Success` hay không?

5. Tạo pipeline:
    - Tạo `folder` để tổ chức các `pipelines` dành riêng cho một dự án: `Dashboard > New Item > Folder`.

    - Trong `folder` mới tạo, tạo một `pipeline` mới bằng cách `New Item > Pipeline`, và chú ý những
    trường thông tin sau
        - `Discard old builds`.
        - `Gitlab Connection`: kiểm tra xem chọn đúng `connection` không?
        - `Build when a change is pushed to Gitlab`.
        - `Pipeline`: chọn `Pipeline script fron SCM`.
            - `SCM`: Git
            - Repository URL: URL đến Repository
            - `Credentials`: Tạo mới và sử dụng `credentials` với `Username with password`.
            - `Branch Specifier`: Branch nào để  `jenkins` trigger `build`.

6. Cấu hình `Gitlab Webhook` kết nối đến `jenkins`:
    - Trên `jenkins`, chọn `Admin > Security > API Token > Add new Token > Generate` và copy `Token`.
    - Trên `Gitlab`:
        - chọn `Admin > Settings > Network > Outbound requests` và stick chọn `Allow requests to the
        local network from web hooks and services`.
        - chọn `<Gitlab Repo> > Settings > Webhooks`:
            - URL: `http://<jenkins-user>:<token>@<jenkins-url>/project/<project-path>`. E.g.:
            `http://admin:11c093bcfb3c60c4febfb8ba155d0b22a6@jenkins-server.remote/project/gitlab-actions/shoeshop`
            - Trigger: Chọn những `event` để trigger.
            - Nếu dùng `http` thì bỏ chọn `Enable SSL verification`.
            - `Test` thử `webhook`.

7. Thiết lập thư mục lưu trữ và những phân quyền cần thiết để chạy `pipeline`:

    **Chú ý**: Trong thực tế, mình sẽ dùng `user` dành riêng cho mỗi dự án chứ không dùng `user` của
    `jenkins`.

    - Tạo thư mục `data`:

        ```bash
        makedir -p /datas/<project-name>
        ```

    - Tạo `user` cho dự án:

        ```bash
        adduser <user-name>
        ```

    - Cấp quyền cho user `jenkins` và cho `jenkins` sử dụng `sudo` không cần nhập password:

        ```bash
        visudo

        ```

        ```ini
        # User privilege specification
        # Grants the gitlab-runner user permission to run any cp command (copy) without a password.
        jenkins ALL=(ALL) NOPASSWD: /bin/cp*
        # Grants the gitlab-runner user permission to run any chown command (change file ownership)
        # without a password
        jenkins ALL=(ALL) NOPASSWD: /bin/chown*
        # Grants the gitlab-runner user permission to run any chmod command (change file permission)
        # without a password
        jenkins ALL=(ALL) NOPASSWD: /bin/chmod*
        # Grants the gitlab-runner user permission to run any kill command to kill running process
        # without a password
        jenkins ALL=(ALL) NOPASSWD: /bin/chmod*
        # Grants the gitlab-runner user permission to run the su command for any user starting with
        # shoeshop  without a password.
        # Lệnh này đảm bảo mình sẽ dùng user riêng cho dự án để triển khai dự án.
        jenkins ALL=(ALL) NOPASSWD: /bin/su <user-name>*
        ```

8. Viết `Jenkinsfile`

    ```groovy
        pipeline {
        // This defines the agent that will be used for executing the pipeline steps
        // In this case, the agent is a Jenkins node with the label 'dev-server'
        agent {
            label 'dev-server'
        }

        // This defines environment variables that will be available throughout the pipeline execution
        environment {
            // Defining application-specific variables for use in various stages
            appUser = "shoeshop"  // The user under which the application will run
            appName = "shoe-ShoppingCart"  // The name of the application
            appVersion = "0.0.1-SNAPSHOT"  // The version of the application
            appType = "jar"  // Type of the application (JAR file in this case)

            // Constructing the process name, which combines the appName, appVersion, and appType
            processName = "${appName}-${appVersion}.${appType}"

            // Folder location for deploying the application
            folderDeploy = "/datas/${appUser}"

            // The command to build the project using Maven
            buildScript = "mvn clean install -DskipTests=true"

            // Script to copy the generated JAR file to the deploy folder
            copyScript = "sudo cp target/${processName} ${folderDeploy}"

            // Script to set the correct permissions for the deploy folder
            permsScript = "sudo chown -R ${appUser}. ${folderDeploy}"

            // Script to kill the running application if it's already running
            killScript = """if [ -n "\$(ps -ef | grep ${processName} | grep -v grep)" ]; then sudo kill -9 \$(ps -ef | grep ${processName} | grep -v grep | awk '{print \$2}'); fi"""

            // Script to run the application in the background (nohup) as the specified user
            runScript = 'sudo su ${appUser} -c "cd ${folderDeploy}; java -jar ${processName} > nohup.out 2>&1 &"'
        }

        // The 'stages' block defines the different stages of the pipeline
        stages {

            // 'build' stage: responsible for building the project
            stage('build') {
                steps {
                    // Executes the Maven build command (ignores tests to speed up the build)
                    sh(script: """ ${buildScript} """, label: "build with maven")
                }
            }

            // 'deploy' stage: responsible for deploying the application
            stage('deploy') {
                steps {
                    // Copies the built JAR file from the 'target' directory to the deployment folder
                    sh(script: """ ${copyScript} """, label: "copy the .jar file into deploy folder")

                    // Sets the correct permissions for the deploy folder so the app user can access it
                    sh(script: """ ${permsScript} """, label: "set permission folder")

                    // Terminates any running process of the application if it's already running
                    sh(script: """ ${killScript} """, label: "terminate the running process")

                    // Runs the application as the appUser, in the background using 'nohup'
                    sh(script: """ ${runScript} """, label: "run the project")
                }
            }
        }

    }
    ```

**Chú ý**:

- Không để cùng một branch mà chạy cả `Gitlab CI` và `Jenkins`. Như vậy chưa hay.
- Với `Jenkins` trong dự án thực tế, cân nhắc không nên chạy `Push events` trong `Webhook`. Vì không
nên `push` một vài commit nhỏ nhỏ mà lại chạy lại cả hệ thống.

## Jenkins CI/CD (Continuos Delivery)

### Tại sao cần Continous Delivery

Có nhiều lý do tại sao bước triển khai cần phải thực hiện thủ công:

- Cần có một người xác nhận rằng code đảm bảo chất lượng trước khi được triển khai lên môi trường cao
hơn như: staging, production, etc, ...
- Ở những dự án quy mô lớn, với hàng ngàn hoặc lên tới chục triệu người dùng. Nếu triển khai hoàn toàn
tự động đôi khi sẽ có rủi ro, làm lỗi hệ thống => Ảnh hưởng đến người dùng => Ảnh hưởng đến uy tín công ty.

**Chú ý**: không có bất kỳ quy trình triển khai nào gọi là chuẩn. Vì sẽ tuỳ thuộc vào từng doanh nghiệp.
Vì vậy tuỳ theo từng doanh nghiệp, cần phải xây dựng một quy trình **_phù hợp với doanh nghiệp_**,
**_nhanh(tự động được nhiều nhất)_** và **_chất lượng_**.

### Viết `Jenkinsfile` đơn giản

```groovy
pipeline {
    agent {
        label 'lab-server'
    }
    environment {
        appUser = "shoeshop"
        appName = "shoe-ShoppingCart"
        appVersion = "0.0.1-SNAPSHOT"
        appType = "jar"
        processName = "${appName}-${appVersion}.${appType}"
        folderDeploy = "/datas/${appUser}"
        buildScript = "mvn clean install -DskipTests=true"
        copyScript = "sudo cp target/${processName} ${folderDeploy}"
        permsScript = "sudo chown -R ${appUser}. ${folderDeploy}"
        killScript = "sudo kill -9 \$(ps -ef| grep ${processName}| grep -v grep| awk '{print \$2}')"
        runScript = 'sudo su ${appUser} -c "cd ${folderDeploy}; java -jar ${processName} > nohup.out 2>&1 &"'
    }
    stages {
        stage('build') {
            steps {
                sh(script: """ ${buildScript} """, label: "build with maven")
            }
        }
        stage('deploy') {
            steps {
                script {
                    try {
                        timeout(time: 5, unit: 'MINUTES') {
                            env.useChoice = input message: "Can it be deployed?",
                                parameters: [choice(name: 'deploy', choices: 'no\nyes', description: 'Choose "yes" if you want to deploy!')]
                        }
                        if (env.useChoice == 'yes') {
                            sh(script: """ ${copyScript} """, label: "copy the .jar file into deploy folder")
                            sh(script: """ ${permsScript} """, label: "set permission folder")
                            sh(script: """ ${killScript} """, label: "terminate the running process")
                            sh(script: """ ${runScript} """, label: "run the project")
                        }
                        else {
                            echo "Do not confirm the deployment!"
                        }
                    } catch (Exception err) {

                    }
                }

            }
        }
    }
}
```

## Jenkins CI/CD parameter

`Jenkins CI/CD` nâng cao phù hợp triển khai môi trường `Production`. Cách triển khai này phù hợp cho
những hệ thống từ **_vừa đến lớn_**.

Sự khác nhau trong tư duy giữa `Dev` và `DevOps`, hay người làm `System`:

- Dev: Chạy là được. Tư tưởng quản trị và bảo vệ hệ thống còn khá yếu. Khi code xong feature
mới chỉ quan tâm đến chức năng mới chạy là được, nhưng nếu hệ thống với chức năng đó gặp lỗi thì sao?
- DevOps: Khôi phục được thì mới triển khai. Để đảm bảo hệ thống luôn hoạt động.

Khi có một hành động gì tác động đến triển khai dự án, đều sẽ nên thực hiện thủ công như: start, stop,
restart, update.

### Sử dụng default choice parameter

- Tương tự như tạo `pipeline` ở trên.
- Sử dụng tuỳ chọn `This project is parameterized`, và tạo `parameters`:
  - `Choice Parameter`:
    - Name: server
    - Choices: dev-server, dev-server-1, dev-server-2
  - `Choice Parameter`:
    - Name: action
    - Choices: start, stop, restart, update

Tuy nhiên, cách này quá đơn giản, không phù hợp với dự án thực tế.

### Cài đặt sử dụng active choices

`Jenkins` là công cụ mạnh mẽ vì tích hợp được nhiều nền tảng và nhiều `plugins` hỗ trợ.

Cài đặt `Active Choices` plugin.

Sử dụng `Active Choices Parameter` với `groovy script` và các tuỳ chọn khác. Nên tick chọn `Use
Groovy Sandbox`.

### Chức năng start, stop, upcode

Sử dụng `Active Choices Parameter`:

- Name: action.
- Script: `return ["start", "stop", "upcode"]`.

Khi thực hiện chức năng `upcode`, mình cần:

- Backup.
- Stop.
- Checkout to specific `commit hash`. Để chỉ định `commit hash` cần sử dụng `String Parameter`.
- Build.
- Start.

### Chức năng backup và rollback

Trước khi `upcode` những `thay đổi mới trong source code` thì cần phải phải `backup`, để đảm bảo có
thể khôi phục lại hệ thống `rollback` trở về trạng thái cũ nếu có lỗi xảy ra.

Sẽ có nhiều phương pháp `backup` khác nhau và tối ưu hơn (ví dụ: dùng `commit hash`, `docker`). Tuy
nhiên, trong bài này mình sẽ sử dụng phương pháp `backup` đơn giản như sau:

- Nén tất cả `files` cần thiết để triển khai hệ thống thành `file zip`.
- Version `file zip` bằng cách đặt tên theo cấu trúc `<app-name>-<ddMMyyyy>-<HHmm>`.
- Lưu trữ trong thư mục dành riêng cho `backup`. Ví dụ: `/datas/shoeshop/backups/`.

Để lựa chọn được `version` để `rollback`. Mình cần sử dụng `Active Choices Reactive Parameters` để
tự động _tuỳ chỉnh_ `parameter` theo giá trị những `parameter khác`:

- Name: version_rollback.
- Groovy Script:

    ```groovy
    // Import Jenkins core model classes to interact with Jenkins internals
    import jenkins.model.*
    // Import FilePath class from Hudson to handle file operations across local/remote nodes
    import hudson.FilePath

    // Define the path where backups are stored for a specific project
    // The path is hardcoded to a directory structure: /datas/<project-name>/backups
    // Replace <project-name> with the actual project name in practice
    backupPath = "/datas/<project-name>/backups"

    // Retrieve the Jenkins node (server) by its name, where 'server' is a variable representing the target node's name
    // Jenkins.getInstance() gets the running Jenkins instance
    def node = Jenkins.getInstance().getNode(server)

    // Create a FilePath object for the remote backup directory
    // node.getChannel() provides the communication channel to the specified node (useful for remote operations)
    // The FilePath constructor allows file operations on the remote node at the specified backupPath
    def remoteDir = new FilePath(node.getChannel(), "${backupPath}")

    // List all files in the remote backup directory
    // remoteDir.list() returns an array of FilePath objects representing the files in the directory
    def files = remoteDir.list()

    // Extract just the names of the files from the FilePath objects
    // Transform the array of FilePath objects into a list of filenames (strings)
    def nameFile = files.collect { it.name }

    // Check if the 'action' variable (assumed to be defined elsewhere) equals "rollback"
    // This suggests the script can perform different operations based on the value of 'action'
    if (action == "rollback") {
        // If action is "rollback", return the list of filenames
        // This could be used to display available backups for rollback purposes
        return nameFile
    }

    ```

- Referenced parameters: server, action. Để `Groovy Script` ở trên hiểu được giá trị của `server` và `node`.
- `Approve signatures` ở trong `Manage Jenkins > ScriptApprove` để chạy được code.

### Pipeline script

```groovy
// Define application-specific variables
appUser = "shoeshop"                          // The Linux user under which the application runs
appName = "shoe-ShoppingCart"                 // Name of the application
appVersion = "0.0.1-SNAPSHOT"                 // Version of the application
appType = "jar"                               // Type of the application artifact (JAR file in this case)
processName = "${appName}-${appVersion}.${appType}"  // Construct the full name of the process artifact (e.g., shoe-ShoppingCart-0.0.1-SNAPSHOT.jar)
folderDeploy = "/datas/${appUser}/run"        // Deployment directory where the app runs
folderBackup = "/datas/${appUser}/backups"    // Directory where backups are stored
folderMain = "/datas/${appUser}"              // Parent directory for deployment and backups
buildScript = "mvn clean install -DskipTests=true"  // Maven command to build the project, skipping tests
copyScript = "sudo cp target/${processName} ${folderDeploy}"  // Command to copy the built JAR to the deployment folder
permsScript = "sudo chown -R ${appUser}. ${folderDeploy}"  // Command to set ownership of deployment folder to appUser
runScript = "sudo su ${appUser} -c 'cd ${folderDeploy}; java -jar ${processName} > nohup.out 2>&1 &'"  // Command to run the JAR as appUser in the background

// Replace <git_url_to_repository> with the actual repository url in practice
gitLink = <git_url_to_repository>  // Git repository URL for the project

// Function to retrieve the process ID (PID) of the running application
def getProcessId(processName) {
    // Execute a shell command to find the PID of the process by grepping for the processName
    // ps -ef lists all processes, grep filters for processName, awk extracts the PID
    def processId = sh(returnStdout:true, script: """ ps -ef| grep ${processName}| grep -v grep| awk \'{print \$2}\' """, label: "get processId")
    return processId.trim()  // Return the PID after trimming whitespace
}

// Function to start the application process
def startProcess() {
    stage('start') {  // Define a Jenkins pipeline stage named "start"
        // Execute the runScript to start the application as the appUser
        sh(script: """ ${runScript} """, label: "run the project")
        sleep 5  // Wait for 5 seconds to allow the process to start
        def processId = getProcessId("${processName}")  // Check if the process is running by getting its PID
        if ("${processId}" == "") {  // If no PID is found, the process failed to start
            error("Cannot start process")  // Fail the pipeline with an error message
        }
    }
    // Log a message indicating the application started successfully on the specified server
    echo("${appName} with server " + params.server + " started")
}

// Function to stop the application process
def stopProcess() {
    stage('stop') {  // Define a Jenkins pipeline stage named "stop"
        def processId = getProcessId("${processName}")  // Get the PID of the running process
        if ("${processId}" != "") {  // If a PID exists, the process is running
            // Kill the process using SIGKILL (-9) to forcefully stop it
            sh(script: """ sudo kill -9 ${processId} """, label: "kill process")
        }
        // Log a message indicating the application was stopped on the specified server
        echo("${appName} with server " + params.server + " stopped")
    }
}

// Function to update the application code and deploy a new version
def upcodeProcess() {
    stage('checkout') {  // Define a Jenkins pipeline stage named "checkout"
        if (params.hash == "") {  // Check if the Git commit hash is provided (required for code update)
            error("require hashing for code update.")  // Fail the pipeline if hash is missing
        }
        // Checkout the specific Git commit using the provided hash
        checkout([$class: 'GitSCM', branches: [[name: params.hash]],
                  userRemoteConfigs: [[credentialsId: 'jenkins-gitlab-user-account', url: gitLink]]])
    }
    stage('build') {  // Define a Jenkins pipeline stage named "build"
        // Build the project using Maven (buildScript skips tests for faster builds)
        sh(script: """ ${buildScript} """, label: "build with maven")
    }
    stage('config') {  // Define a Jenkins pipeline stage named "config"
        // Copy the built JAR file to the deployment folder
        sh(script: """ ${copyScript} """, label: "copy .jar file into the deploy folder")
        // Set appropriate permissions on the deployment folder for the appUser
        sh(script: """ ${permsScript} """, label: "assign project permissions")
    }
}

// Function to create a backup of the current deployment
def backupProcess() {
    stage('backup') {  // Define a Jenkins pipeline stage named "backup"
        // Create a timestamp in the format DDMMYYYY_HHMM (e.g., 03022024_1011)
        def timeStamp = new Date().format("ddMMyyyy_HHmm")
        // Construct the backup filename using the appName and timestamp (e.g., shoe-ShoppingCart_03022024_1011.zip)
        def zipFileName = "${appName}_${timeStamp}.zip"
        // Zip the contents of the deployment folder and store the backup in the backup folder
        // The command runs as appUser to ensure proper permissions
        sh(script: """ sudo su ${appUser} -c "cd ${folderMain}; zip -jr ${folderBackup}/${zipFileName} ${folderDeploy}" """, label: "backup old version")
    }
}

// Function to rollback to a previous version using a backup
def rollbackProcess() {
    stage('rollback') {  // Define a Jenkins pipeline stage named "rollback"
        // Delete all files in the deployment folder to clear the current version
        sh(script: """ sudo su ${appUser} -c "cd ${folderDeploy}; rm -rf *" """, label: "delete the current version")
        // Unzip the specified backup (params.rollback_version) into the deployment folder
        sh(script: """ sudo su ${appUser} -c "cd ${folderBackup}; unzip ${params.rollback_version} -d ${folderDeploy}" """, label: "rollback process")
    }
}

// Main pipeline execution block, targeting a specific Jenkins node
node(params.server) {  // Run the pipeline on the node specified in params.server
    currentBuild.displayName = params.action  // Set the Jenkins build display name to the action being performed
    if (params.action == "start") {  // If the action is "start", start the application
        startProcess()
    }
    if (params.action == "stop") {  // If the action is "stop", stop the application
        stopProcess()
    }
    if (params.action == "upcode") {  // If the action is "upcode", perform a full code update and deployment
        currentBuild.description = "server " + params.server + " with hash " + params.hash  // Add a description to the build
        backupProcess()  // Backup the current version
        stopProcess()    // Stop the running application
        upcodeProcess()  // Update the code and deploy the new version
        startProcess()   // Start the new version
    }
    if (params.action == "rollback") {  // If the action is "rollback", revert to a previous version
        stopProcess()    // Stop the running application
        rollbackProcess()  // Rollback to the specified backup version
        startProcess()   // Start the rolled-back version
    }
}

```

## Ứng dụng khác của Jenkins

`Jenkins` không chỉ dùng để làm `CI/CD` mà bất cứ công việc nào có thể tự động hoá thì công cụ này
đều làm được.

Có nhiều kiểu Checklog theo số dòng, theo keyword, theo ngày tháng, etc:

- Dùng `String Parameter` để lấy thông tin về số dòng, keyword.
- Dùng `Active Choices Parameter` để quyết định kiểu (`type`) của checklog.

### Checklog

Thay vì lên `server` và gõ các `command` một cách thủ công để `checklog` từng dự án, thì mình có thể
dùng `jenkins` để tự động hoá.

## Kinh nghiệm lựa chọn `Github Actions`, `Gitlab CI`, hay `Jenkins` cho công ty và dự án?

Theo kinh nghiệm của tác giả:

1. Cả 3 công cụ Jenkins, Gitlab CI, Github Action đều được sử dụng ưa chuộng trong các doanh nghiệp từ lớn đến nhỏ.

2. Đối với những dự án/doanh nghiệp nhỏ chưa có nhiều nguồn lực, chưa có nhiều ngân sách sẽ ưu tiên Github Action hơn vì đầu tiên có thể tận dụng nền tảng Github lưu trữ giảm đi phần việc quản trị và vẫn đạt được mục đích "tự động".  Tuy nhiên chắc chắn rồi vẫn có các doanh nghiệp vừa sử dụng nhưng thật sự những doanh nghiệp lớn anh làm và hỗ trợ chưa từng sử dụng Github Action (anh xin nhắc lại đây chỉ là kinh nghiệm cá nhân nhé).

3. Những doanh nghiệp sử dụng Jenkins và Gitlab CI là tất cả các loại dự án/doanh nghiệp từ nhỏ đến lớn. Đề cao sự bảo mật, tính toàn vẹn, và theo quy định luật pháp hiện hành (có những doanh nghiệp còn bắt buộc không được sử dụng nền tảng lưu trữ thứ 3 đừng nói đến công cụ lưu trữ source code dùng sẵn như Github). Và những doanh nghiệp lớn như ngân hàng, chứng khoán thường ưu tiên sử dụng Jenkins, họ xây dựng 1 nền tảng riêng dành cho pipeline kết hợp với các quy trình/công cụ tạo thành pipeline DevSecOps với nhiều khâu quan trọng mà chắc chắn Github Action không bao giờ - anh nhấn mạnh Không bao giờ làm nổi.

=> Cuối cùng vẫn tùy thuộc vào doanh nghiệp nhưng anh xin tóm lại những dự án/doanh nghiệp lớn anh từng trải nghiệm chưa doanh nghiệp nào sử dụng Github lưu trữ và Github Action.
