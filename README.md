🏗️ Arquitetura e Lógica do Workflow no n8n

O fluxo foi desenhado para garantir rastreabilidade total (registro de 100% dos chamados) aliada a um atendimento ágil para incidentes críticos (alertas em tempo real apenas para alta prioridade).
Plaintext

[n8n Form Trigger] ──> [Basic LLM Chain + Groq] ──> [Google Sheets] ──> [If (Urgência == 'Alta')]
                                                                                   │
                                                                           ┌───────┴───────┐
                                                                         [TRUE]          [FALSE]
                                                                           │               │
                                                                    [Gmail Node]     (Fim da execução)

📦 Nós do n8n Utilizados e Suas Configurações
1. n8n Form Trigger (Entrada do Chamado)

    Função: Capturar as informações enviadas pelo usuário via formulário nativo do n8n.

    Campos do Formulário:

        Nome (Text)

        E-mail (Text)

        Mensagem / Problema (Text Area / Multiple Lines)

    Saída esperada: Objeto JSON contendo os dados inseridos no formulário.

2. Basic LLM Chain + Groq Chat Model (Processamento por IA)

    Função: Processar o texto do chamado utilizando o modelo Llama 3 via Groq e extrair dados em formato JSON estruturado.

    Sub-nó de Modelo: Groq Chat Model (utilizando o modelo llama3-70b-8192 ou llama3-8b-8192).

    Prompt do Sistema (System Message):
    Plaintext

    Você é um assistente especializado em triagem de chamados de TI.
    Analise a mensagem do usuário e retorne EXATAMENTE um JSON válido no seguinte formato, sem formatação Markdown e sem texto adicional:

    {
      "categoria": "Infraestrutura | Software | Acesso | Hardware | Outros",
      "urgencia": "Alta | Média | Baixa",
      "resumo": "Breve resumo do problema em uma frase"
    }

    Prompt do Usuário (User Message):
    Plaintext

    Mensagem do cliente: {{ $json['Mensagem / Problema'] }}

3. Google Sheets (Registro Geral)

    Função: Gravar todos os chamados na planilha centralizadora do Google Sheets.

    Operação: Append Row (ou Append or Update).

    Documento/Aba: Documento Chamados, Aba Centralizador.

    Mapeamento de Campos:

        Data/Hora ➔ {{ $now.format('yyyy-MM-dd HH:mm:ss') }}

        Nome ➔ {{ $('n8n Form Trigger').item.json.Nome }}

        E-mail ➔ {{ $('n8n Form Trigger').item.json['E-mail'] }}

        Mensagem Original ➔ {{ $('n8n Form Trigger').item.json['Mensagem / Problema'] }}

        Categoria ➔ {{ JSON.parse($('Basic LLM Chain').item.json.text).categoria }}

        Urgência ➔ {{ JSON.parse($('Basic LLM Chain').item.json.text).urgencia }}

        Resumo ➔ {{ JSON.parse($('Basic LLM Chain').item.json.text).resumo }}

4. If (Triagem Condicional)

    Função: Avaliar o nível de urgência definido pela IA e direcionar a execução.

    Condição:

        Value 1: {{ JSON.parse($('Basic LLM Chain').item.json.text).urgencia }}

        Operator: Equal (String)

        Value 2: Alta

    Comportamento das Rotas:

        Rota TRUE: Executada se a urgência for estritamente igual a "Alta".

        Rota FALSE: Executada para urgências "Média" e "Baixa" (o fluxo encerra aqui, pois o registro na planilha já foi efetuado no nó anterior).

5. Gmail (Notificação de Alerta)

    Função: Disparar e-mail de alerta para a equipe de TI apenas para chamados críticos.

    Autenticação: Credencial OAuth2 configurada no Google Cloud Console com as URI de redirecionamento do n8n e o usuário de teste ativo.

    Resource / Operation: Message / Send

    Configurações dos Campos:

        To: E-mail da equipe de suporte (ex: suporte@empresa.com).

        Subject (Assunto): 🚨 ALERTA CRÍTICO: Chamado Urgente - {{ $('n8n Form Trigger').item.json.Nome }}

        Email Type: Text

        Message Body:
        Plaintext

        Atenção Equipe de TI,

        Um novo chamado de ALTA URGÊNCIA foi registrado.

        • Solicitante: {{ $('n8n Form Trigger').item.json.Nome }}
        • E-mail: {{ $('n8n Form Trigger').item.json['E-mail'] }}
        • Categoria: {{ JSON.parse($('Basic LLM Chain').item.json.text).categoria }}
        • Urgência: ALTA 🚨
        • Resumo: {{ JSON.parse($('Basic LLM Chain').item.json.text).resumo }}

        • Mensagem Original:
        "{{ $('n8n Form Trigger').item.json['Mensagem / Problema'] }}"

        --------------------------------------------------
        Por favor, verifiquem a fila de atendimento imediatamente.

⚙️ Regras e Requisitos Técnicos

    Garantia de Registro Universal: O nó do Google Sheets é posicionado antes do nó If. Isso garante o cumprimento da Regra 1: todos os chamados ficam gravados no banco de dados, independentemente da urgência.

    Filtragem Estrita: Apenas os itens que passam na validação do nó If (com valor exato Alta) seguem para o nó do Gmail.

    OAuth2 do Gmail: Para evitar erros de permissão (403 Access Not Configured ou Token Expired), a aplicação no Google Cloud deve estar com o status de publicação correto em OAuth consent screen e a conta de envio listada em Test users.
