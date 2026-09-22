# Histórico de Melhorias e Hardening: Pod `dns-service`

Este documento registra as alterações arquiteturais e operacionais implementadas nos manifestos do `dns-service` (`deployment.yaml`, `configmap.yaml`, `pvc.yaml`), detalhando a motivação técnica, os riscos mitigados e os trade-offs de cada decisão.

---

## 1. Resumo das Alterações Realizadas

| Componente | Arquivo | Alteração Realizada | Categoria |
| :--- | :--- | :--- | :--- |
| **Healthchecks** | `deployment.yaml` | Adição de `startupProbe`, `readinessProbe` e `livenessProbe` no container `dnsd`. | Confiabilidade / Resiliência |
| **QoS e Recursos** | `deployment.yaml` | Definição explícita de `requests` e `limits` de CPU e Memória para ambos os containers. | Estabilidade de Nós / SRE |
| **Configuração** | `configmap.yaml` + `deployment.yaml` | Externalização de `DNS_TARGET_IP` e `DNS_API_URL` via `ConfigMap` desacoplado (`envFrom`). | 12-Factor App / Boas Práticas |
| **Persistência** | `pvc.yaml` + `deployment.yaml` | Migração do volume `hostPath` (`/home/opc/dns-data`) para `PersistentVolumeClaim` (`local-path`). | Portabilidade / Cloud-Native |
| **Ciclo de Vida** | `deployment.yaml` | Adição de `terminationGracePeriodSeconds: 15`. | Integridade de Dados |
| **Segurança** | `deployment.yaml` | Adição de `allowPrivilegeEscalation: false` no `ingress-watcher`. | Hardening de Pod |

---

## 2. Detalhamento e Justificativa Técnica

### A. Healthchecks (`startupProbe`, `readinessProbe` e `livenessProbe`)

* **O Problema:** Anteriormente, o pod não possuía nenhuma sonda configurada. Se o processo `dnsd` entrasse em deadlock ou falhasse ao subir a API HTTP na porta `8081`, o Kubernetes continuava marcando o pod como saudável (`Ready`), roteando requisições e mantendo o contêiner em execução sem tentar recuperá-lo.
* **A Alteração:**
  * **`startupProbe`:** Aguarda a inicialização do `dnsd` na porta `8081` (rota `/records`) com até 20 segundos de tolerância (`periodSeconds: 2`, `failureThreshold: 10`). Evita que o `livenessProbe` mate o container prematuramente durante a carga de inicialização.
  * **`readinessProbe`:** Avalia a cada 5 segundos se o `dnsd` está respondendo requisições HTTP antes de colocá-lo como endpoint elegível no Service.
  * **`livenessProbe`:** Testa a integridade contínua a cada 10 segundos. Se o processo travar e falhar 3 vezes consecutivas, o kubelet reinicia o container automaticamente.

---

### B. Gestão de Recursos (`requests` e `limits`) e QoS Class

* **O Problema:** Manifestos sem declaração de recursos colocam o Pod na classe de QoS **`BestEffort`**. Em situações de pico de carga ou contenção de memória na máquina física, pods `BestEffort` são os **primeiros a serem despejados (evicted)** ou abatidos pelo Kernel via OOM Killer, o que para um serviço de DNS local paralisaria a infraestrutura de rede do laboratório.
* **A Alteração:**
  * **`dnsd`:**
    * Requests: `cpu: 20m`, `memory: 32Mi`
    * Limits: `cpu: 100m`, `memory: 64Mi`
  * **`ingress-watcher`:**
    * Requests: `cpu: 10m`, `memory: 24Mi`
    * Limits: `cpu: 50m`, `memory: 48Mi`
* **Benefício:** O pod passa a ter QoS **`Burstable`**, garantindo reserva de capacidade no escalonador do Kubernetes e prevenindo que vazamentos de memória em um dos contêineres derrubem o nó inteiro.

---

### C. Externalização de Configuração com `ConfigMap`

* **O Problema:** Os parâmetros operacionais do sidecar (`DNS_TARGET_IP: "10.9.0.1"` e `DNS_API_URL: "http://127.0.0.1:8081"`) estavam embutidos diretamente no YAML do `Deployment`. Qualquer alteração de IP de entrada ou porta exigia modificar a especificação do workload.
* **A Alteração:**
  * Criação do manifesto `configmap.yaml` com o recurso `dns-watcher-config`.
  * Utilização de `envFrom` no container `ingress-watcher`.
* **Benefício:** Alinhamento com o princípio de configuração independente (12-Factor App) e facilidade de manutenção sem tocar no manifesto principal de deployment.

---

### D. Migração de `hostPath` para `PersistentVolumeClaim` (PVC)

* **O Problema:** O uso de `hostPath` apontando diretamente para `/home/opc/dns-data` criava acoplamento físico com a estrutura de diretórios do usuário `opc` no nó, impedia portabilidade e exigia permissões elevadas de filesystem no nó.
* **A Alteração:**
  * Criação do manifesto `pvc.yaml` (`dns-data-pvc`) utilizando o StorageClass padrão do K3s (`local-path`).
  * Atualização da referência de volume no `deployment.yaml` para consumir o `persistentVolumeClaim`.
* **Benefício:** O provisionamento e gerenciamento do diretório passam a ser responsabilidade nativa do Kubernetes, permitindo migração futura para outros StorageClasses compartilhados (ex: Longhorn ou NFS) sem alterar o Deployment.

---

### E. Desligamento Gracioso (`terminationGracePeriodSeconds`)

* **O Problema:** Ao realizar um rollout ou atualização com `strategy: Recreate`, o pod precisa garantir que qualquer escrita pendente no arquivo JSON de zona (`/data/records.json`) seja concluída em disco antes de o processo ser encerrado abruptamente por um `SIGKILL`.
* **A Alteração:**
  * Configuração explícita de `terminationGracePeriodSeconds: 15`.
* **Benefício:** Concede tempo hábil para que os sockets UDP e TCP sejam fechados de forma limpa e que o cache da zona faça flush para o disco com integridade.

---

### F. Hardening de Segurança no Sidecar (`allowPrivilegeEscalation: false`)

* **O Problema:** Processos de pods rodando sem restrições de escalonamento de privilégio podem permitir que atacantes que explorem vulnerabilidades no binário obtenham mais permissões do que o processo pai possui.
* **A Alteração:**
  * Adicionado `securityContext.allowPrivilegeEscalation: false` no contêiner `ingress-watcher`.
* **Benefício:** Reforço do princípio de menor privilégio (Least Privilege) no ciclo de execução do container sidecar.
