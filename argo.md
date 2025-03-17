Ошибка “permission denied” в ArgoCD при попытке использования history and rollback связана с недостатком прав у пользователя или сервисного аккаунта. Разберем необходимые разрешения и способы диагностики.

⸻

1. Какие права нужны для history and rollback?

ArgoCD использует RBAC (Role-Based Access Control) для управления правами. Для работы с историей и откатом (rollback) необходимы следующие права:
	•	Просмотр истории развертываний
	•	applications, get
	•	applications, get/log
	•	applications, get/resource
	•	Выполнение отката (rollback)
	•	applications, update

Пример корректных разрешений в RBAC:

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argocd-app-manager
  namespace: argocd
rules:
- apiGroups: ["argoproj.io"]
  resources: ["applications"]
  verbs: ["get", "list", "update"]



⸻

2. Проверка текущих прав пользователя

Проверить права можно командой в ArgoCD:

argocd account can-i get applications
argocd account can-i update applications

Если ответ No, значит у пользователя недостаточно прав.

Проверить через Kubernetes RBAC

Если используется Kubernetes RBAC, проверить можно так:

kubectl auth can-i update applications --as=<username> -n argocd



⸻

3. Возможные причины ошибки и решения

Причина	Решение
У пользователя нет прав update на ресурс applications	Назначить права через ArgoCD RBAC (policy.csv) или Kubernetes RBAC
Ошибка в конфигурации ArgoCD RBAC	Проверить argocd-rbac-cm в ConfigMap
Проблема с ServiceAccount	Проверить, с каким SA работает ArgoCD и дать нужные права
ArgoCD использует внешний OIDC/SAML (SSO)	Проверить групповые разрешения в argocd-rbac-cm



⸻

4. Дополнительные проверки

Проверить настройки ArgoCD RBAC (если используется ConfigMap)

kubectl get configmap argocd-rbac-cm -n argocd -o yaml

Если в policy.csv нет строки:

p, role:developer, applications, update, */*, allow

То добавить ее в ConfigMap.

Проверить права сервисного аккаунта

Если ArgoCD работает через Kubernetes ServiceAccount, то проверить:

kubectl get rolebinding -n argocd
kubectl describe rolebinding <rolebinding-name> -n argocd

Если у SA нет update applications, нужно исправить RoleBinding.

⸻

5. Пример исправления прав

Если нужен доступ только к rollback:

p, role:developer, applications, get, */*, allow
p, role:developer, applications, update, */*, allow

Добавить в argocd-rbac-cm и перезапустить ArgoCD:

kubectl rollout restart deployment argocd-server -n argocd

Если используется Kubernetes RBAC:

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argocd-rollback
  namespace: argocd
rules:
- apiGroups: ["argoproj.io"]
  resources: ["applications"]
  verbs: ["get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argocd-rollback-binding
  namespace: argocd
subjects:
- kind: User
  name: <username>
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: argocd-rollback
  apiGroup: rbac.authorization.k8s.io



⸻

6. Вывод

Ошибка “permission denied” возникает из-за нехватки прав update на applications. Для решения проблемы:
	1.	Проверить текущие права (argocd account can-i или kubectl auth can-i).
	2.	Настроить ArgoCD RBAC (ConfigMap argocd-rbac-cm).
	3.	Настроить Kubernetes RBAC (Role + RoleBinding).
	4.	Перезапустить ArgoCD, если были изменены права.

Если проблема осталась, уточните детали: какой метод аутентификации используется (OIDC, SSO, Kubernetes RBAC) и какие права уже назначены.