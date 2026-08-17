# CISO Assistant, deploy e modelagem de SGSI

Plataforma de GRC open source implantada em laboratório, com SGSI modelado sobre a ISO/IEC 27001:2022.

Cada requisito da norma ligado a uma prática implementada e a uma evidência verificável.

> Laboratório pessoal de P&D. Endereçamento genérico.

---

## Stack

| Componente | Versão |
|---|---|
| Ubuntu Server | 24.04 LTS |
| Docker CE | 29.7.2 |
| Docker Compose | v5.5.0 |
| CISO Assistant | v3.21.1 community |

VM com 6 vCPU, 12 GB de RAM e 120 GB de disco.

| Container | Função | Porta |
|---|---|---|
| `backend` | API Django, SQLite | 8000 |
| `frontend` | SvelteKit com SSR | 3000 |
| `caddy` | Proxy reverso, TLS | 8443 |
| `huey` | Fila assíncrona | — |
| `qdrant` | Base vetorial | 6333 |

---

## Escolha da ferramenta

| Categoria | Exemplos | Resolve | Ciclo |
|---|---|---|---|
| DFIR | DFIR-IRIS, TheHive | Caso, timeline, cadeia de custódia | Diário |
| ITSM | GLPI, Zammad, iTop | Fila, SLA, atendimento | Diário |
| GRC | CISO Assistant, Eramba, SimpleRisk | Controle, auditoria, evidência | Trimestral / anual |

Dentro de GRC.

| Ferramenta | Ponto forte | Consideração |
|---|---|---|
| CISO Assistant | Deploy simples, mapeamento multi framework | Comunidade menor |
| Eramba | Mais completo e maduro | Curva longa, edição comunitária restrita |
| SimpleRisk | Forte em gestão de risco | Não cobre GRC inteiro |

TheHive deixou de ser open source puro na v5. A partir da 5.3 a edição comunitária exige licença registrada, sem a qual a interface fica em modo somente leitura.

---

## Deploy

### Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
```

Rotação de log, aplicada antes de criar containers.

```bash
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{"log-driver":"json-file","log-opts":{"max-size":"10m","max-file":"3"}}
EOF
sudo systemctl restart docker
```

### Aplicação

```bash
cd /opt
sudo git clone https://github.com/intuitem/ciso-assistant-community.git
sudo chown -R $USER:$USER /opt/ciso-assistant-community
cd ciso-assistant-community

git tag --sort=-v:refname | head -5
git checkout v3.21.1
```

> `--sort=-v:refname` ordena por versão. Sem o parâmetro, `v2.4.9` aparece após `v2.4.29`.

### Endereço e exposição

```bash
cp docker-compose.yml docker-compose.yml.bak
IP=192.168.x.x

sed -i "s|https://localhost:8443|https://${IP}:8443|g" docker-compose.yml
sed -i "s|ALLOWED_HOSTS=backend,localhost|ALLOWED_HOSTS=backend,localhost,${IP}|g" docker-compose.yml
sed -i "s|CA_MCP_ALLOWED_HOSTS=localhost:8443|CA_MCP_ALLOWED_HOSTS=${IP}:8443|g" docker-compose.yml
sed -i "s|127.0.0.1:8443:8443|8443:8443|g" docker-compose.yml
```

A última substituição libera a porta, que por padrão fica publicada apenas no loopback.

---

## TLS por endereço IP

A aplicação exige HTTPS, pois os cookies de sessão são emitidos com flag `Secure`.

### Certificado com SAN de IP

```bash
mkdir -p certs
openssl req -x509 -nodes -newkey rsa:2048 -days 825 \
  -keyout certs/lab.key -out certs/lab.crt \
  -subj "/CN=192.168.x.x" \
  -addext "subjectAltName=IP:192.168.x.x,DNS:localhost"
chmod 644 certs/lab.crt certs/lab.key

openssl x509 -in certs/lab.crt -noout -text | grep -A2 "Subject Alternative Name"
# IP Address:192.168.x.x, DNS:localhost
```

### Caddy definido por porta

O SNI não transporta endereço IP (RFC 6066), então o site é definido pela porta.

```yaml
    volumes:
      - ./db/caddy:/data/caddy
      - ./certs:/certs:ro
    command: |
      sh -c 'echo ":8443 {
      reverse_proxy /api/* backend:8000
      reverse_proxy /mcp* mcp:8001
      reverse_proxy /* frontend:3000
      tls /certs/lab.crt /certs/lab.key
      }" > Caddyfile && caddy run'
```

```bash
docker compose config > /dev/null && echo "YAML OK"
docker compose up -d
curl -k https://192.168.x.x:8443 -o /dev/null -w "%{http_code}\n"
# 302
```

---

## Administração

```bash
docker compose exec backend python manage.py createsuperuser
docker compose exec backend python manage.py changepassword usuario@dominio

docker compose exec backend python manage.py shell -c \
  "from django.contrib.auth import get_user_model; \
   print(list(get_user_model().objects.values_list('email', flat=True)))"
```

- Login por **email**, não por nome de usuário
- A imagem usa `uv`, não `poetry`
- SQLite persiste em `./db/`, sobrevive a `docker compose down`
- Backend leva cerca de 40 s até passar no healthcheck

---

## Estrutura da ISO 27001

| Parte | Conteúdo | Avaliáveis |
|---|---|---|
| Cláusulas 4 a 10 | Sistema de gestão | 30 |
| Anexo A | Controles de segurança | 93 |
| **Total** | | **123** |

| Tema do Anexo A | Faixa |
|---|---|
| Organizacional | A.5.1 a A.5.37 |
| Pessoas | A.6.1 a A.6.8 |
| Físico | A.7.1 a A.7.14 |
| Tecnológico | A.8.1 a A.8.34 |

Numeração colide entre as partes. Cláusula 5.1 trata de liderança, controle A.5.1 trata de políticas.

---

## Modelagem

```
Domínio                      segrega permissão e dados
  └── Perímetro              escopo do SGSI
        └── Audit            framework instanciado
              └── Requisito
                    ├── Applied control   prática implementada
                    └── Evidence          prova de operação
```

Applied control e requisito têm relação de **muitos para muitos**. Uma prática atende vários requisitos, o que elimina duplicação de registro.

ISO 27001:2022 e NIST CSF v2.0 vêm carregados por padrão. A plataforma projeta a avaliação de um framework sobre o outro automaticamente.

---

## Configuração do audit

| Opção | Recomendação | Efeito se ligada |
|---|---|---|
| Anchor N/A to target score | Desligado | Requisitos não aplicáveis pontuam como máximo, inflando o resultado |
| Suggest controls | Não usar | Cria applied controls automáticos sem correspondência com a realidade |
| Score calculation | Average | Comportamento mais previsível |
| Locked | Desligado no preenchimento | Bloqueia edição dos itens |
| Version | Incrementar por ciclo | Preserva histórico para série temporal |

### Campos por requisito

| Campo | Função |
|---|---|
| Status | Andamento da avaliação |
| Result | Conformidade do requisito |
| Observation | Justificativa, sustenta a avaliação |
| Applied control | Prática vinculada |
| Evidence | Prova anexada |

---

## Métricas de cobertura

| Métrica | Mede |
|---|---|
| Controls coverage | Requisitos com applied control vinculado |
| Evidence coverage | Requisitos com evidência anexada |

Requisito conforme sem nenhum dos dois é conformidade declarada sem lastro.

```
Requisito da norma   →   o que é exigido
Applied control      →   prática implementada
Evidence             →   prova de que opera
```

---

## Referências

- [CISO Assistant](https://github.com/intuitem/ciso-assistant-community)
- [Documentação](https://intuitem.gitbook.io/ciso-assistant)
- [Caddy, diretiva tls](https://caddyserver.com/docs/caddyfile/directives/tls)
- RFC 6066, TLS Extensions, Server Name Indication
- ISO/IEC 27001:2022 e ISO/IEC 27002:2022
- NIST Cybersecurity Framework 2.0

---
