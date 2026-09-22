# Documentação de Arquitetura e Funcionamento: Pod `dns-service`

Este documento detalha o funcionamento, arquitetura e fluxo operacional do pod **`dns-service-7b85c5d67f-vgfqq`**, seus manifestos GitOps e suas imagens locais.

---

## 1. Visão Geral

O pod `dns-service` opera no namespace `default` do cluster Kubernetes (K3s). O seu objetivo central é fornecer **resolução e automação de DNS dinâmica para o ambiente local (`*.lab`)**.

Ele atua como um resolvedor DNS autoritativo que se **auto-configura em tempo real**: sempre que uma nova aplicação ou serviço cria um recurso `Ingress` com terminação `.lab`, o pod detecta o evento e cadastra automaticamente a rota DNS correspondente para o IP de entrada da infraestrutura (`10.9.0.1`).

```
                              Kubernetes Cluster
                      ┌─────────────────────────────────┐
                      │    Ingress (ex: demo.lab)       │
                      └────────────────┬────────────────┘
                                       │ Watch Event (Informer)
                                       ▼
┌────────────────────────────────────────────────────────────────────────────────┐
│ Pod: dns-service                                                               │
│                                                                                │
│  ┌─────────────────────────────┐           ┌────────────────────────────────┐  │
│  │ Container: ingress-watcher  │ ────────► │ Container: dnsd                │  │
│  │ (Sidecar - Go client-go)    │ HTTP REST │ (DNS Server UDP :53 / API :8081)│  │
│  │ Watcher de Ingresses        │ 127.0.0.1 │ Autoridade DNS + Cache         │  │
│  └─────────────────────────────┘           └───────────────┬────────────────┘  │
└────────────────────────────────────────────────────────────┼───────────────────┘
                                                             │ hostPath Volume
                                                             ▼
                                                    /home/opc/dns-data/records.json
```

---

## 2. Padrão Arquitetural: Sidecar Pattern

O Pod adota o padrão **Sidecar** composto por dois contêineres que compartilham a mesma camada de rede (`localhost` / `127.0.0.1`) e ciclo de vida:

### Container 1: `dnsd` (Servidor DNS Principal)
* **Imagem:** `dnsd:2.0` (`imagePullPolicy: Never`)
* **Código-fonte:** `/opt/docker-images/dns-resolver/`
* **Tecnologia:** Go (compilação estática `linux/arm64` com `alpine:3.21`)
* **Portas Expostas:**
  * `53/UDP` (`hostPort: 53`): Atende requisições DNS da máquina e da rede local.
  * `8081/TCP`: Servidor HTTP REST e painel web integrado (via `go:embed`).
* **Volume Persistente:** Monta o hostPath `/home/opc/dns-data` em `/data`, lendo e persistindo registros no arquivo `/data/records.json`.
* **Comportamento:**
  1. Carrega os registros locais do arquivo JSON.
  2. Responde imediatamente para consultas que batem com as entradas locais (`A`, `AAAA`, etc.).
  3. Consultas para domínios externos são encaminhadas ao upstream forwarder (`1.1.1.1:53`) com suporte a cache em memória.
  4. Disponibiliza a API HTTP REST:
     * `POST /records`: Cadastra um registro DNS na zona.
     * `DELETE /records?name=...&type=A`: Remove um registro DNS.
     * `GET /records`: Lista os registros ativos.

---

### Container 2: `ingress-watcher` (Sidecar Controller)
* **Imagem:** `docker.io/library/ingress-watcher:1.4` (`imagePullPolicy: Never`)
* **Código-fonte:** `/opt/docker-images/dns-watcher/`
* **Tecnologia:** Go utilizando `k8s.io/client-go` e `SharedInformer`
* **Variáveis de Ambiente:**
  * `DNS_API_URL`: `http://127.0.0.1:8081` (comunicação direta via loopback com o contêiner vizinho).
  * `DNS_TARGET_IP`: `10.9.0.1` (IP para onde os domínios `.lab` devem apontar).
* **Comportamento:**
  1. Conecta-se à API do Kubernetes via `InClusterConfig`.
  2. Inicia um `SharedInformer` com resync a cada 5 minutos para monitorar a criação, alteração e deleção de objetos `Ingress` em todos os namespaces.
  3. Ao detectar um `Ingress` cujo `host` termina em `.lab`:
     * **Criação/Atualização:** Envia um `POST /records` para o `dnsd` cadastrando `host -> 10.9.0.1`.
     * **Remoção:** Envia um `DELETE /records?name=<host>&type=A` limpando a entrada.

---

## 3. Origem e Construção das Imagens (`/opt/docker-images`)

As duas imagens usadas pelo pod são customizadas e compiladas localmente no nó da máquina:

| Diretório | Imagem Gerada | Descrição |
| :--- | :--- | :--- |
| `/opt/docker-images/dns-resolver` | `dnsd:2.0` | Binário `dnsd` em Go que gerencia a zona DNS, expõe a porta 53 e a API HTTP 8081. |
| `/opt/docker-images/dns-watcher` | `ingress-watcher:1.4` | Binário `watcher` em Go que roda o loop de sincronização com o Kubernetes. |

* **Construção:** Ambas utilizam *Multi-stage Build* com Go Alpine e são compiladas com `GOOS=linux GOARCH=arm64` com `-ldflags="-s -w"` para gerar binários mínimos.
* **Armazenamento no K3s:** As imagens foram importadas diretamente para o runtime de contêineres (`containerd` / `crictl`). O manifesto Kubernetes define `imagePullPolicy: Never`, garantindo que o K3s não tente baixá-las de um registry externo, utilizando a versão já presente na máquina.

---

## 4. Estrutura dos Manifestos GitOps

O ciclo de vida do serviço é gerenciado pelo **ArgoCD** no repositório `platform-gitops-lab`:

### A. Bootstrap do ArgoCD
* **Arquivo:** `/home/opc/platform-gitops-lab/bootstrap/dns-app.yaml`
* **Recurso:** `Application` (`argoproj.io/v1alpha1`)
* **Propósito:** Informa ao ArgoCD para sincronizar automaticamente (`automated: prune: true, selfHeal: true`) a pasta `apps/dns-service` do repositório Git com o namespace `default` do cluster.

### B. Manifestos da Aplicação
Localizados em `/home/opc/platform-gitops-lab/apps/dns-service/`:

1. **`deployment.yaml`:**
   * Nome: `dns-service`
   * Estratégia: `type: Recreate` (essencial para evitar conflito de bind da `hostPort: 53` e acesso concorrente ao arquivo de zona em disco durante atualizações).
   * Declara os dois containers (`dnsd` e `ingress-watcher`), as variáveis de ambiente, portas e a montagem do volume `/home/opc/dns-data`.

2. **`rbac.yaml`:**
   * Cria o `ServiceAccount` `dns-service-sa`.
   * Cria o `ClusterRole` `dns-ingress-watcher` concedendo permissões de `["get", "list", "watch"]` no recurso `ingresses` da API `networking.k8s.io`.
   * Cria o `ClusterRoleBinding` `dns-ingress-watcher-binding`, permitindo que o sidecar `ingress-watcher` audite os Ingresses do cluster.

3. **`configmap.yaml`:**
   * ConfigMap `dns-watcher-config` que externaliza as variáveis operacionais (`DNS_TARGET_IP` e `DNS_API_URL`) do sidecar.

4. **`pvc.yaml`:**
   * `PersistentVolumeClaim` `dns-data-pvc` utilizando o storage provider local (`local-path`) para persistência desacoplada do host físico.

5. **`service.yaml`:**
   * Service `NodePort` que expõe a porta HTTP `8081` do `dns-service` na porta externa do nó `30081`.

6. **`ingress.yaml`:**
   * Expõe o painel web/API do DNS para o host `dns.lab` através do Traefik.
   * *Curiosidade:* O próprio `ingress-watcher` detecta este Ingress no boot e cria o registro `dns.lab -> 10.9.0.1` automaticamente no `dnsd`.

---

## 5. Ciclo de Vida e Fluxo de Execução

1. O ArgoCD aplica os manifestos de `apps/dns-service`.
2. O K3s agenda o pod no nó `platform-node-01`.
3. O contêiner `dnsd` inicia, abre a porta `53/UDP` diretamente no host, sobe o servidor HTTP na porta `8081` e carrega os registros existentes do volume `/data/records.json`.
4. O contêiner `ingress-watcher` inicia em paralelo, obtém o token de service account in-cluster e lista todos os Ingresses existentes.
5. Para qualquer Ingress com host terminado em `.lab` (como `dns.lab` e outros do laboratório), o watcher faz um `POST` no `localhost:8081`.
6. O `dnsd` atualiza sua memória e salva o novo registro no volume persistente (`records.json`).
7. Qualquer cliente da rede local que consulte a porta 53 do nó por domínios `.lab` obtém a resposta imediata apontando para o Traefik (`10.9.0.1`).

---

## 6. Registro de Melhorias e Hardening

Para detalhes sobre as decisões arquiteturais de hardening (probes de saúde, requests/limits, mitigação de OOM, desacoplamento de storage e justificativas para entrevistas técnicas), consulte o documento:
* [**`MELHORIAS.md`**](file:///home/opc/platform-gitops-lab/apps/dns-service/MELHORIAS.md)

