# MOTIVA — Sprint 4: Spring Boot + Spring Data JPA + API REST

API REST do sistema **MOTIVA** de monitoramento e priorização de roçada de vegetação em rodovias.
Nesta sprint, o protótipo de console com **JDBC puro** da Sprint 3 vira uma API REST com
**Spring Boot 4.1**, **Spring Data JPA** e **Oracle**. A API reaproveita as mesmas tabelas que
modelamos na Sprint 3.

> Na Sprint 3, cada operação levava cerca de 40 linhas (`Connection`, `PreparedStatement`, `ResultSet`, `finally`).
> Na Sprint 4, a mesma operação é uma linha: `repository.save(trecho);`

---

## Sumário

1. [Tecnologias](#1-tecnologias)
2. [Arquitetura em camadas](#2-arquitetura-em-camadas)
3. [Da Sprint 3 para a Sprint 4](#3-da-sprint-3-para-a-sprint-4)
4. [Como executar](#4-como-executar)
5. [Endpoints](#5-endpoints)
6. [Exemplos cURL](#6-exemplos-curl)
7. [Regras de negócio (motor de prioridade)](#7-regras-de-negócio-motor-de-prioridade)
8. [Tratamento de erros](#8-tratamento-de-erros)
9. [Testes automatizados](#9-testes-automatizados)
10. [Evidências (prints das requisições)](#10-evidências)
11. [Perguntas de reflexão](#11-perguntas-de-reflexão)
12. [Boas práticas de Git](#12-boas-práticas-de-git)

---

## 1. Tecnologias

| Item | Versão / escolha |
| --- | --- |
| Java | 17 |
| Framework | Spring Boot 4.1.x (`spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `spring-boot-starter-validation`) |
| ORM | Hibernate (via Spring Data JPA) |
| Banco | Oracle FIAP (`oracle.fiap.com.br:1521/ORCL`), mesmas tabelas da Sprint 3 |
| Driver | `com.oracle.database.jdbc:ojdbc17` (dependência Maven, sem JAR manual) |
| Build | Maven (`pom.xml`) |
| Testes | JUnit 5 + `@SpringBootTest` + MockMvc + H2 em memória |

## 2. Arquitetura em camadas

```text
src/main/java/br/com/motiva/
├── MotivaApplication.java          # @SpringBootApplication (main)
├── model/                          # @Entity: entidades da Sprint 3 mapeadas com JPA
│   ├── TrechoRodovia.java          #   TRECHO_RODOVIA (herança SINGLE_TABLE)
│   ├── TrechoRodoviaIoT.java       #   mesma tabela, quando CODIGO_SENSOR != null
│   ├── EquipeManutencao.java       #   EQUIPE_MANUTENCAO
│   ├── IntervencaoOperacional.java #   INTERVENCAO_OPERACIONAL (abstrata, SINGLE_TABLE)
│   ├── RocadaMecanizada / RocadaManual / Pulverizacao  # subclasses (discriminador TIPO_INTERVENCAO)
│   ├── RelatorioPrioridade.java    #   RELATORIO_PRIORIDADE
│   ├── MonitoravelViaIoT.java      #   interface da Sprint 2
│   ├── SimNaoConverter.java        #   CHAR(1) 'S'/'N' <-> Boolean
│   └── enums: Especialidade, TipoAmbiente, TipoIntervencao, Prioridade
├── repository/                     # interfaces JpaRepository, SEM SQL (derived queries)
├── service/                        # @Service: regras de negócio + motor de prioridade
├── controller/                     # @RestController: endpoints REST
├── dto/                            # records de entrada/saída (contrato da API)
└── exception/                      # exceções de negócio + @RestControllerAdvice
```

Cada camada só conversa com a vizinha:

```text
Cliente HTTP ──JSON──▶ Controller ──▶ Service ──▶ Repository ──▶ Oracle
                       (HTTP only)   (regras)     (Spring Data)
```

- **Controller**: recebe a requisição, aplica `@Valid` e devolve o código HTTP. Não tem regra de negócio nem `try/catch`.
- **Service**: valida as regras (Sprint 1), decide prioridades e intervenções (Sprint 2) e controla as transações (`@Transactional`).
- **Repository**: interface. O Spring Data gera a implementação e o SQL em tempo de execução.
- **DTOs (`record`)**: separam o contrato da API das entidades JPA. O cliente nunca envia nem recebe uma `@Entity` diretamente.

## 3. Da Sprint 3 para a Sprint 4

| Sprint 3 (JDBC puro) | Sprint 4 (Spring Boot) |
| --- | --- |
| `model/*` + `record` interno de cada DAO | `@Entity` em `model/` |
| `TrechoRodoviaDAO`, `EquipeManutencaoDAO`... (centenas de linhas) | `TrechoRodoviaRepository`, `EquipeManutencaoRepository`... (poucas linhas cada) |
| `ConexaoBD` (Singleton com usuário/senha no código) | `application.properties` + pool HikariCP; credenciais fora do código |
| `TrechoRodoviaDAO.paraObjetoDominio()` decidia `TrechoRodovia` × `TrechoRodoviaIoT` | Herança JPA `SINGLE_TABLE` + `@DiscriminatorFormula` sobre `CODIGO_SENSOR` |
| `TIPO_INTERVENCAO` gravado a partir de `getClass().getSimpleName()` | `TIPO_INTERVENCAO` virou a **coluna discriminadora**: o Hibernate instancia a subclasse certa |
| `ativa ? "S" : "N"` repetido em cada DAO | `SimNaoConverter` (`AttributeConverter`) aplicado automaticamente |
| `service/MotorRegrasPrioridade` com `System.out.println` | `MotorRegrasPrioridadeService` (`@Service`) devolvendo JSON |
| `Main.main()` | `@RestController` + HTTP/JSON |
| Teste manual no console | Testes automatizados (MockMvc) + cURL/Postman |

### Sequences

Na Sprint 3, as PKs foram criadas com `GENERATED ALWAYS AS IDENTITY`. As entidades da Sprint 4 usam
`@GeneratedValue(strategy = SEQUENCE)` + `@SequenceGenerator`, ligadas às sequences
`SEQ_EQUIPE_MANUTENCAO`, `SEQ_TRECHO_RODOVIA`, `SEQ_INTERVENCAO_OPERACIONAL` e `SEQ_RELATORIO_PRIORIDADE`.

A migração é **automática e idempotente**. Ao subir, a API executa `src/main/resources/db/migracao-sequences.sql`, que:

1. cria as tabelas da Sprint 3 que ainda não existirem no usuário Oracle (mesma modelagem e constraints);
2. remove o `IDENTITY` da coluna `ID` (os dados continuam lá);
3. cria cada sequence começando no **maior ID existente + 1**, para não colidir com as linhas da Sprint 3;
4. se o banco estiver vazio, cadastra a massa inicial da Sprint 3 (3 equipes e 5 trechos).

Na segunda execução em diante o script não altera nada. O mesmo conteúdo também está em
`sql/01-migracao-sprint3-para-sequences.sql`, para rodar manualmente no SQL Developer, se preferir.

## 4. Como executar

### Pré-requisitos

- JDK 17 ou superior
- Maven 3.9+ (ou o Maven embutido do Eclipse / IntelliJ)
- Acesso ao Oracle da FIAP, com as tabelas da Sprint 3 criadas (se ainda não existirem, rode `sql/02-criacao-do-zero-com-sequences.sql` e, opcionalmente, `sql/03-dados-de-teste.sql`)

### Passo 1: credenciais (nunca no código, nunca no Git)

Copie o arquivo de exemplo e preencha com o seu RM e a sua senha:

```bash
cp application-local.properties.example application-local.properties
```

```properties
spring.datasource.username=RMxxxxxx
spring.datasource.password=sua_senha
```

O `application-local.properties` está no `.gitignore`. Também dá para usar variáveis de ambiente:
`DB_USERNAME`, `DB_PASSWORD` e (opcional) `DB_URL`.

### Passo 2: subir a API

```bash
mvn spring-boot:run
```

No **Eclipse**: `File > Import > Maven > Existing Maven Projects` e selecione a pasta do projeto.
Depois, clique com o botão direito em `MotivaApplication.java` e escolha `Run As > Java Application`.

A API sobe em **http://localhost:8085** (porta configurável pela variável `SERVER_PORT`).

### Problemas comuns

| Erro | Solução |
| --- | --- |
| Eclipse executa `JavaToJavaConverter` em vez da API | Rode pelo arquivo `MotivaApplication.java` (botão direito > Run As > Java Application), e não pelo projeto |
| `Port 8085 was already in use` | Já existe uma instância rodando: pare-a no botão vermelho do Console ou suba com `SERVER_PORT=8086` |
| `ORA-01017: invalid username/password` | Confira o `application-local.properties` |
| Alteração em `src/main/resources` não teve efeito | Selecione o projeto e aperte F5 (Refresh) antes de rodar |

### Passo 3: rodar os testes

```bash
mvn test
```

Os testes usam H2 em memória, então rodam **sem** precisar do Oracle.

## 5. Endpoints

Base URL: `http://localhost:8085`

### Trechos: `/api/trechos`

| Método | Rota | Ação | Sucesso | Erros |
| --- | --- | --- | --- | --- |
| GET | `/api/trechos` | Lista todos (filtros opcionais `?tipoAmbiente=UMIDO`, `?alturaMinima=50`) | 200 | 400 |
| GET | `/api/trechos/{id}` | Busca por ID | 200 | 404, 400 |
| POST | `/api/trechos` | Cria (com `codigoSensor` → `TrechoRodoviaIoT`) | **201** + `Location` | 400 |
| PUT | `/api/trechos/{id}` | Atualiza | 200 | 400, 404 |
| DELETE | `/api/trechos/{id}` | Remove | **204** | 404, 409 (tem histórico) |
| POST | `/api/trechos/simulacao-crescimento?dias=7` | Simula o crescimento da vegetação (Sprint 1) | 200 | 400 |

### Equipes: `/api/equipes`

| Método | Rota | Ação | Sucesso | Erros |
| --- | --- | --- | --- | --- |
| GET | `/api/equipes` | Lista (filtros `?especialidade=MANUAL`, `?nome=preventiva`) | 200 | 400 |
| GET | `/api/equipes/{id}` | Busca por ID | 200 | 404 |
| POST | `/api/equipes` | Cria | **201** + `Location` | 400 |
| PUT | `/api/equipes/{id}` | Atualiza (inclusive ativar/inativar) | 200 | 400, 404 |
| DELETE | `/api/equipes/{id}` | Remove | **204** | 404, 409 (tem histórico) |

### Intervenções: `/api/intervencoes`

| Método | Rota | Ação | Sucesso | Erros |
| --- | --- | --- | --- | --- |
| GET | `/api/intervencoes` | Histórico (filtros `?trechoId=1`, `?equipeId=2`) | 200 | 400 |
| GET | `/api/intervencoes/{id}` | Busca por ID | 200 | 404 |
| POST | `/api/intervencoes` | **Executa** a intervenção no trecho (a altura muda) e grava no histórico | **201** + `Location` | 400 |
| PUT | `/api/intervencoes/{id}` | Corrige a equipe e/ou o texto do resultado | 200 | 400, 404 |
| DELETE | `/api/intervencoes/{id}` | Remove do histórico | **204** | 404 |

`tipo` aceita `ROCADA_MECANIZADA`, `ROCADA_MANUAL` ou `PULVERIZACAO`.

### Relatórios de prioridade: `/api/relatorios`

| Método | Rota | Ação | Sucesso | Erros |
| --- | --- | --- | --- | --- |
| POST | `/api/relatorios` | Gera o relatório a partir dos trechos atuais e grava no histórico | **201** + `Location` | — |
| GET | `/api/relatorios` | Lista o histórico (mais recente primeiro) | 200 | — |
| GET | `/api/relatorios/periodo?inicio=AAAA-MM-DD&fim=AAAA-MM-DD` | Histórico por período (datas inclusivas) | 200 | 400 |
| GET | `/api/relatorios/{id}` | Busca um relatório | 200 | 404 |

## 6. Exemplos cURL

> No Windows, use `curl.exe` no PowerShell (no PowerShell, `curl` é um alias de outro comando) ou rode no Git Bash.
> Também há uma coleção pronta para o Postman em `docs/MOTIVA-Sprint4.postman_collection.json`.

### Equipes

```bash
# Criar (201)
curl -i -X POST http://localhost:8085/api/equipes \
  -H "Content-Type: application/json" \
  -d '{"nomeEquipe":"Equipe Mecanizada A","especialidade":"MECANIZADA","ativa":true}'

# Listar (200) e filtrar por especialidade (derived query)
curl -i http://localhost:8085/api/equipes
curl -i "http://localhost:8085/api/equipes?especialidade=MANUAL"

# Atualizar / inativar (200)
curl -i -X PUT http://localhost:8085/api/equipes/1 \
  -H "Content-Type: application/json" \
  -d '{"nomeEquipe":"Equipe Mecanizada A","especialidade":"MECANIZADA","ativa":false}'

# Remover (204, ou 409 se a equipe já tiver intervenções)
curl -i -X DELETE http://localhost:8085/api/equipes/1
```

### Trechos

```bash
# Criar trecho comum (201)
curl -i -X POST http://localhost:8085/api/trechos \
  -H "Content-Type: application/json" \
  -d '{"kmInicial":18,"kmFinal":19,"descricao":"Acostamento com acesso dificil","tipoAmbiente":"SECO","alturaVegetacaoCm":92,"areaSinalizacao":false,"acessoDificil":true}'

# Criar trecho com sensor IoT (201): vira TrechoRodoviaIoT
curl -i -X POST http://localhost:8085/api/trechos \
  -H "Content-Type: application/json" \
  -d '{"kmInicial":16,"kmFinal":17,"descricao":"Trecho com sensor","tipoAmbiente":"UMIDO","alturaVegetacaoCm":68,"codigoSensor":"IOT-016"}'

# Listar (200), filtrar e buscar por ID (200 / 404)
curl -i http://localhost:8085/api/trechos
curl -i "http://localhost:8085/api/trechos?tipoAmbiente=UMIDO&alturaMinima=50"
curl -i http://localhost:8085/api/trechos/1
curl -i http://localhost:8085/api/trechos/99999

# Atualizar (200)
curl -i -X PUT http://localhost:8085/api/trechos/1 \
  -H "Content-Type: application/json" \
  -d '{"kmInicial":10,"kmFinal":11,"descricao":"Trecho seco revisado","tipoAmbiente":"SECO","alturaVegetacaoCm":40,"areaSinalizacao":false,"acessoDificil":false}'

# Validação (400): altura negativa
curl -i -X POST http://localhost:8085/api/trechos \
  -H "Content-Type: application/json" \
  -d '{"kmInicial":1,"kmFinal":2,"tipoAmbiente":"SECO","alturaVegetacaoCm":-5}'

# Simular 7 dias de crescimento (200)
curl -i -X POST "http://localhost:8085/api/trechos/simulacao-crescimento?dias=7"

# Remover (204)
curl -i -X DELETE http://localhost:8085/api/trechos/1
```

### Intervenções

```bash
# Executar uma roçada mecanizada no trecho 5 com a equipe 1 (201)
curl -i -X POST http://localhost:8085/api/intervencoes \
  -H "Content-Type: application/json" \
  -d '{"tipo":"ROCADA_MECANIZADA","trechoId":5,"equipeId":1}'

# Equipe com especialidade errada (400)
curl -i -X POST http://localhost:8085/api/intervencoes \
  -H "Content-Type: application/json" \
  -d '{"tipo":"ROCADA_MECANIZADA","trechoId":5,"equipeId":2}'

# Histórico de um trecho (200)
curl -i "http://localhost:8085/api/intervencoes?trechoId=5"
```

### Relatórios

```bash
# Gerar relatório (201)
curl -i -X POST http://localhost:8085/api/relatorios

# Histórico (200)
curl -i http://localhost:8085/api/relatorios

# Por período (200) e período invertido (400)
curl -i "http://localhost:8085/api/relatorios/periodo?inicio=2026-10-01&fim=2026-10-31"
curl -i "http://localhost:8085/api/relatorios/periodo?inicio=2026-12-31&fim=2026-01-01"
```

Resposta real do `POST /api/relatorios` no Oracle da FIAP (23/10/2026), com dois dos cinco itens:

```json
{
  "id": 1,
  "dataGeracao": "2026-10-23T23:16:46.0179373",
  "totalTrechos": 5,
  "qtBaixa": 1, "qtMedia": 1, "qtAlta": 2, "qtCritica": 1,
  "resumo": "KM 10-11: BAIXA (monitorar); KM 12-13: MEDIA (Pulverizacao); KM 14-15: ALTA (RocadaManual); KM 16-17: ALTA (RocadaMecanizada); KM 18-19: CRITICA (RocadaManual);",
  "itens": [
    {
      "trechoId": 4,
      "trecho": "KM 16 ao KM 17 - Trecho com sensor IoT instalado",
      "tipoAmbiente": "UMIDO",
      "leituraIoTCm": 88.42,
      "alturaAnteriorCm": 88.42,
      "prioridade": "ALTA",
      "acaoRecomendada": "RocadaMecanizada",
      "intervencaoId": 4,
      "resultado": "Rocada mecanizada executada pela Equipe Mecanizada A. Vegetacao reduzida para 8 cm.",
      "alturaAtualCm": 8
    },
    {
      "trechoId": 5,
      "trecho": "KM 18 ao KM 19 - Trecho de acostamento com acesso dificil",
      "tipoAmbiente": "SECO",
      "leituraIoTCm": null,
      "alturaAnteriorCm": 97.6,
      "prioridade": "CRITICA",
      "acaoRecomendada": "RocadaManual",
      "intervencaoId": 5,
      "resultado": "Rocada manual executada pela Equipe Manual B. Servico indicado por seguranca ou acesso dificil.",
      "alturaAtualCm": 10
    }
  ]
}
```

## 7. Regras de negócio (motor de prioridade)

Toda a lógica fica em `MotorRegrasPrioridadeService` (camada **Service**), nunca no Controller.
Os limiares são os mesmos das Sprints 2 e 3:

| Altura da vegetação | Prioridade |
| --- | --- |
| ≥ 90 cm | **CRÍTICA** |
| ≥ 70 cm | **ALTA** |
| ≥ 45 cm | **MÉDIA** |
| < 45 cm | **BAIXA** |

| Condição | Intervenção | Equipe exigida | Efeito no trecho |
| --- | --- | --- | --- |
| ≥ 70 cm **e** (acesso difícil **ou** área de sinalização) | `RocadaManual` | MANUAL | altura → 10 cm |
| ≥ 70 cm (demais casos) | `RocadaMecanizada` | MECANIZADA | altura → 8 cm |
| ≥ 45 cm **e** ambiente UMIDO | `Pulverizacao` | PREVENTIVA | altura − 5 cm |
| demais casos | monitorar | — | — |

Fluxo do `POST /api/relatorios` (uma única transação):

1. lê todos os trechos;
2. trechos com sensor (`MonitoravelViaIoT`) transmitem a leitura atual (polimorfismo da Sprint 2);
3. classifica cada trecho e define a intervenção;
4. escolhe a primeira equipe **ativa** da especialidade exigida, executa o serviço (o método `executarServico()` é polimórfico) e grava em `INTERVENCAO_OPERACIONAL`;
5. grava o resumo em `RELATORIO_PRIORIDADE` e devolve o detalhe de cada trecho no JSON.

Outras regras no Service:

- `kmInicial ≥ 0`, `kmFinal ≥ kmInicial` e `alturaVegetacaoCm ≥ 0` (validações da Sprint 1);
- um trecho comum não vira trecho IoT nem o contrário (o tipo é definido na criação);
- uma intervenção só pode ser executada por uma equipe **ativa** da **especialidade correta**;
- trecho ou equipe com histórico de intervenções não pode ser removido (409). A alternativa para a equipe é inativá-la;
- no período, `inicio` não pode ser posterior a `fim`.

## 8. Tratamento de erros

O `GlobalExceptionHandler` (`@RestControllerAdvice`) converte as exceções em códigos HTTP e em um JSON padronizado:

```json
{
  "timestamp": "2026-10-23T22:20:11.108",
  "status": 400,
  "erro": "Bad Request",
  "mensagem": "Dados de entrada invalidos.",
  "caminho": "/api/trechos",
  "campos": [ { "campo": "alturaVegetacaoCm", "mensagem": "alturaVegetacaoCm nao pode ser negativa" } ]
}
```

| Situação | Código |
| --- | --- |
| Recurso inexistente (`RecursoNaoEncontradoException`) | 404 |
| Bean Validation (`@Valid`), regra de negócio, JSON inválido, parâmetro inválido ou ausente | 400 |
| Remover trecho ou equipe com histórico; violação de integridade no banco | 409 |
| Método HTTP não suportado | 405 |
| Erro inesperado (sem stack trace na resposta) | 500 |

## 9. Testes automatizados

```bash
mvn test
```

- `MotorRegrasPrioridadeServiceTest`: testes unitários dos limiares e das intervenções, sem Spring e sem banco.
- `MotivaApiIntegrationTest`: `@SpringBootTest` + MockMvc percorrendo Controller → Service → Repository → H2. Cobre:
  - CRUD completo de trechos e de equipes (201/200/204/404);
  - validações 400 (Bean Validation, regra de negócio, enum inválido, ID inválido);
  - herança JPA (trecho com sensor volta do banco como `TrechoRodoviaIoT`);
  - derived queries (filtros);
  - execução de intervenção, especialidade errada (400) e remoção de trecho com histórico (409);
  - geração de relatório, histórico e consulta por período (200/400).

## 10. Evidências

Em 23/10/2026, a API rodou no Eclipse conectada ao **Oracle da FIAP (19.3)** e **27 requisições** foram executadas.
Todas devolveram o código HTTP esperado. O índice completo está em [`docs/evidencias/README.md`](docs/evidencias/README.md),
há um print por requisição, e o corpo de cada requisição e de cada resposta está em
[`requisicoes-oracle.json`](docs/evidencias/requisicoes-oracle.json).

| # | Requisição | Status | Print |
| --- | --- | --- | --- |
| 02 | `POST /api/equipes`: criar equipe | **201** + `Location` | [ver](docs/evidencias/02-post-criar-equipe.png) |
| 07 | `POST /api/trechos`: criar trecho | **201** + `Location` | [ver](docs/evidencias/07-post-criar-trecho.png) |
| 09 | `PUT /api/trechos/6`: atualizar trecho | **200** | [ver](docs/evidencias/09-put-atualizar-trecho.png) |
| 10 | `GET /api/trechos?tipoAmbiente=UMIDO&alturaMinima=50`: derived query | **200** | [ver](docs/evidencias/10-get-filtrar-trechos-derived-query.png) |
| 11 | `GET /api/trechos/99999`: inexistente | **404** | [ver](docs/evidencias/11-get-trecho-inexistente.png) |
| 12 | `POST /api/trechos` com altura negativa (Bean Validation) | **400** | [ver](docs/evidencias/12-post-validacao-altura-negativa.png) |
| 14 | `POST /api/intervencoes` com equipe de especialidade errada | **400** | [ver](docs/evidencias/14-post-regra-de-negocio-equipe-com-especialidade-errada.png) |
| 15 | `POST /api/intervencoes`: executa a roçada (altura → 8 cm) | **201** | [ver](docs/evidencias/15-post-executar-intervencao-rocada-mecanizada.png) |
| 17 | `DELETE /api/trechos/6` com histórico | **409** | [ver](docs/evidencias/17-delete-remover-trecho-com-historico-conflito.png) |
| 20 | `DELETE /api/trechos/6` | **204** | [ver](docs/evidencias/20-delete-remover-trecho.png) |
| 23 | `POST /api/relatorios`: gerar relatório | **201** | [ver](docs/evidencias/23-post-gerar-relatorio-de-prioridade.png) |
| 24 | `GET /api/relatorios`: histórico | **200** | [ver](docs/evidencias/24-get-historico-de-relatorios.png) |
| 25 | `GET /api/relatorios/periodo?inicio=2026-10-23&fim=2026-10-23` | **200** | [ver](docs/evidencias/25-get-relatorios-por-periodo.png) |
| 26 | `GET /api/relatorios/periodo` com período invertido | **400** | [ver](docs/evidencias/26-get-periodo-invertido.png) |

Testes automatizados (`mvn test` no Eclipse, mesma data): **12 testes, 0 falhas, 0 erros**
(8 em `MotivaApiIntegrationTest` + 4 em `MotorRegrasPrioridadeServiceTest`).

## 11. Perguntas de reflexão

**1. Por que o Repository é uma interface e não uma classe? Quem escreve a implementação e quando?**

Porque nós só declaramos **o que** queremos (o contrato) e deixamos o **como** com o framework. Ao subir a
aplicação, o Spring Data JPA encontra as interfaces que estendem `JpaRepository` e cria, em **tempo de
execução**, um proxy que as implementa (por baixo é a classe `SimpleJpaRepository`, que usa o
`EntityManager` do Hibernate). Esse objeto é registrado como bean e injetado nos Services pelo construtor.
Por isso não existe nenhuma classe `TrechoRodoviaRepositoryImpl` no projeto: as operações de CRUD vêm de
`JpaRepository`, e os métodos como `findByTipoAmbienteOrderByKmInicialAsc` são implementados a partir do nome.

**2. O pattern DAO da Sprint 3 "morreu" na migração ou só mudou de forma?**

Só mudou de forma. A ideia do DAO continua igual: isolar o acesso a dados atrás de uma interface, para
que o resto do sistema não saiba que existe SQL. Na Sprint 3, nós mesmos escrevíamos o DAO: abríamos a
conexão, montávamos o `PreparedStatement`, convertíamos o `ResultSet` em objeto e fechávamos tudo no
`finally`. Na Sprint 4, o `TrechoRodoviaRepository` **é** o DAO, só que o framework gera a implementação. O
pattern foi absorvido pelo Spring Data, e o trabalho repetitivo saiu do nosso código.

**3. Por que a validação `alturaVegetacaoCm >= 0` (Sprint 1) deve ficar no Service e não no Controller?**

Porque é uma **regra de negócio**, e não um detalhe do HTTP. O Controller é só uma das portas de entrada do
sistema. Amanhã a mesma regra pode ser usada por um job agendado, por uma fila de mensagens vinda dos sensores
IoT ou pelo próprio motor de prioridade, que altera alturas sem passar por nenhum Controller. Se a validação
estivesse no Controller, esses caminhos passariam por cima dela. No Service, ela vale para **qualquer**
chamador, fica num único lugar e pode ser testada sem HTTP. O `@PositiveOrZero` do DTO é só uma primeira
barreira, para dar uma resposta 400 rápida. A garantia é do Service.

**4. No JDBC puro vocês escreviam SQL. Onde está o SQL do `findByTipo()`? Quem o gerou?**

Ele não está em nenhum arquivo do projeto. Ao iniciar a aplicação, o Spring Data JPA faz o *parse* do
**nome do método** (`findBy` + `TipoAmbiente` + `OrderBy` + `KmInicial` + `Asc`), confere essas
propriedades na entidade `TrechoRodovia` e monta a consulta (JPQL/Criteria). Na execução, o **Hibernate**
traduz essa consulta para o SQL do dialeto Oracle, usando os nomes de tabela e coluna das anotações
`@Table`/`@Column`. Com `spring.jpa.show-sql=true`, o SQL gerado aparece no console, algo como:

```sql
select tr1_0.ID, case when tr1_0.CODIGO_SENSOR is null then 'PADRAO' else 'IOT' end, tr1_0.ACESSO_DIFICIL, ...
from TRECHO_RODOVIA tr1_0
where tr1_0.TIPO_AMBIENTE=?
order by tr1_0.KM_INICIAL
```

Se o nome do método tiver uma propriedade que não existe, a aplicação **nem sobe**. O erro aparece na
inicialização, e não em produção.

## 12. Boas práticas de Git

- Commits pequenos e incrementais, um por etapa da migração (estrutura → entidades → repositories → services → controllers → tratamento de erros → testes → documentação), com mensagens no padrão *Conventional Commits* (`feat:`, `test:`, `docs:`...).
- `.gitignore` ignora `target/`, arquivos de IDE (`.classpath`, `.project`, `.settings/`, `.idea/`) e o `application-local.properties`.
- **Nenhuma senha real no repositório**: o `application.properties` só tem placeholders e lê as credenciais de variáveis de ambiente ou do arquivo local ignorado.
