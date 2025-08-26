# Установка и настройка Vault в Kubernetes (одна реплика, без PVC)

## Шаг 1: Установка Vault через Helm

1. **Добавьте репозиторий HashiCorp**:
   ```bash
   helm repo add hashicorp https://helm.releases.hashicorp.com
   helm repo update
   ```

2. **Создайте namespace для Vault**:
   ```bash
   kubectl create namespace vault
   ```

3. **Создайте файл конфигурации `override-values.yml`**:
   ```yaml
   global:
     enabled: true
     tlsDisable: true
   server:
     dev:
       enabled: true
       devRootToken: "root"
     resources:
       requests:
         memory: 2Gi
         cpu: 1000m
       limits:
         memory: 4Gi
         cpu: 1000m
   ui:
     enabled: true
     serviceType: "LoadBalancer"
     externalPort: 8200
   csi:
     enabled: true
   injector:
     enabled: false
   ```
   - Режим `dev` использует `in-memory` хранилище, что исключает необходимость в PVC.
   - `devRootToken: "root"` задает корневой токен для упрощения доступа.
   - `tlsDisable: true` отключает TLS для простоты (не для продакшена).
   - `csi: enabled` включает Vault CSI Provider для доставки секретов.

4. **Установите Vault**:
   ```bash
   helm install vault hashicorp/vault --namespace vault -f override-values.yml
   ```

5. **Проверьте статус пода Vault**:
   ```bash
   kubectl get pods -n vault
   ```
   Убедитесь, что под `vault-0` в статусе `Running`.

## Шаг 2: Проверка доступа к Vault
1. **Получите внешний адрес Vault**:
   ```bash
   kubectl get svc -n vault
   ```
   Найдите сервис `vault-ui` с типом `LoadBalancer` и скопируйте его внешний IP или hostname.

2. **Аутентифицируйтесь в Vault**:
   В режиме `dev` Vault не требует инициализации и распечатки. Используйте корневой токен `root`:
   ```bash
   kubectl exec -it vault-0 -n vault -- vault login root
   ```
   Или откройте UI по адресу `http://<EXTERNAL-IP>:8200` и войдите с токеном `root`.

## Шаг 3: Настройка аутентификации Kubernetes

1. **Включите метод аутентификации Kubernetes**:
   ```bash
   kubectl exec -it vault-0 -n vault -- vault auth enable kubernetes
   ```

2. **Настройте доступ к API Kubernetes**:
   ```bash
   kubectl exec -it vault-0 -n vault -- vault write auth/kubernetes/config \
     kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
   ```

3. **Создайте политику для чтения секретов**:
   ```bash
   kubectl exec -it vault-0 -n vault -- vault policy write internal-app - <<EOF
   path "secret/data/db-pass" {
     capabilities = ["read"]
   }
   EOF
   ```

4. **Создайте роль для сервисного аккаунта**:
   ```bash
   kubectl exec -it vault-0 -n vault -- vault write auth/kubernetes/role/database \
     bound_service_account_names=webapp-sa \
     bound_service_account_namespaces=vault \
     policies=internal-app \
     ttl=20m
   ```

## Шаг 4: Создание секрета в Vault



1. **Создайте тестовый секрет**:
   ```bash
   kubectl exec -it vault-0 -n vault -- vault kv put secret/db-pass password="12345678"
   ```

2. **Проверьте секрет**:
   ```bash
   kubectl exec -it vault-0 -n vault -- vault kv get secret/db-pass
   ```
   Ожидаемый вывод:
   ```
   ====== Data ======
   Key         Value
   ---         -----
   password    12345678
   ```

## Шаг 5: Установка CSI-драйвера

1. **Добавьте репозиторий CSI-драйвера**:
   ```bash
   helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
   helm repo update
   ```

2. **Установите CSI-драйвер**:
   ```bash
   helm install csi secrets-store-csi-driver/secrets-store-csi-driver \
     --namespace vault \
     --set syncSecret.enabled=true
   ```

3. **Проверьте статус CSI-драйвера**:
   ```bash
   kubectl get pods -n vault -l "app=secrets-store-csi-driver"
   ```

## Шаг 6: Настройка SecretProviderClass

1. **Создайте файл `spc-vault-database.yaml`**:
   ```yaml
   apiVersion: secrets-store.csi.x-k8s.io/v1
   kind: SecretProviderClass
   metadata:
     name: vault-database
     namespace: vault
   spec:
     provider: vault
     parameters:
       vaultAddress: "http://vault.vault.svc:8200"
       roleName: "database"
       objects: |
         - objectName: "db-password"
           secretPath: "secret/data/db-pass"
           secretKey: "password"
   ```

2. **Примените SecretProviderClass**:
   ```bash
   kubectl apply -f spc-vault-database.yaml -n vault
   ```

## Шаг 7: Создание пода с доступом к секретам

1. **Создайте сервисный аккаунт**:
   ```bash
   kubectl create serviceaccount webapp-sa punten
   ```

2. **Создайте файл `webapp-pod.yaml`**:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: webapp
     namespace: vault
   spec:
     serviceAccountName: webapp-sa
     containers:
     - image: jweissig/app:0.0.1
       name: webapp
       volumeMounts:
       - name: secrets-store-inline
         mountPath: "/mnt/secrets-store"
         readOnly: true
     volumes:
     - name: secrets-store-inline
       csi:
         driver: secrets-store.csi.k8s.io
         readOnly: true
         volumeAttributes:
           secretProviderClass: "vault-database"
   ```

3. **Примените конфигурацию пода**:
   ```bash
   kubectl apply -f webapp-pod.yaml -n vault
   ```

4. **Проверьте доступ к секрету**:
   ```bash
   kubectl exec webapp -n vault -- cat /mnt/secrets-store/db-password
   ```
   Ожидаемый вывод: `12345678`

## Шаг 8: Проверка и отладка

- **Логи пода**:
   ```bash
   kubectl logs webapp -n vault
   ```
- **Статус Vault**:
   ```bash
   kubectl exec -it vault-0 -n vault -- vault status
   ```
- **Статус SecretProviderClass**:
   ```bash
   kubectl describe secretproviderclass vault-database -n vault
   ```

## Примечания
- **Режим `dev`**: Данные Vault теряются при перезапуске пода, так как используется `in-memory` хранилище.
- **Безопасность**: Для продакшена включите TLS (`tlsDisable: false`) и настройте сертификаты.
- **Удаление**:
   ```bash
   helm uninstall vault -n vault
   kubectl delete namespace vault
   ```
