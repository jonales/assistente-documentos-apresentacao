# Assistente de Documentos

Um assistente que responde dúvidas sobre os **sistemas instalados na empresa** a partir dos manuais em PDF.
O administrador envia os PDFs e o assistente responde no chat, em **português ou inglês**, citando o arquivo
e a página de onde tirou a resposta.

Tudo roda nos servidores da empresa, com **modelos de linguagem locais** (Ollama): os documentos não saem da
rede interna.

> Esta página é atualizada automaticamente a partir do repositório do projeto.

<p align="center">
  <img src="telas/05-chat-en-claro.png" alt="Chat respondendo uma pergunta com a fonte citada" width="49%">
  <img src="telas/04-chat-pt-escuro.png" alt="Chat em português, tema escuro" width="49%">
</p>

## Como funciona

1. O **administrador** cria um **acervo** para cada sistema (ERP, RH, portal...) e envia os PDFs dele.
2. Os PDFs entram numa **fila** e são processados um por vez: o texto é dividido em trechos e cada trecho vira
   um vetor de busca.
3. O **usuário** pergunta no chat. O assistente busca os trechos mais parecidos **só nos acervos que a equipe
   dele pode consultar** e o modelo escreve a resposta com base neles, citando a fonte.
4. O usuário avalia a resposta (👍/👎), e o administrador acompanha a **qualidade** com perguntas de teste.

## Funcionalidades

### Chat com fontes
Respostas em streaming, no idioma da pergunta, com as fontes citadas. Cada pessoa tem o próprio histórico
de conversas e pode restringir uma conversa a um sistema específico.

### Acervos e equipes
Cada PDF pertence a um acervo (um sistema). Equipes definem quem conversa sobre quais acervos; uma pessoa pode
estar em várias equipes. A regra é conferida a cada pergunta: tirar alguém de uma equipe vale na hora.

<p align="center">
  <img src="telas/07-acervos.png" alt="Acervos" width="49%">
  <img src="telas/08-equipes.png" alt="Equipes" width="49%">
</p>

### Fila de processamento
Mesmo enviando 100 PDFs de uma vez, eles são processados **um por vez**, na ordem de envio, para o chat
continuar rápido. A tela mostra o que está em andamento e a posição de cada arquivo.

<p align="center"><img src="telas/06-documentos-fila.png" alt="Fila de processamento dos PDFs" width="80%"></p>

### Qualidade das respostas
O administrador mantém perguntas de teste com a resposta esperada. Cada rodada de avaliação mede se o
assistente **citou o PDF certo** e **o quanto a resposta se parece com a esperada**, guardando a configuração
usada, para comparar antes e depois de qualquer ajuste. Respostas aprovadas pelos usuários viram perguntas
de teste com um clique.

<p align="center">
  <img src="telas/09-qualidade.png" alt="Rodadas de avaliação" width="49%">
  <img src="telas/10-qualidade-detalhe.png" alt="Detalhe de uma rodada" width="49%">
</p>

### Acesso seguro
- Login com **verificação em duas etapas** (app autenticador) para todos, com **códigos de recuperação** para
  quem perder o celular.
- **Bloqueio temporário** depois de tentativas erradas de senha ou de código.
- Sessão encerrada após **30 minutos sem uso**; senha provisória obrigatoriamente trocada no primeiro acesso.
- **HTTPS** com certificado automático e cabeçalhos de segurança.

<p align="center">
  <img src="telas/01-login.png" alt="Login" width="32%">
  <img src="telas/02-2fa-cadastro.png" alt="Cadastro do app autenticador" width="32%">
  <img src="telas/03-codigos-recuperacao.png" alt="Códigos de recuperação" width="32%">
</p>

### Inglês e português, claro e escuro, no computador e no celular
A interface segue o idioma do navegador e o tema do sistema operacional, e cada pessoa pode escolher os seus.

<p align="center">
  <img src="telas/11-celular-chat.png" alt="Chat no celular" width="28%">
  <img src="telas/14-celular-perdeu-celular.png" alt="Entrar com código de recuperação no celular" width="28%">
</p>

## Arquitetura

```mermaid
flowchart LR
    U["Usuários<br/>(navegador)"] -->|HTTPS| P["Proxy HTTPS<br/>Caddy"]
    P --> A["Aplicação<br/>Java 21 · Spring Boot 4 · Spring AI 2"]
    A --> DB[("PostgreSQL + pgvector<br/>usuários, conversas, PDFs,<br/>trechos e vetores")]
    A -->|"perguntas e trechos"| L["Servidor de LLM<br/>Ollama"]
    L --- M1["qwen2.5:7b<br/>escreve as respostas"]
    L --- M2["bge-m3<br/>busca multilíngue"]

    subgraph S1 ["Servidor da aplicação"]
        P
        A
        DB
    end
    subgraph S2 ["Servidor de LLM"]
        L
        M1
        M2
    end
```

**Fluxo de uma pergunta (RAG):**

```mermaid
sequenceDiagram
    actor U as Usuário
    participant A as Aplicação
    participant V as pgvector
    participant M as Modelo (Ollama)
    U->>A: pergunta (PT ou EN)
    A->>A: acervos que a equipe pode consultar
    A->>M: vetor da pergunta (bge-m3)
    A->>V: 5 trechos mais parecidos, só nesses acervos
    A->>M: instruções + trechos + histórico + pergunta
    M-->>U: resposta em streaming
    A-->>U: fontes citadas (arquivo e página)
```

| Camada | Tecnologia |
|---|---|
| Linguagem e framework | Java 21, Spring Boot 4.1, Spring Security 7 (2FA), Spring AI 2 |
| Telas | Thymeleaf, HTML e CSS próprios, sem framework de JavaScript |
| Banco | PostgreSQL 17 com pgvector, migrações com Flyway, sessões no banco |
| Modelos | Ollama: `qwen2.5:7b` (respostas) e `bge-m3` (busca, multilíngue) |
| Infraestrutura | Docker Compose; proxy HTTPS Caddy; servidor de LLM separado; opção de cluster com Docker Swarm |
| Qualidade | Testes automatizados com Postgres real (Testcontainers) e roteiro de telas no Firefox |

A aplicação não guarda nada em disco (PDFs e sessões ficam no banco), então pode rodar em mais de um servidor.

## Próximos passos

- Opção de usar **IAs pagas** (OpenAI, Anthropic, Gemini) por acervo, mantendo a busca local.
- **Fine-tuning** com as perguntas de teste e as respostas aprovadas pelos usuários.
