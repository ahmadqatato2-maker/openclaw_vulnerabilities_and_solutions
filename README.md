# OpenClaw — публичная документация / Public Documentation

Санитизированные версии рабочей документации по развёртыванию OpenClaw на VPS. Из файлов удалена приватная информация (домены, IP-адреса, токены, имена ботов, данные хостинга), сохранены архитектурные решения, требования безопасности и план развёртывания.

Sanitized versions of working documentation for deploying OpenClaw on a VPS. All private information has been removed (domains, IP addresses, tokens, bot names, hosting details); architectural decisions, security requirements, and the deployment plan are preserved.

---

## Файлы / Files

| Файл / File | Содержание / Contents |
|---|---|
| [openclaw_обзор_public.md](openclaw_обзор_public.md) | **RU** — Обзор платформы: возможности, архитектура, безопасность, системные требования, установка, мониторинг, модель оплаты |
| [openclaw_overview_public_en.md](openclaw_overview_public_en.md) | **EN** — Platform overview: features, architecture, security, system requirements, installation, monitoring, billing model |
| [развёртывание_решение_public.md](развёртывание_решение_public.md) | **RU** — План развёртывания: этапы (0–12), архитектурные решения, найденные проблемы и решения, отложенные задачи |
| [deployment_plan_public_en.md](deployment_plan_public_en.md) | **EN** — Deployment plan: stages (0–12), architectural decisions, issues found and solutions, deferred tasks |

---

## Как читать / How to read

**RU:**
- Начните с обзора (`openclaw_обзор_public.md`) — общее понимание платформы, её возможностей и рисков безопасности.
- Затем план развёртывания (`развёртывание_решение_public.md`) — пошаговая реализация с реальным опытом: какие проблемы возникли и как были решены.

**EN:**
- Start with the overview (`openclaw_overview_public_en.md`) — understand the platform, its capabilities, and security risks.
- Then the deployment plan (`deployment_plan_public_en.md`) — step-by-step implementation with real-world experience: what problems arose and how they were resolved.

---

## Что удалено / What was removed

- Доменные имена, публичные и Tailscale IP-адреса / Domain names, public and Tailscale IP addresses
- Названия Telegram-ботов, токены, ID пользователей / Telegram bot names, tokens, user IDs
- Конкретный хостинг-провайдер, тарифы, рублёвые цены / Specific hosting provider, plans, and prices in rubles
- Локальные пути с именами пользователей / Local paths with usernames
- Операционные детали, специфичные для конкретной инсталляции / Operational details specific to a particular installation
