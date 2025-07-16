
### Tutorial para instalação do RespondeAI

Olá! Se você baixou um fluxo de Agente de IA para o n8n, o próximo passo é conectar suas contas para que tudo funcione. Este guia vai te ensinar a criar as 3 credenciais necessárias:

**O que vamos configurar:**

0.  **Importação do Fluxo:** O ponto de partida para usar seu novo agente.
1.  **Chave de API do Google Gemini:** O "cérebro" do agente.
2.  **Credenciais do Supabase:** A memória e a base de conhecimento do agente.
3.  **Credencial do Google (OAuth):** Para que o agente possa ler arquivos do seu Google Drive.


Vamos lá!

---

### Etapa 0: Importando o Fluxo de Trabalho para o n8n

Antes de tudo, você precisa carregar o arquivo `.json` do fluxo para a sua instância do n8n.

1.  Com o n8n aberto, clique no ícone **"+"** no canto superior direito da tela.
2.  No menu que aparecer, selecione a opção **"Import from file..."**.
3.  Selecione o arquivo `.json` que você baixou.
4.  Pronto! O fluxograma do Agente de IA deve aparecer na sua tela, pronto para ser configurado.



---

### 1. Criando a Chave de API do Google Gemini

Esta chave permite que o n8n utilize os modelos de linguagem do Google, como o Gemini, para processar as informações e conversar.

1.  Acesse o Google AI Studio em: [**https://aistudio.google.com/app/apikey**](https://aistudio.google.com/app/apikey)
2.  Faça login com sua conta Google.
3.  Clique no botão **"Create API key in new project"**.
4.  **Copie a chave de API gerada imediatamente!** Guarde-a em um local seguro, pois você vai precisar dela.
5.  **No n8n:**
    *   Vá até os nós `Google Gemini Chat Model` e `Embeddings Google Gemini`.
    *   No campo "Credential", clique em **"Create New Credential"**.
    *   Dê um nome para a credencial (ex: `Minha Chave Gemini`) e cole a chave que você copiou no campo **API Key**. Salve.
    *   Selecione essa mesma credencial para ambos os nós do Gemini.

---



### 2. Criando e Configurando o Supabase

O Supabase nos fornecerá duas coisas: um banco de dados **Postgres** para a memória do agente e uma **Vector Store** para armazenar e consultar seus documentos.

#### Parte A: Criando o Projeto no Supabase

1.  Acesse o site do Supabase: [**https://supabase.com/**](https://supabase.com/)
2.  Clique em **"Start your project"** e crie uma conta (pode usar sua conta do GitHub para facilitar).
3.  No seu dashboard, clique em **"New project"**.
4.  Dê um nome ao seu projeto (ex: `Agente IA N8N - Demo`).
5.  **CRIE E ANOTE A SENHA DO BANCO DE DADOS (Database Password)!** Esta senha é crucial e você vai precisar dela. Guarde-a em um local seguro.
6.  Escolha uma região próxima a você (ex: `South America (São Paulo)`).
7.  Clique em **"Create new project"** e aguarde alguns minutos enquanto o Supabase prepara tudo.

#### Parte B: Configurando a Memória (Postgres) no n8n

1.  No painel do seu projeto no Supabase, vá para **Project Settings** (ícone de engrenagem) > **Database**.
2.  Você verá os parâmetros de conexão.
3.  **No n8n:**
    *   Vá até o nó `Postgres Chat Memory`.
    *   No campo "Credential", clique em **"Create New Credential"**.
    *   Preencha os campos copiando os valores do Supabase:
        *   **Host:** Copie o valor do campo `Host` do Supabase.
        *   **Database:** `postgres` (geralmente é o padrão).
        *   **User:** `postgres` (geralmente é o padrão).
        *   **Port:** Copie o valor do campo `Port` (ex: `6543`).
        *   **Password:** Use a senha que você anotou na Parte A.
    *   Salve a credencial.



#### Parte C: Configurando a Vector Store (API do Supabase) no n8n

1.  No painel do seu projeto no Supabase, vá para **Project Settings** (ícone de engrenagem) > **API**.
2.  Nesta página, você precisará de duas informações:
    *   **Project URL:** Copie o valor do campo `URL`.
    *   **Project API Keys:** Encontre a chave `service_role` e clique em **Reveal** para vê-la. Copie o valor.
3.  **No n8n:**
    *   Vá até o nó `Supabase Vector Store`.
    *   No campo "Credential", clique em **"Create New Credential"**.
    *   Cole o **Project URL** no campo `Host`.
    *   Cole a chave **service_role** no campo `Service Role Secret`.
    *   Salve a credencial.

#### Parte D: Preparando a Tabela da Vector Store

Para que o Supabase armazene seus documentos, precisamos rodar um script que cria a tabela e as funções necessárias.

1.  No painel do Supabase, vá para o **SQL Editor**.
2.  Clique em **"+ New query"**.
3.  Copie **todo o código abaixo** e cole no editor de SQL:
    ```sql
    -- Enable the pgvector extension to work with embedding vectors
    create extension vector;
    
    -- Create a table to store your documents
    create table documents (
      id bigserial primary key,
      content text, -- corresponds to Document.pageContent
      metadata jsonb, -- corresponds to Document.metadata
      embedding vector(768) -- 1536 works for OpenAI embeddings, 768 for Gemini
    );
    
    -- Create a function to search for documents
    create function match_documents (
      query_embedding vector(768),
      match_count int DEFAULT null,
      filter jsonb DEFAULT '{}'
    ) returns table (
      id bigint,
      content text,
      metadata jsonb,
      similarity float
    )
    language plpgsql
    as $$
    #variable_conflict use_column
    begin
      return query
      select
        id,
        content,
        metadata,
        1 - (documents.embedding <=> query_embedding) as similarity
      from documents
      where metadata @> filter
      order by documents.embedding <=> query_embedding
      limit match_count;
    end;
    $$;
    ```
4.  Clique no botão verde **"RUN"**. Se aparecer "Success. No rows returned", deu tudo certo!

---

### 3. Criando a Credencial do Google (OAuth2)

Esta credencial é necessária para que o n8n possa acessar arquivos no seu Google Drive.

1.  Acesse o Google Cloud Console: [**https://console.cloud.google.com/**](https://console.cloud.google.com/)
2.  Crie um novo projeto ou selecione um existente.
3.  No menu de busca, procure por **"Google Drive API"** e clique em **"Ativar" (Enable)**.
4.  Após ativar, no menu lateral, vá para **APIs e Serviços > Credenciais**.
5.  Clique em **"+ CRIAR CREDENCIAIS"** e escolha **"ID do cliente OAuth"**.
6.  Se for a primeira vez, configure a "Tela de permissão OAuth".
    *   Selecione **"Externo"** e clique em "Criar".
    *   Preencha o nome do app (ex: `n8n Agente IA`), seu e-mail de suporte e o e-mail do desenvolvedor. Salve e continue.
    *   Pule as seções "Escopos" e "Usuários de teste" por enquanto.
7.  De volta à criação de credenciais:
    *   Em "Tipo de aplicativo", selecione **"Aplicativo da Web"**.
    *   Em **"URIs de redirecionamento autorizados"**, clique em **"+ ADICIONAR URI"**.
    *   **No n8n:** ao criar a credencial do Google, ele te dará um **"OAuth Redirect URL"**. Copie esse link.
    *   Cole o link do n8n no campo do Google Cloud Console.
8.  Clique em **"Criar"**. O Google mostrará seu **ID do cliente** e sua **Chave secreta do cliente**.
9.  **No n8n:**
    *   Volte para a tela de criação da credencial do Google.
    *   Copie e cole o **ID do cliente** e a **Chave secreta do cliente** nos campos correspondentes.
    *   Salve a credencial e autorize o acesso à sua conta Google quando solicitado.

---

**Pronto!** Com essas três credenciais configuradas, seu Agente de IA no n8n está pronto para ser ativado e testado. Bom trabalho
