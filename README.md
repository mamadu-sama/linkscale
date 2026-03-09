<div align="center">

# 🔗 LinkScale: Encurtador de URL Enterprise

### Ultra-rápido. Protegido contra abuso. Construído para milhões de redirecionamentos por hora.

[![Java](https://img.shields.io/badge/Java-17%2F21-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](https://www.java.com)
[![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.x-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org)
[![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)](https://redis.io)
[![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=for-the-badge&logo=mongodb&logoColor=white)](https://www.mongodb.com)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com)

[Sobre](#-sobre-o-projeto) •
[Arquitetura](#-arquitetura-e-decisões-técnicas) •
[Tecnologias](#-tecnologias) •
[Como Rodar](#-como-rodar) •
[Endpoints](#-endpoints) •
[Rate Limiting](#-rate-limiting) •
[Autor](#-autor)

</div>

---

## 📌 Sobre o Projeto

O **LinkScale** é um encurtador de URL de alta performance construído para cenários enterprise. O negócio parece simples: receber uma URL longa e devolver um alias curto. A complexidade está em fazer isso com **latência de milissegundos**, **sem colisões de ID** e **protegido contra abuso de bots**, mesmo sob milhões de requisições por hora.

> "Um encurtador de URL é, no fundo, um exercício de trade-off. Cada decisão de arquitetura impacta diretamente a latência que o usuário final sente no clique."

---

## 🏗️ Arquitetura e Decisões Técnicas

### Visão Geral do Fluxo

```
POST /v1/shorten                     GET /{id}
      │                                   │
      ▼                                   ▼
[ Rate Limiter (Bucket4j) ]     [ Redis Cache ] ──▶ HIT: 302 Redirect ⚡
      │                                   │
      ▼                                   ▼ MISS
[ Base62 Encoding ]             [ PostgreSQL ] ──▶ 302 Redirect + popula cache
      │
      ▼
[ PostgreSQL + Redis ]
```

---

### 1. 🔡 Algoritmo de Encurtamento: Base62 Encoding

IDs sequenciais são previsíveis. UUIDs são longos demais. A solução é o **Base62**, que codifica um número usando 62 caracteres (`a-z`, `A-Z`, `0-9`), gerando aliases curtos, únicos e sem símbolos especiais.

**Como funciona:**

```java
private static final String ALPHABET = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
private static final int BASE = ALPHABET.length(); // 62

public String encode(long id) {
    StringBuilder sb = new StringBuilder();
    while (id > 0) {
        sb.append(ALPHABET.charAt((int)(id % BASE)));
        id /= BASE;
    }
    return sb.reverse().toString();
}
```

Com 7 caracteres em Base62, é possível representar mais de **3,5 trilhões de combinações únicas**, eliminando na prática qualquer risco de colisão em escala.

**Por que não UUID?**
`https://lsc.al/3f4a9b2` é infinitamente melhor que `https://lsc.al/3fa85f64-5717-4562-b3fc`. Links curtos precisam ser, acima de tudo, curtos.

---

### 2. ⚡ Performance no Redirecionamento: Redis como Camada de Cache

O endpoint `GET /{id}` é o coração do sistema e precisa ser o mais rápido possível. Consultar o PostgreSQL a cada clique seria um suicídio de performance sob carga.

**Estratégia adotada: Cache-Aside**

```
1. Recebe GET /{id}
2. Consulta Redis → HIT? Retorna 302 imediatamente. Fim.
3. MISS? Consulta PostgreSQL, popula o cache, retorna 302.
```

O tempo de resposta médio com cache quente fica abaixo de **5ms**. Sem cache, uma consulta ao banco sob carga pode ultrapassar 200ms.

**Tratamento de expiração:**
Quando um link expira no banco, o TTL do Redis é configurado para o mesmo tempo de expiração (`expiresIn`). Isso garante que o cache e o banco expirem de forma sincronizada, sem necessidade de invalidação manual.

```java
redisTemplate.opsForValue().set(
    "url:" + id,
    originalUrl,
    link.getExpiresIn(),
    TimeUnit.SECONDS
);
```

---

### 3. 🚦 Rate Limiting: Proteção contra Bots e Abuso

Para proteger a infraestrutura, um mesmo IP não pode encurtar mais de **10 URLs por minuto**. A implementação usa **Bucket4j** com Redis como backend distribuído, garantindo que o limite funcione mesmo com múltiplas instâncias da aplicação rodando.

```java
Bandwidth limit = Bandwidth.classic(10, Refill.greedy(10, Duration.ofMinutes(1)));
Bucket bucket = bucketMap.computeIfAbsent(ipAddress, k ->
    Bucket.builder().addLimit(limit).build()
);

if (!bucket.tryConsume(1)) {
    throw new RateLimitExceededException("Limite de 10 requisições por minuto atingido.");
}
```

Quando o limite é atingido, a API retorna `429 Too Many Requests` com o header `Retry-After` indicando quando o bucket será recarregado.

---

### 4. 📊 Separação de Responsabilidades: PostgreSQL vs. MongoDB

| Banco | Responsabilidade |
|---|---|
| **PostgreSQL** | Persistência dos links (URL original, alias, expiração) |
| **MongoDB** | Logs de acesso (timestamp, User-Agent, IP por clique) |
| **Redis** | Cache de redirecionamento e controle de Rate Limiting |

Essa separação mantém o banco relacional enxuto e focado apenas na leitura/escrita de links. Os logs de analytics, que crescem de forma não estruturada e em alto volume, ficam no MongoDB, que é otimizado exatamente para esse padrão de escrita.

---

## 🛠️ Tecnologias

| Camada | Tecnologia | Função |
|---|---|---|
| Linguagem | Java 17/21 | Core da aplicação |
| Framework | Spring Boot 3.x | API REST |
| Banco Relacional | PostgreSQL | Persistência dos links |
| Cache | Redis | Redirecionamento de baixa latência |
| Banco de Logs | MongoDB | Analytics de cliques |
| Rate Limiting | Bucket4j + Redis | Proteção contra abuso |
| Containerização | Docker & Docker Compose | Ambiente reproduzível |
| Documentação | Swagger / OpenAPI 3 | Exploração da API |

---

## 🚀 Como Rodar

> **Pré-requisito:** Apenas o [Docker](https://docs.docker.com/get-docker/) instalado.

**1. Clone o repositório**
```bash
git clone https://github.com/mamadu-sama/linkscale.git
cd linkscale
```

**2. Suba o ambiente completo (App + PostgreSQL + Redis + MongoDB)**
```bash
docker-compose up -d
```

A aplicação estará disponível em `http://localhost:8080`

**3. Documentação interativa (Swagger UI)**

`http://localhost:8080/swagger-ui.html`

---

## 📡 Endpoints

### Encurtamento

```
POST /v1/shorten
```

```json
{
  "url": "https://uma-url-muito-longa-com-muitos-parametros.com/exemplo",
  "expiresIn": 3600
}
```

**Resposta (`201 Created`):**
```json
{
  "id": "aB3kR7x",
  "shortUrl": "https://lsc.al/aB3kR7x",
  "originalUrl": "https://uma-url-muito-longa...",
  "expiresAt": "2025-03-09T15:00:00Z"
}
```

---

### Redirecionamento

```
GET /{id}
```

Retorna `302 Found` com header `Location` apontando para a URL original. Servido direto do Redis quando disponível em cache.

---

### Analytics

```
GET /v1/stats/{id}
```

**Resposta (`200 OK`):**
```json
{
  "id": "aB3kR7x",
  "totalClicks": 1482,
  "recentAccesses": [
    { "timestamp": "2025-03-09T14:55:10Z", "userAgent": "Mozilla/5.0..." },
    { "timestamp": "2025-03-09T14:54:02Z", "userAgent": "curl/7.88.1" }
  ]
}
```

---

### Demonstração via cURL

**Encurtar uma URL:**
```bash
curl -X POST http://localhost:8080/v1/shorten \
  -H "Content-Type: application/json" \
  -d '{"url": "https://www.exemplo.com/pagina-muito-longa", "expiresIn": 3600}'
```

**Testar o redirecionamento:**
```bash
curl -v http://localhost:8080/aB3kR7x
# < HTTP/1.1 302 Found
# < Location: https://www.exemplo.com/pagina-muito-longa
```

**Consultar estatísticas:**
```bash
curl http://localhost:8080/v1/stats/aB3kR7x
```

---

## 🚦 Rate Limiting

| Regra | Valor |
|---|---|
| Máximo de encurtamentos por IP | 10 por minuto |
| Resposta ao exceder o limite | `429 Too Many Requests` |
| Header de retorno | `Retry-After: <segundos>` |

O controle é distribuído via Redis, o que significa que o limite funciona corretamente mesmo quando a aplicação escala horizontalmente com múltiplas instâncias.

---

## 👤 Autor

<div align="center">

**Mamadú Sama**

Desenvolvedor Backend com foco em sistemas de alta performance e alta confiabilidade.

[![Email](https://img.shields.io/badge/Email-mamadusama19%40gmail.com-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:mamadusama19@gmail.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/mamadusama)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/mamadu-sama)

</div>

---

<div align="center">
<sub>Feito com ☕ e obsessão por milissegundos por Mamadú Sama</sub>
</div>
