# 🛠️ Debug Images — OpenShift / Kubernetes

Duas imagens prontas para debug de cluster, não-root, compatíveis com
as Security Context Constraints (SCC) padrão do OpenShift.

---

## Estrutura

```
.
├── docker-compose.yml
├── debug-full/
│   ├── Dockerfile        ← imagem completa (oc + kubectl + toolbox)
│   └── motd.sh           ← banner exibido no login
└── debug-light/
    ├── Dockerfile        ← imagem leve (toolbox + nginx)
    ├── nginx.conf        ← nginx não-root na porta 8080
    └── html/
        └── index.html    ← página de boas-vindas
```

---

## Build

### Ambas de uma vez
```bash
docker compose build
```

### Individualmente
```bash
# Completa
docker build -t debug-full:1.0.0 ./debug-full

# Leve (com args opcionais de versão)
docker build \
  --build-arg OC_VERSION=4.15.0 \
  --build-arg KUBECTL_VERSION=v1.30.0 \
  -t debug-full:1.0.0 ./debug-full

docker build -t debug-light:1.0.0 ./debug-light
```

---

## Uso local (Docker)

```bash
# Testar imagem completa (shell interativo)
docker run -it --rm --cap-add=NET_RAW debug-full:1.0.0

# Testar imagem leve (web server)
docker run -d -p 8080:8080 --name debug-light debug-light:1.0.0
open http://localhost:8080
```

---

## Uso no OpenShift

### Deploy como pod efêmero de debug
```bash
# Imagem completa — debug interativo
oc run debug-pod \
  --image=sua-registry/debug-full:1.0.0 \
  --restart=Never \
  --rm -it \
  -- bash

# Imagem leve — verificar conectividade com web server
oc run debug-web \
  --image=sua-registry/debug-light:1.0.0 \
  --port=8080 \
  --restart=Never
```

### Manifest de exemplo (debug-full)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
  namespace: seu-namespace
spec:
  containers:
  - name: debug
    image: sua-registry/debug-full:1.0.0
    stdin: true
    tty: true
    securityContext:
      runAsNonRoot: true
      runAsUser: 1001
      capabilities:
        add: ["NET_RAW", "NET_ADMIN"]
  restartPolicy: Never
```

### SCC necessária
As imagens requerem `cap_net_raw` e `cap_net_admin` para tcpdump e nmap.
Use a SCC `privileged` em ambiente de teste ou crie uma SCC customizada:

```bash
oc adm policy add-scc-to-user privileged -z default -n seu-namespace
```

> ⚠️ Em produção, prefira uma SCC customizada que libere apenas NET_RAW/NET_ADMIN.

---

## Ferramentas disponíveis

| Ferramenta    | debug-full | debug-light |
|---------------|:----------:|:-----------:|
| nmap          | ✅         | ✅          |
| tcpdump       | ✅         | ✅          |
| iperf3        | ✅         | ✅          |
| iptraf-ng     | ✅         | ✅          |
| mtr           | ✅         | ✅          |
| traceroute    | ✅         | ✅          |
| curl          | ✅         | ✅          |
| wget          | ✅         | ✅          |
| git           | ✅         | ✅          |
| **oc client** | ✅         | ❌          |
| **kubectl**   | ✅         | ❌          |
| **nginx**     | ❌         | ✅ (8080)   |

---

## Notas de segurança

- UID fixo `1001`, GID `0` (padrão OpenShift arbitrary UID)
- Sem `sudo`, sem `su`
- `nginx` escuta na porta `8080` (não privilegiada)
- Todos os paths temporários do nginx apontam para `/tmp`
- `setcap` aplicado em `tcpdump` e `nmap` para dispensar root
