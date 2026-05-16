# Local Port Map

이 문서는 로컬 개발 환경에서 포트 충돌을 피하기 위한 기준표다. 새 서비스를 추가할 때 먼저 이 파일을 확인하고, 포트를 할당하면 여기에 기록한다.

## Finance reserved ports

| Port | Service | Owner/config | Notes |
|---:|---|---|---|
| 3000 | Finance frontend Vite | `frontend-v2/vite.config.ts` | `strictPort: true`; finance frontend 고정 |
| 58000 | Finance Django | `backend/local.yml` | container `8000` → host `58000` |
| 58001 | Finance Adminer | `backend/local.yml` | container `8080` → host `58001` |
| 5555 | Finance Flower | `backend/local.yml` | Celery monitoring |
| 9000 | Finance backend docs | `backend/local.yml` | Sphinx docs |

## Current listener snapshot

Last checked: 2026-05-16.

| Port | Running service/process | Notes |
|---:|---|---|
| 3000 | `finance/frontend-v2` Vite | Finance frontend |
| 5173 | `media/file-organizer/frontend` Vite | Vite default; high collision risk |
| 5174 | `workspace2/home-dashboard/frontend` Vite | Vite fallback/current runtime port |
| 8000 | `media/file-organizer/backend` Python | Common FastAPI/Django dev default |
| 5432 | Docker-published Postgres | Common DB conflict point |
| 5555 | Docker-published Flower | Finance Flower |
| 58000 | Docker-published Django | Finance backend |
| 58001 | Docker-published Adminer | Finance Adminer |
| 9000 | Docker-published docs | Finance backend docs |
| 9090 | Docker-published Prometheus | Infra monitoring |
| 9100 | Docker-published node-exporter | Infra monitoring |

## Known non-finance project defaults

| Port | Service | Notes |
|---:|---|---|
| 3000 | `crm/contact/contact-crm` frontend, CRA sandboxes, `media/homepage` default | Conflicts with finance frontend unless reconfigured |
| 3010 | `media/homepage` local `.env` | Current local override; default/example is `3000` |
| 4000 | `crm/closecircle` API server | Fastify default from `PORT || 4000` |
| 5000 | `sandbox/app_test` | Flask-style app default |
| 5173 | `media/file-organizer`, `social/one-a-day`, `crm/closecircle` web, multiple Vite sandboxes | Vite default; avoid for long-running services |
| 5432 | `media/file-organizer`, `social/one-a-day`, `crm/contact/contact-crm`, many cookiecutter projects | Prefer not exposing DB ports on host |
| 5554 | `social/with_you` Flower | Avoids host `5555` |
| 5555 | Flower defaults in several Django/cookiecutter projects | Conflicts with finance Flower |
| 6379 | `crm/contact/contact-crm` Redis | Common Redis default |
| 7000 | `sandbox/playground/exp/experiment_manager` docs, `sandbox/entu/service-apigateway` docs container | Can conflict with custom dev UIs |
| 7100 | `sandbox/entu/service-apigateway` docs host mapping | Host override for docs |
| 8000 | `media/file-organizer`, `social/one-a-day`, `crm/contact/contact-crm`, `social/with_you`, Django/FastAPI sandboxes | Highest backend collision risk |
| 8025 | `social/with_you`, `sandbox/entu/service-apigateway` Mailpit/Mailhog | Mail UI |
| 8080 | `social/one-a-day` Adminer, Livy sandbox docs | Finance uses `58001` instead |
| 8090 | `social/one-a-day` Traefik UI | Proxy dashboard |
| 9000 | `social/with_you` docs, finance docs, finance-manager docs | Sphinx docs default |
| 9090 | `infra/monitoring` Prometheus | Monitoring reserved |
| 9100 | `infra/monitoring` node-exporter | Monitoring reserved |

## Collision hotspots

| Port | Why risky | Recommendation |
|---:|---|---|
| 3000 | React/Nuxt/default frontend port; finance frontend is fixed here | Do not assign to new services while finance is active |
| 5173 | Vite default used by many projects | Set explicit `server.port` and `strictPort: true` per project |
| 8000 | FastAPI/Django default used by many backends | Map project backends to a project-specific host range |
| 5432 | Postgres default; many compose files expose it | Keep DB internal unless host access is required |
| 5555 | Flower default | Use host remaps like `5554` or project-specific ranges |
| 9000 | Sphinx/docs default | Remap docs when multiple Django projects run together |

## Suggested workspace ranges

| Area | Suggested range | Notes |
|---|---:|---|
| Finance backend-side services | `58000-58099` | Current finance convention |
| Media apps | `3010-3099` | `media/homepage` already uses `3010` locally |
| CRM apps | `3100-3199` | Avoids finance `3000` |
| Social apps | `3200-3299` | Avoids Vite defaults |
| Monitoring/infra | `9090-9199` | Prometheus/node-exporter already here |
| Temporary sandboxes | framework default only | OK if run alone; remap if run with finance |

## Allocation rules

- Finance web frontend는 `3000`을 유지한다. Vite fallback으로 다른 포트에 뜨지 않도록 `strictPort: true`를 사용한다.
- Finance backend 계열 host port는 가능하면 `58xxx` 범위를 사용한다.
- 새 finance 서비스가 HTTP UI를 노출하면 `58002`부터 순차 할당한다.
- DB/Redis 등 내부 의존성은 가능하면 host port를 열지 않는다.
- 다른 프로젝트의 Vite 기본 포트(`5173`)와 겹치지 않게 한다.
- 포트를 추가하거나 바꾸면 이 파일, 관련 compose/config, `CLAUDE.md`를 같이 갱신한다.

## Quick checks

```bash
lsof -nP -iTCP -sTCP:LISTEN
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```
