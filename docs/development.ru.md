# Руководство по разработке

> **Язык**: [English](./development.md) | [Русский](./development.ru.md)

## Содержание

- [Предварительные требования](#предварительные-требования)
- [Настройка окружения](#настройка-окружения)
- [Качество кода](#качество-кода)
- [Тестирование](#тестирование)
- [Участие в разработке](#участие-в-разработке)

---

## Предварительные требования

### Необходимые инструменты

| Инструмент | Версия | Назначение |
|------------|--------|------------|
| Python | 3.10+ | Среда выполнения |
| Ansible | 2.17+ | Автоматизация |
| ansible-lint | Последняя | Линтинг кода |
| yamllint | Последняя | Валидация YAML |

### Установка

```bash
# Создание виртуального окружения
python3 -m venv .venv
source .venv/bin/activate

# Установка зависимостей
pip install ansible ansible-lint yamllint pytest-testinfra
```

---

## Настройка окружения

### Клонирование репозитория

```bash
git clone https://github.com/ashimov/rke2-ansible.git
cd rke2-ansible
```

### Установка зависимостей Ansible

```bash
ansible-galaxy collection install -r requirements.yml
```

### Структура проекта

```
rke2-ansible/
├── roles/
│   ├── rke2/
│   │   ├── defaults/main.yml    # Переменные по умолчанию
│   │   ├── tasks/
│   │   │   ├── main.yml         # Основная оркестрация
│   │   │   ├── common/          # Переиспользуемые задачи
│   │   │   ├── config.yml       # Конфигурация
│   │   │   └── ...
│   │   └── handlers/main.yml    # Обработчики событий
│   └── testing/                 # Роль тестирования
├── testing/                     # Инфраструктура интеграционных тестов
├── .github/workflows/           # CI/CD пайплайны
└── docs/                        # Документация
```

---

## Качество кода

### Линтинг

Запуск всех линтеров:

```bash
# Ansible lint (профиль production)
ansible-lint --profile=production ./roles

# YAML lint
yamllint .
```

### Стандарты кода

1. **Имена задач**: Используйте описательные имена в повелительном наклонении
   ```yaml
   - name: Create configuration directory
   ```

2. **Именование переменных**: Используйте `snake_case` с префиксом `rke2_`
   ```yaml
   rke2_config_dir: "/etc/rancher/rke2"
   ```

3. **Внутренние переменные**: Используйте префикс подчёркивания
   ```yaml
   register: _api_status
   ```

4. **Использование модулей**: Предпочитайте `ansible.builtin.systemd` вместо `ansible.builtin.service`

5. **Комментарии**: Используйте заголовки секций для организации
   ```yaml
   # =============================================================================
   # Название секции
   # =============================================================================
   ```

6. **Теги**: Добавляйте соответствующие теги ко всем включениям задач
   ```yaml
   - name: Configure RKE2
     ansible.builtin.include_tasks: configure_rke2.yml
     tags:
       - config
       - hardening
   ```

---

## Тестирование

### Запуск тестов

**Ansible тесты:**

```bash
ansible-playbook testing.yml -i inventory/hosts.yml --skip-tags troubleshooting
```

**Python тесты (testinfra):**

```bash
pytest --hosts=rke2_servers \
  --ansible-inventory=inventory/hosts.yml \
  --force-ansible \
  --connection=ansible \
  --sudo \
  testing/basic_server_tests.py
```

### CI/CD пайплайн

Проект использует GitHub Actions для CI/CD:

| Workflow | Триггер | Описание |
|----------|---------|----------|
| `lint.yml` | Push, PR | ansible-lint, yamllint, vale |
| `ubuntu.yml` | PR | Интеграционные тесты на Ubuntu |
| `rocky.yml` | PR | Интеграционные тесты на Rocky Linux |

### Локальное интеграционное тестирование

Требования:
- Настроенные AWS credentials
- Установленный Terraform

```bash
cd testing/
terraform init
terraform apply -var "GITHUB_RUN_ID=local-test" -var "os=rocky9"
```

---

## Участие в разработке

### Процесс Pull Request

1. **Форкните** репозиторий
2. **Создайте** ветку: `git checkout -b feature/my-feature`
3. **Внесите** изменения
4. **Протестируйте** локально
5. **Проверьте** линтером: `ansible-lint --profile=production ./roles`
6. **Закоммитьте** с понятным сообщением
7. **Запушьте** в свой форк
8. **Создайте** Pull Request

### Формат сообщений коммитов

```
тип: Краткое описание

Развёрнутое описание при необходимости.

Fixes #123
```

Типы: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`

### Чеклист код-ревью

- [ ] Код проходит `ansible-lint --profile=production`
- [ ] Код проходит `yamllint`
- [ ] Все задачи имеют описательные имена
- [ ] Нет захардкоженных путей (используются переменные)
- [ ] Добавлены соответствующие теги
- [ ] Документация обновлена при необходимости
