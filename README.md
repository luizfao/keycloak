# RHBK (Red Hat Build of Keycloak) via OpenShift GitOps (Argo CD)

Este repositório contém manifests **GitOps** para instalar e configurar, em um cluster OpenShift, um ambiente de laboratório com:

- **PostgreSQL** (persistente) para o RHBK
- **OpenLDAP** (persistente) para federação de usuários/grupos
- **RHBK Operator v24** (OLM) com `installPlanApproval: Manual`
- **Instância do RHBK** + **RealmImport** + **LDAP mappers**

Documentação do produto (v24): `https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/24.0/`

## Como usar (rápido)

### Pré-requisitos

- Você estar logado no cluster (`oc whoami`)
- OpenShift GitOps (Argo CD) instalado (namespace `openshift-gitops`)
- Este repositório acessível pelo cluster (GitHub público ou credenciais configuradas no ArgoCD)

### 1) Bootstrap do ArgoCD (uma vez)

Aplicar o `Application` que aponta para `gitops/`:

```bash
oc apply -f keycloak/bootstrap/application.yaml
```

Isso cria o `Application` `rhbk` no ArgoCD e ele passa a sincronizar `keycloak/gitops/`.

### 2) Onde tudo é instalado

- **Namespace GitOps/lab**: `rhbk-gitops`
- O Operator do RHBK é instalado **no próprio `rhbk-gitops`** (namespaced) via `OperatorGroup`.

### 3) URLs

Substitua o domínio se necessário (este lab usou `apps.cluster-zrdcz.dynamic.redhatworkshops.io`).

- **Admin Console (login como admin do master)**:
  - `https://rhbk-rhbk-gitops.apps.cluster-zrdcz.dynamic.redhatworkshops.io/admin/master/console/`
- **Account Console (login como usuário do realm `rhbk`, ex.: LDAP)**:
  - `https://rhbk-rhbk-gitops.apps.cluster-zrdcz.dynamic.redhatworkshops.io/realms/rhbk/account/`

### 4) Usuários e senhas (lab)

#### RHBK admin (gerado pelo operator)

- **Usuário**: `admin`
- **Senha**: obtenha do `Secret` `rhbk-initial-admin`:

```bash
oc -n rhbk-gitops get secret rhbk-initial-admin -o jsonpath='{.data.username}' | base64 -d; echo
oc -n rhbk-gitops get secret rhbk-initial-admin -o jsonpath='{.data.password}' | base64 -d; echo
```

#### LDAP

- **Bind DN usado pelo RHBK**: `cn=keycloak,dc=example,dc=org`
- **Bind password**: obtenha do `Secret` `openldap`:

```bash
oc -n rhbk-gitops get secret openldap -o jsonpath='{.data.bindUsername}' | base64 -d; echo
oc -n rhbk-gitops get secret openldap -o jsonpath='{.data.bindPassword}' | base64 -d; echo
```

- **Usuário de teste**: `ldaptest`
- **Senha**: definida no LDIF do seed; para ver no cluster:

```bash
oc -n rhbk-gitops get configmap openldap-seed-ldif -o jsonpath='{.data.seed\.ldif}' | sed -n 's/^userPassword: //p'
```

- **Grupo de teste**: `rhbk-testers` (grupo `groupOfNames` com `member` apontando para `ldaptest`)

#### PostgreSQL

- **Database**: `keycloak`
- **Usuário**: `keycloak`
- **Senha**: obtenha do `Secret` `postgresql`:

```bash
oc -n rhbk-gitops get secret postgresql -o jsonpath='{.data.username}' | base64 -d; echo
oc -n rhbk-gitops get secret postgresql -o jsonpath='{.data.password}' | base64 -d; echo
```

## O que foi criado (arquivos/manifests)

### GitOps “raiz”

- `keycloak/bootstrap/application.yaml`
  - Cria o `Application` do ArgoCD que sincroniza este repo (caminho `gitops/`).
- `keycloak/gitops/application.yaml`
  - Cria o namespace `rhbk-gitops`.

### Dependências

- `keycloak/gitops/postgresql/postgresql.yaml`
  - `Secret` + `Service` + `StatefulSet` com PVC.
- `keycloak/gitops/openldap/openldap.yaml`
  - `Secret` + `Service` + `StatefulSet` com PVC.
  - Inclui `ServiceAccount` + `RoleBinding` para SCC `anyuid` (necessário para a imagem `osixia/openldap` neste lab).
- `keycloak/gitops/openldap/seed/seed.yaml`
  - `ConfigMap` com LDIF + `Job` para seed idempotente (usuário + grupo de teste).

### RHBK

- `keycloak/gitops/rhbk/operator/operatorgroup.yaml`
  - `OperatorGroup` **namespaced** (o operador v24 não suporta `AllNamespaces`).
- `keycloak/gitops/rhbk/operator/subscription.yaml`
  - `Subscription` do `rhbk-operator` no canal **`stable-v24.0`** com `installPlanApproval: Manual`.
- `keycloak/gitops/rhbk/instance/keycloak.yaml`
  - CR `Keycloak` (instância do RHBK) apontando para PostgreSQL.
  - Ajuste de proxy: `proxy.headers: xforwarded` para funcionar atrás do Route edge-terminated.
- `keycloak/gitops/rhbk/realm-import/realm.yaml`
  - CR `KeycloakRealmImport` criando o realm `rhbk` e o provider LDAP `openldap`.
- `keycloak/gitops/rhbk/ldap-mappers/job.yaml`
  - `Job` (hook de sync do ArgoCD) que cria os LDAP mappers necessários:
    - `username` (cn → username)
    - `first name`, `last name`, `email`
    - `groups` (group mapper para `ou=Groups`)

## Principais problemas encontrados e como resolvemos

- **ArgoCD “apagou” o Application durante force sync**:
  - Recriamos via `keycloak/bootstrap/application.yaml`.
- **ImagePull errors**:
  - PostgreSQL em `quay.io/sclorg/...` retornava `unauthorized` → trocamos para `registry.redhat.io/rhel9/postgresql-15:latest`.
  - `bitnami/openldap:latest` não existia → migramos para `osixia/openldap:1.5.0`.
- **Operator RHBK falhando (CSV Failed)**:
  - `AllNamespaces` não é suportado no v24 → instalamos como **namespaced** (`OperatorGroup` + `Subscription` em `rhbk-gitops`).
- **Admin UI “Loading…” / 403 em `/admin/serverinfo`**:
  - Corrigimos o proxy (`proxy.headers: xforwarded`) e usamos a URL correta do console (`/admin/master/console/`).
- **Login LDAP falhando com “null username”**:
  - O provider LDAP estava criado, mas faltavam LDAP mappers padrão → criamos via Job GitOps.
  - O group mapper exigia `mode` → adicionamos `mode: READ_ONLY` e `group.object.classes: groupOfNames`.

## Pontos de melhoria (somente o que realmente importa)

- **Segredos no Git** (valores de laboratório e placeholders): em uso real, migrar para **SealedSecrets/ExternalSecrets/Vault**.
- **Fixar imagens por digest** (evitar drift): hoje há uso de tags como `latest` em alguns lugares (ex.: PostgreSQL/UBI initContainer/OpenLDAP).
- **TLS fim-a-fim**: este lab usa Route com edge termination + HTTP interno. Para produção, preferir TLS adequado (por exemplo reencrypt/passthrough com secret TLS conforme a arquitetura recomendada do produto).
- **GitOps com `prune/selfHeal`**: o `Application` está com `prune: false` e `selfHeal: false`; para operação contínua e limpeza automática, avaliar habilitar conforme política da plataforma.
