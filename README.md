# AvalIA— Gerador de Provas com R/Exams + IA

O **AvalIA** (antigo Rexams) é uma interface web (R + Shiny) para professores gerarem provas em **PDF, HTML ou Moodle XML** a partir de um banco de questões no formato [R/Exams](https://www.r-exams.org/), com **valores sorteados a cada versão** (A, B, C…) e gabarito, sem precisar programar. Também converte questões existentes (texto, PDF ou Word) para esse formato com ajuda de IA.

A diferença para um gerador que pede tudo à IA: aqui a **resposta é calculada em R**, dentro de cada questão. A IA só ajuda a *converter* uma questão que você já tem para o formato do banco, e o resultado passa por validações antes de poder ser salvo.

## Sumário

1. [Como funciona](#como-funciona)
2. [Início rápido](#início-rápido)
3. [Usando a interface](#usando-a-interface)
4. [Importar com IA](#importar-com-ia)
5. [O banco de questões](#o-banco-de-questões)
6. [Instalação](#instalação)
7. [Solução de problemas](#solução-de-problemas)
8. [Limitações conhecidas](#limitações-conhecidas)
9. [Estrutura do projeto](#estrutura-do-projeto)
10. [Próximos passos](#próximos-passos)

## Como funciona

1. As questões ficam no arquivo **`banco_questoes.json`**. Cada uma tem `id`, `disciplina`, `tema` e `conteudo_rmd` (o texto `.Rmd` do R/Exams, com um trecho de código R que sorteia os valores, o enunciado, a solução e os metadados).
2. Na interface, o professor escolhe disciplina e temas. As questões disponíveis aparecem numa tabela.
3. Ao clicar em **Gerar avaliação**, o sistema sorteia, entre as questões filtradas, a quantidade pedida. **O conjunto de questões é o mesmo em todas as versões**; o pacote `exams` sorteia de novo os **valores** de cada questão a cada versão.
4. A saída sai no formato escolhido, com ou sem solução, e pode ser baixada em um `.zip`.

Se a prova incluir uma questão **sem `sample()`** (sem sorteio de valores), o painel de status avisa: "Atenção, sem sorteio: ...". Essa questão ficará igual em todas as versões.

## Início rápido

**Com Docker (recomendado):** só é preciso ter Docker e Docker Compose.

```
# 1. crie o arquivo .env ao lado do docker-compose.yml com a sua chave da Groq
GROQ_API_KEY=gsk_sua-chave-aqui

# 2. suba o sistema
docker compose up -d --build
```

Abra `http://localhost:3838/rexams-web`. Para parar: `docker compose down`.

A chave da Groq só é necessária para **Importar com IA**. Gerar provas funciona sem ela.

**Sem Docker:** veja [Instalação](#instalação).

## Usando a interface

### Aba "Gerar prova"

1. **Disciplina**: escolha a disciplina.
2. **Tema**: marque os temas que podem entrar na prova (todos vêm marcados).
3. **Número de questões**: quantas questões a prova terá.
4. **Número de versões**: de 1 a 20 (A, B, C…), cada uma com valores sorteados de novo.
5. **Formato de saída**: PDF (impressão), HTML (online) ou Moodle (XML).
6. **Incluir soluções?**: "Somente prova" ou "Com gabarito".
7. Clique em **Gerar avaliação**. O painel **Status da geração** mostra os arquivos gerados, o aviso de questões sem sorteio ou a mensagem de erro.
8. Clique em **Baixar prova + gabarito** para baixar um `.zip` com os arquivos da última geração.

Detalhes:

- Se o número de questões pedido for maior que o total filtrado, a geração é interrompida com uma mensagem de erro.
- Os arquivos levam o prefixo `prova` (sem solução) ou `prova_gabarito` (com solução).
- No Moodle, o XML já carrega a resposta correta (necessária para a correção automática); por isso a opção de solução não se aplica a ele.

### Aba "Banco de questões"

Ainda é um espaço reservado: a consulta completa, a edição e a exclusão do banco estão previstas. A tabela de questões disponíveis fica na aba **Gerar prova**.

### Aba "Importar com IA"

Serve para **colocar no banco** uma questão que você já tem. A IA **não cria questões**: ela converte o texto para o formato R/Exams e troca os números por variáveis sorteadas.

1. Escolha a **Disciplina** e o **Tema** (você pode digitar um novo).
2. Escolha a origem: **Texto colado** ou **Arquivo PDF ou Word** (`.pdf` ou `.docx`).
3. Clique em **Converter e parametrizar**.
4. Confira a **Pré-visualização** (o código `.Rmd` gerado).
5. Se estiver correto, dê um nome ao arquivo e clique em **Salvar no banco de questões**. A questão é gravada em `banco_questoes.json`.

Antes de oferecer o botão de salvar, o sistema aplica estas verificações. Se alguma falhar, mostra o erro e **não permite salvar**:

| Verificação | O que checa |
|---|---|
| Resposta não vazia | A IA devolveu algum conteúdo. |
| Fidelidade | Pelo menos 85% das palavras do enunciado original foram mantidas (a IA não pode reescrever). |
| Parametrização | Se o original tem números, o `.Rmd` precisa ter `sample()` e variáveis inline. |
| Compilação | A questão compila em **HTML e em PDF** (uma versão de teste). |

A IA usada é o modelo `openai/gpt-oss-20b` da Groq, com `temperature = 0`, e o prompt exige que a resposta seja **calculada em R** e não escrita como número fixo.

> **Atenção:** as verificações garantem que a questão *compila e varia*. Elas **não garantem** que a resposta está certa nem que o valor original continua entre os sorteados. **Confira sempre a pré-visualização** e teste a questão (gere uma prova com ela) antes de usar com alunos. O sistema também não avisa se o nome escolhido já existe no banco.

## O banco de questões

O arquivo `banco_questoes.json` é uma lista de objetos:

```json
{
  "id": "matematica_pa_decimo_termo",
  "disciplina": "Matematica",
  "tema": "Facil",
  "conteudo_rmd": "```{r, echo = FALSE, results = \"hide\"}\n...código R...\n```\n\nQuestion\n========\n..."
}
```

Hoje há **22 questões** (Matemática, Estatística, Física e Química), nos tipos `schoice` (múltipla escolha) e `string` (resposta digitada).

Para criar uma questão manualmente, escreva o `conteudo_rmd` no formato do R/Exams:

- **`schoice`**: a `Answerlist` da pergunta e a da solução precisam ter o mesmo número de itens. O `exsolution` é uma sequência de 0 e 1, uma posição por alternativa (`10000` = a primeira é a correta). Use `exshuffle: TRUE` para embaralhar.
- **`num`**: o `exsolution` é o valor correto e o `extol` é a tolerância aceita.
- Todo valor que muda entre versões deve ser sorteado com `sample()` e a resposta deve ser **calculada em R**.
- Fórmulas entre `$...$`. Cifrão de moeda escapado: `R\$ 10,00`.

Documentação oficial do formato: [r-exams.org](https://www.r-exams.org/).

> **Padronize os nomes:** disciplina e tema são comparados como texto exato. Hoje o banco tem `Estatistica` e `estatistica` (e `Matematica`), que aparecem como disciplinas separadas na lista. Use sempre a mesma grafia.

A pasta `exercises/` guarda os `.Rmd` de exemplo da versão antiga (uma questão por arquivo). **O app atual não lê essa pasta**, só o `banco_questoes.json`.

## Instalação

### Com Docker

Descrita no [Início rápido](#início-rápido). O `docker-compose.yml` mapeia a pasta do projeto para dentro do container (`/srv/shiny-server/rexams-web`), então o `banco_questoes.json` e as questões salvas ficam na sua máquina. A imagem (`rocker/shiny:4.4.1`) já traz LaTeX, Pandoc, `zip` e os pacotes R necessários.

### Sem Docker (R instalado)

| Requisito | Para quê |
|---|---|
| R (>= 4.2) | Linguagem base. [cran.r-project.org](https://cran.r-project.org) |
| Pacotes R | `shiny`, `bslib`, `exams`, `rmarkdown`, `knitr`, `httr2`, `pdftools`, `officer` (o `packages.R` instala tudo) |
| LaTeX | PDF **e também Importar com IA** (a validação compila um PDF de teste). O TinyTeX é o mais simples; o `packages.R` instala se não houver LaTeX |
| Pandoc | HTML e Moodle (e validação do Importar com IA). Normalmente já vem com o RStudio |
| `zip` no PATH | Botão de download. No Windows, garanta um `zip.exe` (Git Bash ou Rtools) no PATH |

```r
# 1. instalar dependências (uma vez só)
source("packages.R")

# 2. rodar o app
shiny::runApp()
```

**Chave da Groq (só para Importar com IA):** crie um arquivo `.Renviron` na raiz do projeto com

```
GROQ_API_KEY=gsk_sua-chave-aqui
```

e **reinicie o R**. A chave é gratuita em [console.groq.com](https://console.groq.com) (**API Keys** → **Create API Key**). Nunca envie esse arquivo ao Git (o `.gitignore` já o ignora).

### O que cada formato exige

| Formato | Depende de |
|---|---|
| PDF | LaTeX |
| HTML | Pandoc |
| Moodle (XML) | Pandoc |
| Download em `.zip` | `zip` |
| Importar com IA | Chave da Groq, LaTeX e Pandoc |

## Solução de problemas

| Sintoma | O que verificar |
|---|---|
| Erro ao gerar PDF | LaTeX instalado? No R, `Sys.which("pdflatex")`; se vier vazio, `tinytex::install_tinytex()`. A mensagem de erro traz um trecho do log do LaTeX. |
| Erro ao gerar HTML ou Moodle | Pandoc instalado? `rmarkdown::pandoc_available()` |
| O botão de download falha | `zip` no PATH (veja a tabela de requisitos) |
| "O número de questões pedido é maior do que o banco disponível…" | Marque mais temas ou reduza o número de questões |
| "GROQ_API_KEY não encontrada" | Crie o `.env` (Docker) ou o `.Renviron` (R local) e reinicie. No Docker, se a chave continuar sem ser vista pelo app, crie também um `.Renviron` na raiz do projeto com a mesma linha e rode `docker compose restart` |
| "A IA alterou o enunciado…" | A IA reescreveu o texto. Clique em converter de novo ou cole um texto mais curto |
| "A IA não parametrizou a questão…" | A IA não criou `sample()`. Tente de novo; se a questão realmente não tem números, o aviso pode se repetir |
| Erro na compilação durante Importar com IA | Confira LaTeX e Pandoc. O erro de compilação aparece na pré-visualização junto com o texto devolvido pela IA |
| Nova disciplina não aparece | A lista de disciplinas é carregada só ao iniciar o app. Reinicie (`docker compose restart`) |
| Atenção, sem sorteio | A questão não tem `sample()`: não varia entre versões. Parametrize-a no banco |

## Limitações conhecidas

- **Prova e gabarito são gerados separadamente.** Cada clique em **Gerar avaliação** sorteia questões e valores novos, e o `.zip` traz só a última geração. Uma prova gerada num clique e um gabarito gerado em outro **não correspondem entre si**. Para ter os dois da mesma geração, escolha **Com gabarito**, que produz o arquivo `prova_gabarito`.
- O cabeçalho da interface ainda exibe o nome antigo ("STATLAB").
- A lista de disciplinas só atualiza ao reiniciar o app.
- Não há pré-visualização renderizada da prova, histórico de provas nem login de usuários.
- Os arquivos gerados ficam em pastas temporárias.
- A aba "Banco de questões" ainda não permite consultar, editar ou excluir questões.

## Estrutura do projeto

```
AvalIA/
├── app.R                  # interface Shiny (3 abas) e lógica do servidor
├── R/
│   ├── helpers.R          # lê o banco, filtra questões, gera as provas (exams) e o .zip;
│   │                      # template LaTeX com caracteres Unicode (₂, μ, ≤ ...)
│   └── ia_helpers.R       # chamada à Groq, prompt de conversão, validações,
│                          # normalização de LaTeX e leitura de PDF/Word
├── banco_questoes.json    # BANCO DE QUESTÕES (usado pelo app)
├── exercises/             # .Rmd de exemplo da versão antiga (não lidos pelo app)
├── packages.R             # instala as dependências R (rodar uma vez)
├── docker-compose.yml     # sobe o app na porta 3838 e repassa a GROQ_API_KEY
├── deploy/
│   └── Dockerfile         # R + LaTeX + Pandoc + zip + Shiny Server
└── README.md
```

**Tecnologias:** R · Shiny + bslib · `exams` (R/Exams) · rmarkdown/knitr · LaTeX · Pandoc · httr2 (API da Groq) · pdftools · officer · Docker (`rocker/shiny`).

## Próximos passos

Ideias ainda não implementadas:

- Consulta, edição e exclusão de questões na aba "Banco de questões".
- Lista de disciplinas reativa (sem reiniciar o app).
- Prévia renderizada da prova antes de gerar.
- Prova e gabarito sempre da mesma geração.
- Cabeçalho/logo customizável (template LaTeX próprio da escola).
- Autenticação de professores e histórico de provas geradas.
- Exportação direta para o Moodle via API.