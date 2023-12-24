## для образа gitlab:
- hostname: указываем реальный белый ip
- GITLAB_OMNIBUS_CONFIG тут в системных константах для external_url
  указываем домен сайта ('https://your-domen.ru:443' - если домен доступен по https
  или 'http://your-domen.ru' - если домен доступен по http)
- ну забываем прокинуть порты для gitlab
```yaml
    ports:
      - "443:443"
      - "80:80"
      - "465:465"
      - "2224:22"
```

## для настройки smtp в gitlab
заходим в конфигурационный файл гитлаба ./data/etc/gitlab/gitlab.rb

и правим нужные нам строчки
```
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.yandex.ru"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "yourmail@email.ru"
# указываем пароль для почтового приложения из яндекс
gitlab_rails['smtp_password'] = "password"
gitlab_rails['smtp_domain'] = "yandex.ru"
gitlab_rails['smtp_authentication'] = "login"
# gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
gitlab_rails['gitlab_email_from'] = "yourmail@email.ru"
gitlab_rails['gitlab_email_reply_to'] = "yourmail@email.ru"
# gitlab_rails['smtp_pool'] = false

###! **Can be: 'none', 'peer', 'client_once', 'fail_if_no_peer_cert'**
###! Docs: http://api.rubyonrails.org/classes/ActionMailer/Base.html
gitlab_rails['smtp_openssl_verify_mode'] = 'none'
```

## для создания runners

заходим в контейнер с ранером
```bash
docker exec -it <runner_container_name> bash
```
внутри контейнера раннера регистрируем раннер (токен генерится в project/ci/cd settings/)
```bash
gitlab-runner register \
    --non-interactive \
    --registration-token <token-name> \
    --locked=false \
    --description <runner-name> \
    --url <https://gitlab-url> \
    --executor docker \
    --docker-image docker:stable \
    --docker-volumes "/var/run/docker.sock:/var/run/docker.sock"
```

после создания раннера выходим из контейнера и перезагружаем gitlab через docker-compose restart


## деплой по ssh

создаем ssh ключ(если его нет) внутри контейнера с gitlab
* remote_user - пользователь удаленного сервера
* remote_host - адрес удаленного сервера
```bash
docker exec -it gitlab bash
ssh-keygen
ssh-copy-id remote_user@remote_host
```
- значения ключей сохраняем в переменные для деплоя (GTILAB_PRIVATE_KEY GTILAB_PUB_KEY) внутри гитлаб(тоесть через само приложение)
- в переменные REMOTE_SERVER_USER и REMOTE_SERVER_ADDR сохраняем пользователя и адрес удаленного сервера

- пример файла с деплоем через git pull:
```yaml
stages:
  - test-deploy
  - deploy

lint-test-job:
  stage: test-deploy
  script:
    - echo "test 27"
    - sleep 6

deploy-job:
  stage: deploy
  only:
    - main
  before_script:
    - apk add openssh-client
    - eval $(ssh-agent -s)
    - echo "$GTILAB_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh

  script:
    - ssh -o StrictHostKeyChecking=no $REMOTE_SERVER_USER@$REMOTE_SERVER_ADDR "cd /var/docker/ci-cd-test-project/src && git pull"

  environment: production
```

- пример файла с деплоем через git pull и сборкой внутри контейнера на удаленном сервере:
```yaml
production-update-job:
  stage: production-update
  only:
    - main
  before_script:
    - apk add openssh-client
    - eval $(ssh-agent -s)
    - echo "$GTILAB_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh

  script:
    - ssh -o StrictHostKeyChecking=no $REMOTE_SERVER_USER@$REMOTE_SERVER_ADDR "cd ${REMOTE_SERVER_PATH}/src/app && git pull"

  environment: production

production-build-job:
  stage: production-build
  only:
    - main
  before_script:
    - apk add openssh-client
    - eval $(ssh-agent -s)
    - echo "$GTILAB_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh

  script:
    - ssh -o StrictHostKeyChecking=no $REMOTE_SERVER_USER@$REMOTE_SERVER_ADDR "cd ${REMOTE_SERVER_PATH} | docker exec --workdir /var/www/app ${APP_CONTAINER_NAME} npm run build"

  environment: production
```