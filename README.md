# GitOps-репозиторий стенда yc-test

Целевое состояние кластера `yc-test`. Всё, что внутри Kubernetes после
bootstrap, описывается здесь и применяется Argo CD.

Заготовка для проверки цепочки из
[`k8s-install-playbook-v2.md`](../k8s-install-playbook-v2.md). Репозиторий
**публичный**, поэтому секретов в нём быть не должно ни при каких условиях.

## Структура

```
clusters/yc-test/
├── values/
│   ├── cilium.yaml        единственный источник values для Cilium
│   └── argocd.yaml        то же для Argo CD
└── bootstrap/
    ├── root-application.yaml   применяется Ansible на этапе 60
    ├── kustomization.yaml      включает root-application — он самоуправляем
    ├── projects.yaml           AppProject platform-root
    └── applications.yaml       takeover Cilium (wave -10) и Argo CD (wave -20)
```

## Единый источник values

Каталог `values/` читают двое:

* Ansible на этапах 50 и 60 — делает `helm template` с этим файлом;
* Argo CD в `applications.yaml` — через multi-source ссылается на тот же
  файл (`$values/clusters/yc-test/values/...`).

Расхождение между ними означало бы вечный `OutOfSync` после takeover.
Собственных шаблонов values у Ansible нет намеренно (раздел 4.3 playbook).

## Порядок первого запуска

Argo CD ставится с **выключенной** автоматикой: в `root-application.yaml`
нет блока `syncPolicy.automated`, а finalizer
`resources-finalizer.argocd.argoproj.io` не задан. Ansible это проверяет
и откажется применять манифест, если они появятся.

```bash
argocd app diff cluster-bootstrap     # разобрать все расхождения
argocd app sync cluster-bootstrap     # первый sync вручную
argocd app diff cilium                # реальных диффов в spec быть не должно
```

После проверки `cilium status --wait` автоматика включается отдельным PR:
сначала `automated` + `selfHeal`, `prune` — последним.

## Чем это отличается от production

| Здесь | В production |
| ----- | ------------ |
| публичный репозиторий по HTTPS, без аутентификации | приватный, deploy key из OpenBao (раздел 18.2) |
| `operator.replicas: 1` у Cilium | 2 |
| Argo CD без HA, `redis-ha` выключен | HA-установка (раздел 3.1) |
| `server.insecure: true` | ingress с TLS |
| SSO выключен | SSO подключён, встроенный admin ограничен (раздел 29.4) |
| root Application в проекте `default` | переводится в `platform-root` с запретом delete |
