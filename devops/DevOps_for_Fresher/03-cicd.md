# CI/CD

## Tại sao CI/CD lại quan trọng như vậy?

Trong doanh nghiệp, mỗi dự án sẽ có hàng chục source code. Vậy nếu có 20 dự án, thì cần
bao nhiêu nhân lực để kịp thời triển khai các dự án?

Mỗi dự án đều có 3 môi trường phát triển: `development`, `staging` và `production`. Làm sao để kịp thời
triển khai các thay đổi mới ở các môi trường trong mỗi dự án? Không lẽ mỗi khi dev commit một feature
mới lên `development` lại cần phải chờ đợi đến khi được triển khai để kiểm tra được feature đó?

Triển khai thủ công rất tốn thời gian, lặp đi lặp lại và dễ gặp sai lầm.

**CI/CD** giúp tự động hoá được công việc triển khai dự án, giúp tiết kiệm thời gian, nhân lực, và
giảm thiểu sai sót. Bây giờ, `dev` chỉ cần thực hiện commit quá trình triển khai sẽ được CI/CD tự động
hoá: `commit` -> `build` -> `test` -> `deploy`.

## CI/CD là gì?

**CI** là Continuous Integration. CI sẽ thực hiện hiện các công việc: clone code, build, unit test,
integration test, clean code test, ...

**CD** là Continuous deployment và Continuous delivery:
    - Continuous deployment: triển khai hoàn toàn tự động.
    - Continuous delivery: bước triển khai sẽ là thủ công.

**Chú ý**: Cả 2 chiến lược **_CD_** đều có điểm mạnh và điểm yếu. Tự động hoàn toàn giúp tối ưu hiệu
suất, còn thủ công một chút giúp kiểm soát tốt hơn. Tuy nhiên, có thể phát triển ra một chiến lược
tận dụng các điểm mạnh của 2 chiến lược trên: vừa nhanh vừa chất lượng.

**Chú ý**: Có được quy trình CI/CD phù hợp không dễ dàng vì còn phụ thuộc vào văn hoá, quy mô,
công nghệ của doanh nghiệp.

## Gitlab CI/CD

### Các bước tiến hành

1. Cài đặt công cụ tự động: `Gitlab runner`, `Jenkins`, ...
2. Viết file cấu hình công việc: `.gitlab-ci.yml`, `.github/workflows/*.yaml`, ...

### Install và register Gitlab-runner

**Chú ý**: Cài đặt `gitlab-runner` phải trên đúng server (Ví dụ: `dev server`, `staging server`) chứ
không phải trên `gitlab server`.

1. Cài đặt `gitlab-runner` sử dụng `curl` và `apt install gitlab-runner`.
    **Chú ý**: `gitlab-runner` sẽ tạo ra một account mới có tên `gitlab-runner`. Kiểm tra trong
    `/etc/passwd`.

2. Register `runner` sử dụng `gitlab-runner register` với `registration token` lấy từ
`Settings > CI/CD > Runners`.
    - `Executor` chỉ định nơi thực hiện các câu lệnh. Ví dụ `shell excutor` sẽ thực hiện
    các câu lệnh trực tiếp trên `server`, `docker executor` thực hiện các câu lệnh trong `container`.

    - Sẽ có file cấu hình được tạo ra ở `/etc/gitlab-runner/config.toml`. Cấu hình lại
    `concurrent = 4`, có nghĩa là `runner` có khả năng thực hiện đồng thời 4 jobs.

3. Chạy `gitlab-runner`:

    ```bash
    nohup gitlab-runner --working-directory /home/gitlab-runner/ \
                        --config /etc/gitlab-runner/config.toml \
                        --service gitlab-runner --user gitlab-runner 2>&1 &
    ```

4. Thiếp lập `runner` trên `gitlab`. `Edit` `runner` trên `gitlab` và disable `Lock to current
project`, nó cho phép `runner` được phép chạy đồng thời nhiều dự án.

## Continuous Deployment

Tiến hành cài CI/CD để tự động [triển khai dự án bằng Java Spring Boot](01-linux.md#triển-khai-dự-án-java-spring-boot)

1. Thiết lập thư mục lưu trữ và những phân quyền cần thiết để chạy `workflow`:
    **Chú ý**: Trong thực tế, mình sẽ dùng `user` dành riêng cho mỗi dự án chứ không dùng `user` của
    `gitlab-runner`.

    - Tạo thư mục `data`:

        ```bash
        makedir -p /datas/<project-name>
        ```

    - Tạo `user` cho dự án:

        ```bash
        adduser <user-name>
        ```

    - Cấp quyền cho user `github-runner` và cho `github-runner` sử dụng `sudo` không cần nhập password:

        ```bash
        visudo

        ```

        ```ini
        # User privilege specification
        # Grants the gitlab-runner user permission to run any cp command (copy) without a password.
        gitlab-runner ALL=(ALL) NOPASSWD: /bin/cp*
        # Grants the gitlab-runner user permission to run any chown command (change file ownership)
        # without a password
        gitlab-runner ALL=(ALL) NOPASSWD: /bin/chown*
        # Grants the gitlab-runner user permission to run any chmod command (change file permission)
        # without a password
        gitlab-runner ALL=(ALL) NOPASSWD: /bin/chmod*
        # Grants the gitlab-runner user permission to run any kill command to kill running process
        # without a password
        gitlab-runner ALL=(ALL) NOPASSWD: /bin/chmod*
        # Grants the gitlab-runner user permission to run the su command for any user starting with
        # shoeshop  without a password.
        # Lệnh này đảm bảo mình sẽ dùng user riêng cho dự án để triển khai dự án.
        gitlab-runner ALL=(ALL) NOPASSWD: /bin/su <user-name>*
        ```

2. Cấu hình `workflow`:

```yml
# Define global variables that can be used in all jobs
variables:
    project_name: shoe-ShoppingCart        # The name of the project
    version: 0.0.1-SNAPSHOT               # The version of the project
    project_path: /datas/shoeshop          # The path to the project directory on the server
    filename: ${project_name}-${version}.jar   # The name of the generated JAR file using project_name and version
    user: shoeshop                         # The user for deploying and running the application

# Define the stages of the pipeline in order
stages:
    - build                               # First stage: build the project
    - deploy                              # Second stage: deploy the project
    - showlog                             # Third stage: show the logs of the application

# Define the 'build' job
build:
    stage: build                         # This job is part of the 'build' stage
    variables:
        GIT_STRATEGY: clone              # Use 'clone' strategy to clone the repository for this job
    script:
        - mvn install -DskipTests=true   # Run Maven to build the project, skipping tests
    tags:
        - dev-server                     # Use the runner tagged 'dev-server'
    only:
        - tags                           # This job will only run when a tag is pushed (e.g., version tag)

# Define the 'deploy' job
deploy:
    stage: deploy                        # This job is part of the 'deploy' stage
    variables:
        GIT_STRATEGY: none               # No need to clone the repository for the deploy job
    script:
        # Copy the JAR file to the project directory on the server
        - sudo cp target/${filename} ${project_path}

        # Change the ownership of the project directory to the user
        - sudo chown -R ${user}. ${project_path}

        # Set appropriate permissions on the project directory
        - sudo chmod -R 750 ${project_path}

        # If the app is already running, kill the existing process (using the JAR filename to match the process)
        - if [ -n "$(ps -ef | grep ${filename} | grep -v grep)" ]; then sudo kill -9 $(ps -ef | grep ${filename} | grep -v grep | awk '{print $2}'); fi

        # Start the new application in the background with nohup (no hangup)
        - sudo su ${user} -c "cd ${project_path}; nohup java -jar ${filename} > nohup.out 2>&1 &"
    tags:
        - dev-server                     # Use the runner tagged 'dev-server'
    only:
        - tags                           # This job will only run when a tag is pushed (e.g., version tag)

# Define the 'showlog' job
showlog:
    stage: showlog                        # This job is part of the 'showlog' stage
    variables:
        GIT_STRATEGY: none                # No need to clone the repository for the log job
    script:
        # Show the last 1000 lines of the application log file (nohup.out)
        - sudo su ${user} -c "cd ${project_path}; tail -n 1000 nohup.out"
    tags:
        - dev-server                      # Use the runner tagged 'dev-server'
    only:
        - tags                            # This job will only run when a tag is pushed (e.g., version tag)

```

## Continous delivery

### Khi nào cần sử dụng `Continous delivery`?

Trong thực tế, `source code` của mình sẽ cần quét qua các công cụ bảo mật, hoặc clean code trong quá
trình `CI/CD`, và sẽ cần có một người (có đủ quyền) để `kiểm tra` và `munually approve` để tiến hành
deploy dự án lên các môi trường cao hơn (staging, production).

### Cấu hình `workflow`

```yaml
# Global variables to be used throughout the pipeline
variables:
    project_name: shoe-ShoppingCart            # The name of the project
    version: 0.0.1-SNAPSHOT                   # The version of the project
    project_path: /datas/shoeshop              # Path where the project is deployed on the server
    filename: ${project_name}-${version}.jar  # Name of the generated JAR file based on project name and version
    user: shoeshop                             # User under which the application will run

# Define the stages in the pipeline
stages:
    - build                                   # First stage: Build the project
    - deploy                                  # Second stage: Deploy the project
    - showlog                                 # Third stage: Show the application logs

# 'build' job in the build stage
build:
    stage: build                             # This job is part of the 'build' stage
    variables:
        GIT_STRATEGY: clone                  # Use the 'clone' strategy to fetch the repository
    script:
        - mvn install -DskipTests=true       # Run Maven to install dependencies and build, skipping tests
    tags:
        - dev-server                         # The job will run on a runner tagged 'dev-server'
    only:
        - tags                               # This job runs only on tag pushes (typically version tags)

# 'deploy' job in the deploy stage
deploy:
    stage: deploy                            # This job is part of the 'deploy' stage
    variables:
        GIT_STRATEGY: none                   # No need to fetch the repository in the deploy job
    when: manual                             # The deploy job will run manually, triggered by a user
    script:
        # Check if the current GitLab user is <user-name> before proceeding with deployment
        - >
            if [ "${GITLAB_USER_LOGIN}" == '<user-name>' ]; then
                # Copy the generated JAR file to the project path
                sudo cp target/${filename} ${project_path}

                # Change the ownership of the project directory to the deploy user
                sudo chown -R ${user}. ${project_path}

                # Set the correct permissions for the project directory
                sudo chmod -R 750 ${project_path}

                # If the application is already running, kill the old process
                if [ -n "$(ps -ef | grep ${filename} | grep -v grep)" ]; then
                    sudo kill -9 $(ps -ef | grep ${filename} | grep -v grep | awk '{print $2}');
                fi

                # Run the new version of the application in the background
                sudo su ${user} -c "cd ${project_path}; nohup java -jar ${filename} > nohup.out 2>&1 &"
            else
                # If the user is not <user-name>, deny permission and exit
                echo "Permission Denied"
                exit 1
            fi
    tags:
        - dev-server                         # The job will run on a runner tagged 'dev-server'
    only:
        - tags                               # This job runs only on tag pushes (typically version tags)

# 'showlog' job in the showlog stage
showlog:
    stage: showlog                           # This job is part of the 'showlog' stage
    variables:
        GIT_STRATEGY: none                    # No need to fetch the repository in the log job
    when: manual                              # The job will run manually, triggered by a user
    script:
        # Display the last 1000 lines of the 'nohup.out' log file (application logs)
        - sudo su ${user} -c "cd ${project_path}; tail -n 1000 nohup.out"
    tags:
        - dev-server                          # The job will run on a runner tagged 'dev-server'
    only:
        - tags                                # This job r

```

Chỉ có người dùng với `user-name` được cấp quyền trong `CI/CD` mới có thể `manually trigger deploy
job`. Còn với `showlog` thì ai cũng có quyền để `manually trigger`.
