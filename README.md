# GitHub Action: Сборка Maven-проекта и публикация Docker-образа

![GitHub Actions](https://img.shields.io/github/actions/workflow/status/demid1984/maven-build-docker-action/main.yml?logo=github&style=flat-square)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=flat&logo=docker&logoColor=white)
![Maven](https://img.shields.io/badge/apache_maven-C71A36?style=flat&logo=apachemaven&logoColor=white)

Composite Action для автоматической сборки Maven-проекта в Docker-образ с последующей публикацией в указанный Docker Registry (Docker Hub, GitHub Container Registry, GitLab, приватный реестр и т.д.).

## 🎯 Возможности

- ✅ Автоматическое извлечение `artifactId` и `version` из `pom.xml`
- ✅ Поддержка двух сборщиков образов:
    - `spring` — сборка через `spring-boot:build-image` (встроенный buildpack)
    - `jib` — сборка через `jib:build` (Google Jib)
- ✅ Аутентификация в Docker Registry по `username`/`password`
- ✅ Тегирование образа двумя тегами: `version` и `latest`
- ✅ Гибкая настройка Maven-сборки:
    - кастомный `settings.xml`
    - Maven профиль (`-P`)
    - произвольные аргументы (`-D`, `-pl`, и т.д.)

## 📋 Требования

- Проект должен быть Maven-проектом с корректно определёнными `artifactId` и `version` в `pom.xml`
- В корне репозитория должен присутствовать `./mvnw` (Maven Wrapper)
- Для `spring` режима: используется Spring Boot 2.3+ с включённым buildpack support (в `pom.xml` должен быть плагин `spring-boot-maven-plugin`)
- Для `jib` режима: подключен плагин `jib-maven-plugin` (например, `com.google.cloud.tools:jib-maven-plugin`)
- Docker **не требуется локально**, action использует `docker/login-action`, который интегрируется с GitHub-hosted runner’ом

> ⚠️ **Важно**:
> Для приватных Maven-репозиториев укажите `m2-settings` и используйте `docker-password` как `GITHUB_TOKEN` или PAT с соответствующими правами.

## 🛠️ Использование

### Пример: Публикация образа в Docker Hub

```yaml
name: Build and push Docker image

on:
  push:
    branches: [ main ]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build and publish Docker image
        uses: demid1984/maven-build-docker-action@v0.0.3
        with:
          registry: "docker.io"
          registry-repository: "myrepo"
          docker-username: ${{ secrets.DOCKER_USERNAME }}
          docker-password: ${{ secrets.DOCKER_PASSWORD }}
          image-builder: "spring"
          m2-settings: ".m2/settings.xml"
          profile-name: "prod"
          maven-args: "-Dmy.custom.prop=value"
```

### Пример: Публикация в GitHub Container Registry (ghcr.io)

```yaml
- name: Publish to ghcr.io
  uses: demid1984/maven-build-docker-action@v0.0.3
  with:
    registry: "ghcr.io"
    registry-repository: "${{ github.repository }}"
    docker-username: "${{ github.actor }}"
    docker-password: "${{ secrets.GITHUB_TOKEN }}"
    image-builder: "jib"
```

> 💡 **Совет**: Для `ghcr.io` убедитесь, что репозиторий имеет публичный доступ *или* настроены права на запись для `GITHUB_TOKEN`.

## 📥 Входные параметры

| Параметр | Обязательный | По умолчанию | Описание |
|----------|--------------|--------------|----------|
| `registry` | ✅ Да | — | Адрес Docker Registry (например, `docker.io`, `ghcr.io`, `registry.gitlab.com`) |
| `registry-repository` | ✅ Да | — | Имя репозитория в реестре (без тега, без `https://`) |
| `docker-username` | ❌ Нет | — | Имя пользователя для аутентификации в реестре (используется `docker/login-action@v2`) |
| `docker-password` | ❌ Нет | — | Пароль/токен для аутентификации (рекомендуется использовать `secrets.DOCKER_PASSWORD`) |
| `image-builder` | ✅ Да | — | Тип сборщика: `"spring"` или `"jib"` |
| `m2-settings` | ❌ Нет | — | Путь к кастомному `settings.xml` (например, `.github/maven/settings.xml`) |
| `profile-name` | ❌ Нет | — | Имя активируемого Maven-профиля (аналог `-P`) |
| `maven-args` | ❌ Нет | — | Дополнительные аргументы Maven (например, `-DskipITs -pl :my-module`) |

## 📤 Выходные параметры

| Параметр | Описание |
|----------|----------|
| `version` | Версия проекта, извлечённая из `pom.xml` (например, `1.2.3`) |
| `application` | `artifactId` проекта (например, `my-service`) |

> 📌 Эти параметры доступны в последующих шагах через `steps.<id>.outputs.version` и `steps.<id>.outputs.application`.

## ⚙️ Под капотом

1. Выполняется `checkout` текущей ветки (`github.ref`) с полной историей.
2. Считывается `project.version` и `project.artifactId` через Maven Help Plugin.
3. Происходит login в указанный registry (`docker/login-action@v2`).
4. В зависимости от `image-builder`:
    - **`spring`** → `./mvnw spring-boot:build-image -DskipTests ...`
      (создаёт образ в формате `image:version`, теги не управляются явно)
    - **`jib`** → `./mvnw clean compile jib:build -DskipTests ...`
      (bild в реестр напрямую, без сборки локального образа)
5. После сборки:
    - Принудительно тегируется образ как `:latest`
    - Пушатся **оба тега**: `:version` и `:latest`

> 🔍 *Обратите внимание*:
> `spring-boot:build-image` **напрямую пушит** в registry, а `jib:build` — тоже делает push, но без локального образа.
> Чтобы обеспечить тегирование `:latest`, action использует `docker tag` + `docker push`.

## 📜 Лицензия

Материалы распространяются под лицензией [MIT](LICENSE). См. файл `LICENSE` для подробной информации.

---

## 🤝 Вклад в проект

Приветствуются PR и issues!
Следуйте стандартам: проверяйте форматирование, добавляйте тесты, описывайте изменения.

---

© 2026, demid1984
Сделано с ❤️ для надёжных CI/CD потоков.
