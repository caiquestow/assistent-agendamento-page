# 🗓️ Assistente de Agendamento - Documentação Técnica

## 📋 Índice
1. [Visão Geral](#visão-geral)
2. [Arquitetura do Sistema](#arquitetura-do-sistema)
3. [Componentes Principais](#componentes-principais)
4. [Fluxos de Funcionamento](#fluxos-de-funcionamento)
5. [Desafios Enfrentados e Soluções](#desafios-enfrentados-e-soluções)
6. [Tecnologias Utilizadas](#tecnologias-utilizadas)
7. [Requisitos e Configuração](#requisitos-e-configuração)
8. [Demonstração](#demonstração)
9. [Melhorias Futuras](#melhorias-futuras)

## 📝 Visão Geral

O Assistente de Agendamento é uma aplicação que permite aos usuários criar eventos de calendário usando comandos em linguagem natural. O sistema interpreta comandos como "Agendar reunião com João amanhã às 10h", extrai as informações relevantes e automatiza todo o processo de agendamento, incluindo:

- Criação do evento no Google Calendar
- Envio de email de confirmação
- Registro do evento em uma planilha do Google Sheets
- Processamento de comandos por voz (feature bônus)

## 🏗️ Arquitetura do Sistema

O projeto segue uma arquitetura modular baseada em serviços, facilitando manutenção e escalabilidade:

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│                 │     │                  │     │                 │
│  Interface Web  │────▶│  API FastAPI     │────▶│  Interpretação  │
│  (Frontend)     │     │  (Backend)       │     │  (NLP/OpenAI)   │
│                 │     │                  │     │                 │
└─────────────────┘     └──────────────────┘     └────────┬────────┘
                                                          │
                                                          ▼
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│                 │     │                  │     │                 │
│  Notificação    │◀────│  Registro        │◀────│  Integração     │
│  (Email)        │     │  (Google Sheets) │     │  (G Calendar)   │
│                 │     │                  │     │                 │
└─────────────────┘     └──────────────────┘     └─────────────────┘
```

### Estrutura de Diretórios

```
app/
├── api/                # Camada de API
│   ├── controllers/    # Controladores
│   ├── models/         # Modelos de dados
│   └── routes/         # Rotas da API
├── core/               # Configurações centrais
├── services/           # Regras de negócio e integrações
├── static/             # Interface web (frontend)
└── utils/              # Utilitários diversos
```

## 🧩 Componentes Principais

### 1. Interface Web (Frontend)

Uma interface simples construída com HTML, CSS e JavaScript que permite:
- Autorização do Google Calendar
- Envio de comandos de texto
- Gravação de comandos de voz
- Visualização dos resultados

Exemplo de código da interface:

```html
<!-- Componente de entrada de comando -->
<div class="command-section">
  <h2>Comando de Texto</h2>
  <form id="text-command-form">
    <input type="text" id="command-input" placeholder="Ex: Agendar reunião com João amanhã às 10h" required>
    <button type="submit">Enviar</button>
  </form>
</div>
```

### 2. API (Backend)

Construída com FastAPI (Python), expondo endpoints para:
- Processamento de comandos de texto e voz
- Autenticação OAuth com Google
- Listagem de eventos

Trecho representativo da estrutura da API:

```python
@router.post("/command/text", response_model=CommandResponse)
async def process_text_command(request: CommandRequest):
    """
    Processa um comando de texto para agendamento
    """
    return await command_controller.process_text_command(request)

@router.post("/command/voice", response_model=CommandResponse)
async def process_voice_command(request: VoiceCommandRequest):
    """
    Processa um comando de voz para agendamento
    """
    return await command_controller.process_voice_command(request)
```

### 3. Serviço de Interpretação

O componente que extrai informações estruturadas dos comandos em linguagem natural:
- Utiliza OpenAI API com "function calling" para extrair dados precisos
- Possui fallback com processamento tradicional (regex) para redundância

Trecho central do interpretador:

```python
async def parse_command(self, command: str) -> Dict[str, Any]:
    """
    Analisa o comando de texto usando function calling da OpenAI e extrai informações relevantes
    """
    try:
        # Obter data e hora atuais para contexto
        now = datetime.datetime.now()
        date_str = now.strftime("%Y-%m-%d")
        day_of_week = now.strftime("%A")
        time_str = now.strftime("%H:%M")
        
        # Criar prompt de sistema com contexto temporal atual
        system_prompt = f"""
        Você é um assistente especializado em extrair informações precisas de comandos de agendamento.
        
        A data e hora atuais são:
        - Data: {date_str} ({day_of_week})
        - Hora: {time_str}
        
        IMPORTANTE:
        1. Ao identificar o nome da pessoa, extraia APENAS o nome - não inclua palavras como 'amanhã', 'hoje' etc.
        2. Ao extrair a data, use a data atual como referência para interpretar expressões como 'amanhã'.
        3. O formato da data deve ser YYYY-MM-DDTHH:MM:SS.
        """
        
        # Chamar API da OpenAI
        from openai import AsyncOpenAI
        client = AsyncOpenAI(api_key=settings.OPENAI_API_KEY)
        
        response = await client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": command}
            ],
            tools=[{
                "type": "function",
                "function": self.functions[0]
            }],
            tool_choice={"type": "function", "function": {"name": "extract_event_details"}}
        )

        # Processar resposta...
```

### 4. Serviço de Google Calendar

Gerencia a autenticação OAuth e a criação de eventos:
- Implementa fluxo completo OAuth 2.0
- Cria eventos com detalhes extraídos do comando
- Gerencia tokens e refrescamento automático

Trecho de código relevante:

```python
def create_event(self, event_data):
    """
    Cria um evento no Google Calendar
    """
    # Parse da data ISO
    start_time = datetime.datetime.fromisoformat(event_data["data"])
    end_time = start_time + datetime.timedelta(hours=1)  # Evento de 1 hora por padrão

    # Detalhes adicionais do evento
    description = event_data.get("detalhes", "")

    # Criação do evento
    event = {
        'summary': f"{event_data['tipo'].title()} com {event_data['nome']}",
        'description': description,
        'start': {
            'dateTime': start_time.isoformat(),
            'timeZone': 'America/Sao_Paulo',
        },
        'end': {
            'dateTime': end_time.isoformat(),
            'timeZone': 'America/Sao_Paulo',
        },
        'attendees': [
            {'email': f"{event_data['nome'].lower().replace(' ', '.')}@example.com"},
        ],
        'reminders': {
            'useDefault': False,
            'overrides': [
                {'method': 'email', 'minutes': 24 * 60},
                {'method': 'popup', 'minutes': 30},
            ],
        },
    }

    created_event = self.service.events().insert(
        calendarId='primary',
        body=event,
        sendUpdates='none'
    ).execute()

    return {
        'id': created_event['id'],
        'link': created_event.get('htmlLink', ''),
        'summary': created_event.get('summary', '')
    }
```

### 5. Serviço de Email

Responsável pelo envio de confirmações por email:
- Suporta formatação HTML
- Garante emails válidos através de normalização
- Implementa tratamento de erros e fallbacks

### 6. Serviço de Google Sheets

Realiza o registro de todos os eventos agendados:
- Cria planilha automaticamente caso não exista
- Adiciona cabeçalhos e formata dados
- Mantém histórico de todas as ações realizadas

### 7. Serviço de Voz (Feature Bônus)

Implementa processamento de comandos por voz:
- Transcrição de áudio para texto (OpenAI Whisper API)
- Geração de respostas em áudio (OpenAI TTS API)
- Gerenciamento de arquivos temporários

## 🔄 Fluxos de Funcionamento

### Fluxo Principal (Comando de Texto)

1. Usuário insere comando de texto na interface
2. Frontend envia comando para a API via POST
3. API passa comando para o serviço de interpretação
4. Serviço de interpretação extrai nome, data/hora e tipo de evento
5. Serviço do Google Calendar cria o evento
6. Serviço de email envia confirmação
7. Serviço do Google Sheets registra o evento
8. API retorna detalhes do agendamento para o frontend
9. Interface exibe confirmação e detalhes ao usuário

### Fluxo Alternativo (Comando de Voz)

1. Usuário clica no botão "Gravar" e diz o comando
2. Frontend codifica o áudio em base64 e envia para API
3. Serviço de voz transcreve o áudio em texto
4. O sistema segue o fluxo principal (passos 3-8)
5. Adicionalmente, gera uma resposta em áudio
6. Frontend reproduz a resposta em áudio para o usuário

## 🔍 Desafios Enfrentados e Soluções

### 1. Interpretação Precisa de Comandos

**Desafio:** Extrair corretamente nomes e expressões temporais como "amanhã" ou "próxima semana".

**Solução:** Implementei uma abordagem em duas camadas:

1. **Uso de Function Calling da OpenAI API:**
   - Fornecendo instruções claras e contexto temporal
   - Especificando exatamente quais campos extrair e em que formato

   ```python
   # Definição das funções para OpenAI
   self.functions = [
       {
           "name": "extract_event_details",
           "description": "Extrai os detalhes de um evento de uma solicitação de agendamento",
           "parameters": {
               "type": "object",
               "properties": {
                   "nome": {
                       "type": "string",
                       "description": "APENAS o nome da pessoa, sem incluir palavras temporais como 'amanhã'"
                   },
                   "data": {
                       "type": "string",
                       "description": "Data e hora do evento no formato ISO 8601 (YYYY-MM-DDTHH:MM:SS)"
                   },
                   # outros campos...
               },
               "required": ["nome", "data", "tipo"]
           }
       }
   ]
   ```

2. **Sistema de Fallback Robusto:**
   - Implementei processamento tradicional com regex para casos de falha da API
   - Validação adicional para garantir que dados essenciais são sempre extraídos

### 2. Autenticação OAuth do Google

**Desafio:** Implementar o fluxo completo de autenticação OAuth 2.0 com redirecionamento correto.

**Solução:**
- Criei endpoints específicos para manipular o fluxo de autorização
- Implementei armazenamento seguro de tokens
- Adicionei verificações de status e depuração para o processo de autenticação

### 3. Processamento de Voz

**Desafio:** Obter uma transcrição precisa de comandos de voz e gerar respostas em áudio natural.

**Solução:**
- Uso da API Whisper da OpenAI para transcrição de alta precisão
- Implementação de Text-to-Speech com vozes naturais

## 🛠️ Tecnologias Utilizadas

- **Backend:** FastAPI (Python 3.9+)
- **Frontend:** HTML5, CSS3, JavaScript
- **APIs e Integrações:**
  - OpenAI API (GPT 4o Mini [default] para interpretação, Whisper para STT, TTS para resposta de voz)
  - Google API (Calendar, Sheets)
  - SMTP para envio de emails
- **Bibliotecas Python Principais:**
  - `google-auth` e `google-api-python-client` para integrações Google
  - `pydantic` para validação de esquemas
  - `python-dotenv` para gerenciamento de variáveis de ambiente
  - `gspread` para interação com Google Sheets
  - `dateutil` para processamento avançado de datas

## ⚙️ Requisitos e Configuração

### Variáveis de Ambiente

O sistema requer as seguintes configurações no arquivo `.env`:

```
OPENAI_API_KEY=sua_chave_api_openai
GOOGLE_CLIENT_ID=seu_client_id
GOOGLE_CLIENT_SECRET=seu_client_secret
GOOGLE_REDIRECT_URI=http://localhost:8000/api/auth/callback
EMAIL_USER=seu_email@gmail.com
EMAIL_PASS=sua_senha_de_app
OPENAI_MODEL=modelo_gpt
```

### Configuração do Projeto Google Cloud

Para o funcionamento correto das integrações Google, é necessário:

1. Criar um projeto no Google Cloud Console
2. Habilitar as seguintes APIs:
   - Google Calendar API
   - Google Sheets API
   - Google Drive API
3. Configurar auth

## 🚀 Melhorias Futuras

   - Implementação de refresh tokens mais seguros em um DB.
   - Utilização da meta pra msgs via whatsapp.
   - Migração para uso com LangChain para fluxos mais complexos e outras LLMs com mais agilidade.
   - Utilização de contextos com base em configurações prévias, como emails de equipes/usuarios ou preferências, e criação de mecanismo conversacional.
   - Histórico de Comandos: Implementar um registro histórico dos comandos processados.
   - Utilizaçao de docker para continuar o desenvolvimento em ambientes cloud e produtivos.
   - Criação de testes e monitoramento.

   - Suporte a Cancelamento/Edição: Adicionar suporte para cancelar ou editar eventos já criados.
