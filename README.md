# Analyses statiques

Nous allons utiliser le projet https://github.com/nicolas-sanch/minimalist-blog-laravel pour mettre en oeuvre différents outils d'analyse statique.

## 1 - PHP CodeSniffer
Sources : https://les-enovateurs.com/php-codesniffer-loutil-ultime-pour-valider-et-corriger-votre-code-php

Pour installer l'outil :
```bash
docker run --rm --interactive --tty \
  --volume $PWD:/app \
  composer require "squizlabs/php_codesniffer=*" --dev
```

Pour l'utiliser :
```bash
vendor/bin/sail php ./vendor/bin/phpcs -p ./app/Models # Pour analyser le dossier app/Models
```
phpcs analyse les fichiers, détecte les violations des standards du développement PHP <br>
phpcbf permet de corriger certaines erreurs automatiquement <br>

Pour corriger l'erreur _Missing file doc comment_: https://stackoverflow.com/questions/54476894/any-way-to-ignore-missing-file-doc-comment-yii2-using-php-strom

## 2 - Copy/Paste Detector

Détecte les copier/coller de code

Sources : https://github.com/sebastianbergmann/phpcpd et https://packagist.org/packages/sebastian/phpcpd

Pour installer l'outil :
```bash
docker run --rm --interactive --tty \
  --volume $PWD:/app \
  composer require "sebastian/phpcpd" --dev
```
Pour l'utiliser
```vendor/bin/sail php ./vendor/bin/phpcpd ./app/Models # Pour analyser le dossier app/Models```

## Check dependencies

Le CEO de Symfony maintient un outil permettant de checker les vulnérabilités connues présentes dans nos librairies. <br>
Sources : https://github.com/fabpot/local-php-security-checker

## PHPMetrics

Pour générer et visualiser les métriques de notre code, nous utilison PHPMetrics : https://phpmetrics.github.io/PhpMetrics/ <br>

Pour l'installer :
```bash
docker run --rm --interactive --tty \
  --volume $PWD:/app \
  composer require "phpmetrics/phpmetrics" --dev 
```

Pour générer :
```vendor/bin/sail php ./vendor/bin/phpmetrics --report-html=myreport ./app```

Il est ensuite nécessaire d'ouvrir le fichier myreport/index.html dans notre navigateur

## Gitlab-ci

Intégrer les précédentes analyses ainsi que nos tests unitaires à notre intégration continue nous permet d'améliorer la qualité de notre code source en production. <br>

Doc : https://docs.gitlab.com/ee/ci/examples/laravel_with_gitlab_and_envoy/ <br>

Pour générer un pipeline .gitalab-ci :
* Créer un projet Gitlab
* Créer un fichier .gitlab-ci.yml à la racine du projet
* Push

Voici un exemple de fichier .gitlab-ci.yml
```yml
image: laravelsail/php81-composer

variables:
  MYSQL_DATABASE: minimalist_blog_laravel
  MYSQL_ROOT_PASSWORD: root
  DB_HOST: mysql

stages:
  - preparation
  - build
  - test

before_script:
  - docker-php-ext-install pdo_mysql

composer:
  stage: preparation
  script:
    - php -v
    - composer install --prefer-dist --no-ansi --no-interaction --no-progress --no-scripts
    - cp .env.example .env
    - php artisan key:generate
  artifacts:
    paths:
      - vendor/
      - .env
    expire_in: 1 days
    when: always
  cache:
    paths:
      - vendor/

db-seeding:
  stage: build
  services:
    - mysql:latest
  dependencies:
    - composer
  script:
    - php artisan migrate:fresh --seed
  artifacts:
    paths:
      - storage/logs # for debugging
      - db.sql
    expire_in: 1 days
    when: always

unit_test:
  stage: test
  services:
    - mysql:latest
  dependencies:
    - composer
    - db-seeding
  script:
    - vendor/bin/phpunit
  artifacts:
    paths:
      - ./storage/logs # for debugging
    expire_in: 1 days
    when: on_failure

codestyle:
  stage: test
  dependencies: []
  script:
    - composer require "squizlabs/php_codesniffer=*" --dev
    - php vendor/bin/phpcs --standard=PSR2 --ignore=app/Support/helpers.php app
  allow_failure: true

phpcpd:
  stage: test
  script:
    - test -f phpcpd.phar || curl -L https://phar.phpunit.de/phpcpd.phar -o phpcpd.phar
    - php phpcpd.phar app/ --min-lines=50
  dependencies: []
  cache:
    paths:
      - phpcpd.phar
  allow_failure: true

check-deps:
  stage: test
  script:
    - curl -L  https://github.com/fabpot/local-php-security-checker/releases/download/v1.2.0/local-php-security-checker_1.2.0_linux_386 --output local-php-security-checker
    - chmod +x local-php-security-checker
    - ./local-php-security-checker --format=junit --path=./composer.lock > security-checker-report.xml
  artifacts:
    reports:
      junit:
        - security-checker-report.xml
  allow_failure: true
```
