# API Gateway

## Требования

- маршрутизация запросов к нужному сервису на основе конфигурации;
- проверка аутентификационной информации в запросах;
- терминация HTTPS.

## Сравнение решений

Для сравнения взял решения, которые чаще всего встречаются в self-hosted инсталляциях. 

| | Nginx (OSS) | Traefik (OSS) | Kong Gateway (OSS) | Apache APISIX | Envoy / Envoy Gateway | KrakenD (CE) |
|---|---|---|---|---|---|---|
| Основа | C | Go | Nginx + Lua (OpenResty) | Nginx + Lua (OpenResty) | C++ | Go |
| Маршрутизация по конфигурации | да, статический конфиг, нужен reload | да, динамически: файлы, Docker labels, K8s | да, декларативный YAML (DB-less) или Admin API | да, Admin API / YAML, конфиг хранится в etcd | да, xDS или Gateway API в K8s | да, декларативный JSON/YAML |
| Матчинг маршрутов | host, path | host, path, method, headers | host, path, method, headers | host, path, method, headers, args | host, path, method, headers | path, method |
| API key | вручную через map | нет | да, плагин key-auth | да, плагин key-auth | через фильтры / ext_authz | да |
| JWT | только в Nginx Plus | только в платных редакциях | да, плагин jwt | да, плагин jwt-auth | да, фильтр jwt_authn | да |
| OAuth2 / OIDC | нет (через внешний сервис) | нет (через ForwardAuth) | OAuth2 да, OIDC в Enterprise | да, openid-connect | да, через фильтр oauth2 / ext_authz | частично |
| Внешний сервис авторизации | auth_request | ForwardAuth | да | forward-auth | ext_authz | нет |
| Терминация HTTPS | да | да | да | да | да | да |
| Let's Encrypt из коробки | нет (certbot) | да | нет (cert-manager) | нет (cert-manager) | нет (cert-manager) | нет |
| mTLS до сервисов | да | да | да | да | да | да |
| Rate limiting | limit_req | middleware | плагин | плагин | фильтр | да |
| Метрики Prometheus | через exporter | да | плагин | плагин | да | да |
| Kubernetes | Ingress-контроллер (community ingress-nginx выведен из поддержки в марте 2026) | Ingress, Gateway API | Ingress Controller, Gateway API | Ingress Controller, Gateway API | Gateway API, основа Istio | обычный Deployment |
| Доп. компоненты | нет | нет | PostgreSQL (не нужен в DB-less) | etcd | control plane (Envoy Gateway / Istio) | нет |
| Сложность настройки | низкая | низкая | средняя | средняя | высокая | низкая |

## Что выбрал и почему

Все три требования из коробки закрывают Kong, APISIX, Envoy и KrakenD. Nginx OSS и Traefik OSS проверяют JWT только через отдельный сервис аутентификации (auth_request / ForwardAuth), то есть его придётся писать и поддерживать отдельно. Поэтому их отбросил.

Из оставшихся выбрал **Kong Gateway в режиме DB-less**:

1. Маршрутизация. Маршруты описываются в YAML (services → routes → upstreams) и хранятся в Git вместе с остальной инфраструктурой. Конфиг применяется через deck в CI/CD. В DB-less режиме база данных не нужна, поэтому дополнительных компонентов нет.
2. Аутентификация. Готовые плагины key-auth, jwt, basic-auth, oauth2, плюс acl для ограничения доступа к маршрутам по группам потребителей. Плагины включаются на нужных маршрутах, в коде сервисов ничего менять не надо. После проверки Kong передаёт в сервис заголовки с данными потребителя.
3. HTTPS. Сертификаты хранятся в конфиге Kong, поддерживается SNI. Внутри кластера трафик к сервисам идёт по HTTP или mTLS. Для автоматического выпуска сертификатов в Kubernetes можно использовать cert-manager.
4. Rate limiting, CORS, логи и метрики Prometheus тоже есть плагинами, при эксплуатации они понадобятся.
5. Самое распространённое open source решение такого класса, много документации и готовых примеров. Есть Ingress Controller с поддержкой Gateway API.

Почему не остальные:
1. APISIX по функциям почти такой же, но требует etcd. Это ещё один кластер, который нужно поддерживать.
2. Envoy самый производительный и гибкий, но сложный в настройке. Имеет смысл, если в будущем планируется service mesh (Istio). Тогда гейтвей и межсервисный трафик будут на одной технологии.
3. KrakenD простой и быстрый, но слабее по матчингу маршрутов и набору функций аутентификации.
