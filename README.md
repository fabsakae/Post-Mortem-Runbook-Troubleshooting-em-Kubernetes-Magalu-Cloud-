#  Post-Mortem & Runbook: Troubleshooting em Kubernetes (Magalu Cloud)

##  Objetivo
Este repositório documenta a resolução de um incidente duplo ocorrido em um laboratório prático de Kubernetes na Magalu Cloud. O objetivo é servir como um guia de estudos e um *Runbook* de operações para diagnosticar falhas de infraestrutura (esgotamento de recursos) e falhas de aplicação (conexão com banco de dados).

---

##  Arquitetura e Contexto
*   **Provedor Cloud:** Magalu Cloud
*   **Orquestrador:** Kubernetes (K3s)
*   **Aplicação:** API Python (Uvicorn/FastAPI) conteinerizada.
*   **Banco de Dados:** PostgreSQL (DBaaS)
*   **Monitoramento:** Prometheus e Grafana

---

##  Incidente Parte 1: O Colapso da Infraestrutura

### O Sintoma
Os pods da aplicação entraram em ciclo infinito de reinicialização (`CrashLoopBackOff`) e os comandos do `kubectl` e `helm` começaram a apresentar lentidão extrema ou falhas de `timeout` (ex: `http2: client connection lost`).

### A Investigação 
Para entender se o problema era a aplicação ou o servidor, desci uma camada na abstração e investiguei o Sistema Operacional Linux.

1. **Verificação de RAM e CPU:**
   ```bash
   free -h && uptime
   ```

O free mostrou que a máquina tinha apenas 8MB de RAM disponíveis. O uptime revelou um Load Average na CPU de 30.00 (um estado de Thrashing, onde o Kernel gasta todo o processamento tentando gerenciar a falta de memória).

2. **Verificação da memória:**
  ```bash
  top -b -n 1 -o %MEM | head -n 15
  ```
Ordenou os processos que mais consumiam RAM. Descobri que o K3s, Prometheus e Grafana estavam asfixiando a máquina virtual (VM) de apenas 2GB de RAM.

3. **Confirmando a Intervenção do Kernel:**
   ```bash
   sudo dmesg -T | grep -i -E "killed process|oom"
   ```
Li os logs profundos do sistema operacional e confirmei que o OOM-Killer (Out Of Memory) estava assassinando processos essenciais para evitar o travamento total do servidor.

###A Resolução (Primeiros Socorros e Resize)
Estancar o consumo, depois aumentar o hardware.

1. **Parar o Kubernetes forçadamente para liberar RAM:**
   ```bash
   sudo systemctl stop k3s
   ```
2. **Realizei o Resize da VM na Magalu Cloud para 4vCPU | 8GB RAM | 40GB Disco | Ubuntu 24.04 LTS**

3. **Religar os serviços e validar a saúde do nó:**
  ```bash
  sudo systemctl start k3s
  kubectl get nodes
  ```
##  Incidente Parte 2: A Falha Simulada de Banco de Dados

### O Sintoma
Com a infraestrutura saudável e rodando perfeitamente, simular uma falha de conexão corrompendo a credencial do banco de dados.

1. **Para corromper a DATABASE_URL (injetando um host inválido no Secret):**
  ```bash
  kubectl create secret generic db-secret \
  --from-literal=url="postgresql://usuario:senha@host-invalido:5432/orders" \
  --dry-run=client -o yaml | kubectl apply -f -
  ```
2. **Para forçar os pods a reiniciarem e consumirem a nova senha corrompida:**
  ```bash
  kubectl rollout restart deployment/cloud-application
  ```
3. **Para acompanhar a queda da aplicação em tempo real:**
  ```bash
  kubectl get pods -w
  ```

 ## Parte 3:  Investigar com logs
 ler os logs para diagnosticar o incidente documentando a falha.
   ```bash
  kubectl logs cloud-application-588955d4f4-k2xgz
  ```
Saída do comando: sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) could not translate host name "host-invalido" to address: Name or service not known

## Parte 4: Restaurar e observar a recuperação

1. **Para injetar a URL correta e consertar o Secret:**
  ```bash
  kubectl create secret generic db-secret \
--from-literal=url="postgresql://usuario:senha@172.18.1.62:5432/orders" \
--dry-run=client -o yaml | kubectl apply -f -
  ```
2. **Para dar a ordem de reinicialização da aplicação e ela consumir a senha nova:**
  ```bash
  kubectl rollout restart deployment/cloud-application
  ```
