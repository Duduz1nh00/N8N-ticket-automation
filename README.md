# ⚡ Triagem Inteligente de Chamados com n8n e IA (Groq / Llama 3)

> Projeto desenvolvido para o Desafio Criativo: **"Planejando Automações com N8N Usando Apenas Bons Prompts"** da [DIO (Digital Innovation One)](https://dio.me).

---

## 📌 Visão Geral do Projeto

Este projeto consiste em uma automação desenvolvida no **n8n** para triagem, categorização e priorização automática de chamados de suporte técnico de TI. 

Utilizando **Engenharia de Prompt** e um modelo de linguagem ultrarrápido (**Llama 3 via Groq**), o fluxo analisa mensagens enviadas via formulário, extrai dados estruturados em JSON, grava 100% dos registros no **Google Sheets** para auditoria e dispara alertas imediatos via **Gmail** exclusivamente para chamados de **Alta criticidade**.

---

## 🏗️ Arquitetura do Workflow

```mermaid
flowchart LR
    A[📋 n8n Form Trigger<br>Entrada de Dados] --> B[🧠 Basic LLM Chain<br>Llama 3 via Groq]
    B --> C[(📊 Google Sheets<br>Registro Geral 100%)]
    C --> D{Urgência == 'Alta'?}
    D -- Sim [TRUE] --> E[🚨 Gmail<br>Alerta Crítico Imediato]
    D -- Não [FALSE] --> F[🏁 Fim da Execução]


📝 Planejamento e Engenharia de Prompt (Passo a Passo)
A construção desta automação seguiu a metodologia em 3 etapas proposta no desafio:s

🧱 Passo 1: Definição da Automação Desejada
Objetivo: Triar e notificar chamados críticos para a equipe de tecnologia.
Público Impactado: Equipe de tecnologia e suporte responsável pela gestão e resolução de incidentes.
Resultado Esperado: Receber as solicitações dos clientes via formulário, categorizar o problema com IA, armazenar todos os chamados no Google Sheets e notificar a equipe técnica por e-mail em incidentes de alta urgência.
🧱 Passo 2: Contexto, Ferramentas e Regras
Ferramentas Envolvidas: n8n, Groq AI (llama3-70b-8192 / Basic LLM Chain), Google Sheets e Gmail (OAuth2).
Sequência do Fluxo:
Entrada do Chamado: Usuário preenche nome, e-mail e descrição do problema.
Processamento por IA & Registro: A IA analisa o relato, extrai JSON estruturado e o n8n faz append na planilha Centralizador.
Triagem Condicional & Alerta: Avaliação da urgência com nó If. Chamados com urgência "Alta" geram e-mail imediato; outros encerram normalmente.
Regras Importantes:
Garantia de registro universal: 100% dos chamados são salvos no Google Sheets antes de qualquer filtro.
Alerta estrito: apenas urgência exatamente igual a "Alta" aciona o nó de e-mail.
Segurança: autenticação via OAuth2 devidamente configurada no Google Cloud.


🧱 Passo 3: O Prompt Final Consolidado
Atue como um especialista em N8N.

Crie uma automação para triagem e notificação automática de chamados críticos para a equipe de tecnologia.

Público:
Equipe de tecnologia e suporte responsável pela gestão e resolução de chamados.

Ferramentas envolvidas:
- n8n (Form Trigger, nós de controle e lógica)
- Groq AI (Llama 3 / Basic LLM Chain)
- Google Sheets
- Gmail (OAuth2)

Fluxo:
1. Entrada do Chamado: O usuário envia as informações de suporte através do formulário do n8n (Nome, E-mail, Mensagem).
2. Processamento por IA & Registro: O nó do Groq analisa o texto da mensagem e devolve os dados em formato JSON estruturado (urgência, categoria e resumo). O n8n faz o parsing e insere uma nova linha com todos os dados na planilha do Google Sheets (Chamados > Centralizador).
3. Triagem & Alerta: O nó condicional (If) analisa o campo de urgência. Se a urgência for igual a "Alta", dispara automaticamente um e-mail de alerta formatado via Gmail para a equipe responsável.

Regras:
1. O registro no Google Sheets ocorre para todos os chamados, independentemente do nível de urgência.
2. Apenas chamados classificados com urgência estritamente igual a "Alta" acionam a rota TRUE do nó If e enviam a notificação por Gmail.
3. As credenciais do Gmail devem utilizar a porta de callback OAuth2 registrada no Google Cloud Console com acesso de usuário de teste ativo.

Explique quais nós do N8N devem ser utilizados e a lógica de funcionamento do workflow.


📦 Especificação Técnica dos Nós no n8n
1. n8n Form Trigger (Gatilho de Entrada)
Tipo de Nó: n8n-nodes-base.formTrigger
Finalidade: Fornecer uma interface web amigável para captura do chamado.
Campos configurados:
Nome (Text)
E-mail (Email)
Mensagem / Problema (Text Area / Multiline)
2. Basic LLM Chain + Groq Chat Model (Inteligência Artificial)
Tipo de Nó: @n8n/n8n-nodes-langchain.chainLlm
Modelo: Groq Chat Model (llama3-70b-8192 ou llama3-8b-8192)
System Prompt:
Você é um assistente especializado em triagem de chamados de TI.
Analise a mensagem do usuário e retorne EXATAMENTE um JSON válido no seguinte formato, sem formatação Markdown e sem texto adicional:

{
  "categoria": "Infraestrutura | Software | Acesso | Hardware | Outros",
  "urgencia": "Alta | Média | Baixa",
  "resumo": "Breve resumo do problema em uma frase"
}


User prompt
Mensagem do cliente: {{ $json['Mensagem / Problema'] }}


Exemplo de Retorno do LLM:
{
  "urgencia": "Alta",
  "categoria": "Infraestrutura",
  "resumo": "Servidor de produção inacessível e banco de dados fora do ar."
}

3. Google Sheets (Persistência e Histórico)
Operação: Append Row
Documento / Planilha: Chamados / Aba Centralizador
Mapeamento de Expressões:
Coluna Planilha	Expressão n8n
Data/Hora	{{ $now.format('yyyy-MM-dd HH:mm:ss') }}
Nome	{{ $('n8n Form Trigger').item.json.Nome }}
E-mail	{{ $('n8n Form Trigger').item.json['E-mail'] }}
Mensagem Original	{{ $('n8n Form Trigger').item.json['Mensagem / Problema'] }}
Categoria	{{ JSON.parse($('Basic LLM Chain').item.json.text).categoria }}
Urgência	{{ JSON.parse($('Basic LLM Chain').item.json.text).urgencia }}
Resumo	{{ JSON.parse($('Basic LLM Chain').item.json.text).resumo }}

4. If (Lógica Condicional e Roteamento)
Tipo de Nó: n8n-nodes-base.if
Condição:
Valor 1: {{ JSON.parse($('Basic LLM Chain').item.json.text).urgencia }}
Operador: Equal (String)
Valor 2: Alta
Comportamento:
Rota TRUE: Encaminha para o envio de e-mail de alerta.
Rota FALSE: Finaliza a execução sem disparar e-mail (fluxo não-crítico concluído).
5. Gmail (Notificação Crítica)
Tipo de Nó: n8n-nodes-base.gmail
Operação: Message / Send
Autenticação: Google OAuth2 API
Destinatário (To): suporte@empresa.com
Assunto (Subject): 🚨 ALERTA CRÍTICO: Chamado Urgente - {{ $('n8n Form Trigger').item.json.Nome }}
Corpo do E-mail:
text


Atenção Equipe de TI,
Um novo chamado de ALTA URGÊNCIA foi registrado na plataforma.
• Solicitante: {{ $('n8n Form Trigger').item.json.Nome }}
• E-mail: {{ $('n8n Form Trigger').item.json['E-mail'] }}
• Categoria: {{ JSON.parse($('Basic LLM Chain').item.json.text).categoria }}
• Urgência: ALTA 🚨
• Resumo: {{ JSON.parse($('Basic LLM Chain').item.json.text).resumo }}
• Mensagem Original do Usuário:
"{{ $('n8n Form Trigger').item.json['Mensagem / Problema'] }}"
--------------------------------------------------
Acesse a planilha centralizadora para assumir o atendimento imediatamente.
⚙️ Boas Práticas e Requisitos Técnicos
Idempotência e Segurança: A gravação na planilha ocorre antes da tomada de decisão condicional. Caso o serviço de e-mail falhe, o chamado já está salvo de forma segura.
Parsing do JSON: O retorno do LLM é convertido usando JSON.parse(), garantindo que os nós subsequentes consigam ler os campos (categoria, urgencia, resumo) individualmente.
Escalabilidade: O modelo Groq garante tempos de resposta inferiores a 1 segundo para a análise do chamado, otimizando o consumo de recursos.


