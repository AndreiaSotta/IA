{
  "name": "Bilingual AI Chatbot System - Final",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "cronExpression",
              "expression": "0 2 * * *"
            }
          ]
        }
      },
      "id": "daily-backup-trigger",
      "name": "Daily Backup Trigger",
      "type": "n8n-nodes-base.cron",
      "typeVersion": 1,
      "position": [200, 300]
    },
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "/chat",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "webhook-chat",
      "name": "Webhook Chat",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 1,
      "position": [200, 600],
      "webhookId": "chat-webhook"
    },
    {
      "parameters": {
        "jsCode": "// Language Detection with GDPR Compliance\nconst crypto = require('crypto');\n\n// Get input data\nconst userMessage = $input.first().json.body.message || '';\nconst userAgent = $input.first().json.headers['user-agent'] || '';\nconst clientIP = $input.first().json.headers['x-forwarded-for'] || $input.first().json.headers['x-real-ip'] || 'unknown';\nconst sessionId = $input.first().json.body.sessionId;\n\n// Hash sensitive data for GDPR compliance\nconst hashedUserAgent = crypto.createHash('sha256').update(userAgent).digest('hex');\nconst hashedIP = crypto.createHash('sha256').update(clientIP).digest('hex');\n\n// Language detection patterns\nconst portuguesePatterns = [\n  /\\b(olá|oi|bom dia|boa tarde|boa noite|obrigad[oa]|por favor|desculp[ae]|sim|não|como está|tudo bem)\\b/i,\n  /\\b(quero|preciso|gostaria|poderia|consegue|ajuda|informação|serviço)\\b/i\n];\n\nconst englishPatterns = [\n  /\\b(hello|hi|good morning|good afternoon|good evening|thank you|thanks|please|sorry|yes|no|how are you)\\b/i,\n  /\\b(want|need|would like|could|can you|help|information|service)\\b/i\n];\n\n// Detect language\nlet detectedLanguage = 'en'; // default\nlet confidence = 0;\n\n// Check Portuguese patterns\nlet ptMatches = 0;\nportuguesePatterns.forEach(pattern => {\n  if (pattern.test(userMessage)) ptMatches++;\n});\n\n// Check English patterns\nlet enMatches = 0;\nenglishPatterns.forEach(pattern => {\n  if (pattern.test(userMessage)) enMatches++;\n});\n\nif (ptMatches > enMatches) {\n  detectedLanguage = 'pt';\n  confidence = ptMatches / portuguesePatterns.length;\n} else if (enMatches > 0) {\n  detectedLanguage = 'en';\n  confidence = enMatches / englishPatterns.length;\n}\n\nreturn {\n  json: {\n    userMessage,\n    sessionId,\n    detectedLanguage,\n    confidence,\n    hashedUserAgent,\n    hashedIP,\n    timestamp: new Date().toISOString()\n  }\n};"
      },
      "id": "language-detection",
      "name": "Language Detection",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [400, 600]
    },
    {
      "parameters": {
        "url": "{{ $env.SUPABASE_URL }}/rest/v1/conversations",
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            {
              "name": "session_id",
              "value": "eq.{{ $json.sessionId }}"
            },
            {
              "name": "order",
              "value": "created_at.desc"
            },
            {
              "name": "limit",
              "value": "10"
            }
          ]
        },
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "Bearer {{ $env.SUPABASE_SERVICE_KEY }}"
            },
            {
              "name": "apikey",
              "value": "{{ $env.SUPABASE_SERVICE_KEY }}"
            }
          ]
        }
      },
      "id": "get-conversation-history",
      "name": "Get Conversation History",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [600, 600]
    },
    {
      "parameters": {
        "url": "{{ $env.SUPABASE_URL }}/rest/v1/company_knowledge",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "Bearer {{ $env.SUPABASE_SERVICE_KEY }}"
            },
            {
              "name": "apikey",
              "value": "{{ $env.SUPABASE_SERVICE_KEY }}"
            }
          ]
        }
      },
      "id": "get-company-knowledge",
      "name": "Get Company Knowledge",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [800, 600]
    },
    {
      "parameters": {
        "jsCode": "// Prompt Builder\nconst language = $('Language Detection').first().json.detectedLanguage;\nconst userMessage = $('Language Detection').first().json.userMessage;\nconst conversationHistory = $('Get Conversation History').all();\nconst companyKnowledge = $('Get Company Knowledge').all();\n\n// System prompts by language\nconst systemPrompts = {\n  pt: `Você é um assistente virtual bilíngue da AIQUIMIA, especializado em fornecer informações sobre os nossos serviços de IA e automação. \n\nComportamento:\n- Responda sempre em português (PT-PT) quando o utilizador falar português\n- Seja profissional, prestável e amigável\n- Use informações da base de conhecimento da empresa quando relevante\n- Se não souber algo, seja honesto e ofereça ajuda alternativa\n- Mantenha o contexto da conversa anterior`,\n  \n  en: `You are a bilingual virtual assistant for AIQUIMIA, specialized in providing information about our AI and automation services.\n\nBehavior:\n- Always respond in English when the user speaks English\n- Be professional, helpful, and friendly\n- Use company knowledge base information when relevant\n- If you don't know something, be honest and offer alternative help\n- Maintain context from previous conversation`\n};\n\n// Build context from company knowledge\nlet companyContext = '';\nif (companyKnowledge.length > 0) {\n  companyContext = companyKnowledge.map(item => item.json.content || item.json.title).join('\\n');\n}\n\n// Build conversation context\nlet conversationContext = '';\nif (conversationHistory.length > 0) {\n  conversationContext = conversationHistory\n    .slice(0, 5) // Last 5 messages\n    .reverse()\n    .map(msg => `${msg.json.role}: ${msg.json.message}`)\n    .join('\\n');\n}\n\n// Build final prompt\nconst systemPrompt = systemPrompts[language] || systemPrompts.en;\nconst finalPrompt = `${systemPrompt}\n\n## Company Information:\n${companyContext}\n\n## Recent Conversation:\n${conversationContext}\n\n## Current User Message:\n${userMessage}\n\nPlease respond appropriately in ${language === 'pt' ? 'Portuguese (PT-PT)' : 'English'}:`;\n\nreturn {\n  json: {\n    prompt: finalPrompt,\n    language,\n    userMessage,\n    sessionId: $('Language Detection').first().json.sessionId\n  }\n};"
      },
      "id": "prompt-builder",
      "name": "Prompt Builder",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1000, 600]
    },
    {
      "parameters": {
        "url": "{{ $env.LLAMA_URL }}/api/generate",
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "model",
              "value": "{{ $env.LLAMA_MODEL }}"
            },
            {
              "name": "prompt",
              "value": "{{ $json.prompt }}"
            },
            {
              "name": "stream",
              "value": false
            },
            {
              "name": "options",
              "value": "{\n  \"temperature\": 0.7,\n  \"max_tokens\": 500,\n  \"top_p\": 0.9\n}"
            }
          ]
        },
        "options": {
          "timeout": 30000
        }
      },
      "id": "llama-ai-call",
      "name": "Llama3.2 AI API Call",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [1200, 600],
      "onError": "continueErrorOutput"
    },
    {
      "parameters": {
        "conditions": {
          "boolean": [
            {
              "value1": "={{ $json.error }}",
              "operation": "exists"
            }
          ]
        }
      },
      "id": "check-ai-error",
      "name": "Check AI Error",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [1400, 600]
    },
    {
      "parameters": {
        "jsCode": "// Error Handler - Fallback Response\nconst language = $('Prompt Builder').first().json.language;\n\nconst fallbackMessages = {\n  pt: 'Peço desculpa, mas estou com dificuldades técnicas no momento. Por favor, tente novamente em alguns minutos ou contacte-nos diretamente.',\n  en: 'I apologize, but I\\'m experiencing technical difficulties at the moment. Please try again in a few minutes or contact us directly.'\n};\n\nconst errorDetails = {\n  sessionId: $('Prompt Builder').first().json.sessionId,\n  timestamp: new Date().toISOString(),\n  error: $input.first().json.error || 'Unknown AI API error',\n  context: 'AI API call failed'\n};\n\nreturn {\n  json: {\n    response: fallbackMessages[language] || fallbackMessages.en,\n    language,\n    sessionId: errorDetails.sessionId,\n    isError: true,\n    errorDetails\n  }\n};"
      },
      "id": "error-handler",
      "name": "Error Handler",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1600, 750]
    },
    {
      "parameters": {
        "fromEmail": "{{ $env.SMTP_USER }}",
        "toEmail": "admin@aiquimia.com",
        "subject": "Critical Error in AI Chatbot",
        "emailType": "text",
        "message": "Timestamp: {{ $json.errorDetails.timestamp }}\nSession: {{ $json.errorDetails.sessionId }}\nError: {{ $json.errorDetails.error }}\n\nContext:\n{{ $json.errorDetails.context }}",
        "credentials": {
          "smtp": {
            "id": "smtp-credentials",
            "name": "SMTP Credentials"
          }
        }
      },
      "id": "email-admin-error",
      "name": "Email Admin on Error",
      "type": "n8n-nodes-base.emailSend",
      "typeVersion": 2,
      "position": [1800, 750]
    },
    {
      "parameters": {
        "jsCode": "// Process AI Response\nconst aiResponse = $input.first().json.response || '';\nconst language = $('Prompt Builder').first().json.language;\nconst sessionId = $('Prompt Builder').first().json.sessionId;\nconst userMessage = $('Prompt Builder').first().json.userMessage;\n\nreturn {\n  json: {\n    response: aiResponse,\n    language,\n    sessionId,\n    userMessage,\n    timestamp: new Date().toISOString(),\n    isError: false\n  }\n};"
      },
      "id": "process-ai-response",
      "name": "Process AI Response",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1600, 500]
    },
    {
      "parameters": {
        "url": "{{ $env.SUPABASE_URL }}/rest/v1/conversations",
        "sendBody": true,
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "Bearer {{ $env.SUPABASE_SERVICE_KEY }}"
            },
            {
              "name": "apikey",
              "value": "{{ $env.SUPABASE_SERVICE_KEY }}"
            },
            {
              "name": "Content-Type",
              "value": "application/json"
            }
          ]
        },
        "bodyParameters": {
          "parameters": [
            {
              "name": "session_id",
              "value": "{{ $json.sessionId }}"
            },
            {
              "name": "role",
              "value": "user"
            },
            {
              "name": "message",
              "value": "{{ $json.userMessage }}"
            },
            {
              "name": "language",
              "value": "{{ $json.language }}"
            },
            {
              "name": "created_at",
              "value": "{{ $json.timestamp }}"
            }
          ]
        }
      },
      "id": "save-user-message",
      "name": "Save User Message",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [1800, 500]
    },
    {
      "parameters": {
        "url": "{{ $env.SUPABASE_URL }}/rest/v1/conversations",
        "sendBody": true,
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "Bearer {{ $env.SUPABASE_SERVICE_KEY }}"
            },
            {
              "name": "apikey",
              "value": "{{ $env.SUPABASE_SERVICE_KEY }}"
            },
            {
              "name": "Content-Type",
              "value": "application/json"
            }
          ]
        },
        "bodyParameters": {
          "parameters": [
            {
              "name": "session_id",
              "value": "{{ $json.sessionId }}"
            },
            {
              "name": "role",
              "value": "assistant"
            },
            {
              "name": "message",
              "value": "{{ $json.response }}"
            },
            {
              "name": "language",
              "value": "{{ $json.language }}"
            },
            {
              "name": "created_at",
              "value": "{{ $json.timestamp }}"
            }
          ]
        }
      },
      "id": "save-ai-response",
      "name": "Save AI Response",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [2000, 500]
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "{\n  \"response\": \"{{ $json.response }}\",\n  \"language\": \"{{ $json.language }}\",\n  \"sessionId\": \"{{ $json.sessionId }}\",\n  \"timestamp\": \"{{ $json.timestamp }}\"\n}"
      },
      "id": "webhook-response",
      "name": "Webhook Response",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [2200, 500]
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "{\n  \"response\": \"{{ $json.response }}\",\n  \"language\": \"{{ $json.language }}\",\n  \"sessionId\": \"{{ $json.sessionId }}\",\n  \"timestamp\": \"{{ $json.timestamp }}\",\n  \"error\": true\n}"
      },
      "id": "webhook-error-response",
      "name": "Webhook Error Response",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [2000, 750]
    },
    {
      "parameters": {
        "jsCode": "// Supabase Daily Backup\nconst { createClient } = require('@supabase/supabase-js');\n\nconst supabaseUrl = $env.SUPABASE_URL;\nconst supabaseKey = $env.SUPABASE_SERVICE_KEY;\nconst supabase = createClient(supabaseUrl, supabaseKey);\n\nasync function backupTables() {\n  const tables = ['conversations', 'appointments', 'company_knowledge'];\n  const backupData = {};\n  \n  for (const table of tables) {\n    try {\n      const { data, error } = await supabase\n        .from(table)\n        .select('*');\n        \n      if (error) throw error;\n      \n      backupData[table] = {\n        count: data.length,\n        data: data,\n        timestamp: new Date().toISOString()\n      };\n    } catch (error) {\n      backupData[table] = {\n        error: error.message,\n        timestamp: new Date().toISOString()\n      };\n    }\n  }\n  \n  return backupData;\n}\n\nconst backup = await backupTables();\n\nreturn {\n  json: {\n    backup,\n    timestamp: new Date().toISOString(),\n    status: 'completed'\n  }\n};"
      },
      "id": "supabase-backup",
      "name": "Supabase Daily Backup",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [400, 300]
    },
    {
      "parameters": {
        "jsCode": "// Supabase Cleanup - Delete old records\nconst { createClient } = require('@supabase/supabase-js');\n\nconst supabaseUrl = $env.SUPABASE_URL;\nconst supabaseKey = $env.SUPABASE_SERVICE_KEY;\nconst supabase = createClient(supabaseUrl, supabaseKey);\n\n// Calculate dates\nconst thirtyDaysAgo = new Date();\nthirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);\n\nconst ninetyDaysAgo = new Date();\nninetyDaysAgo.setDate(ninetyDaysAgo.getDate() - 90);\n\nasync function cleanupData() {\n  const results = {};\n  \n  // Delete old conversations (30+ days)\n  try {\n    const { count: conversationsDeleted, error: convError } = await supabase\n      .from('conversations')\n      .delete()\n      .lt('created_at', thirtyDaysAgo.toISOString());\n      \n    if (convError) throw convError;\n    results.conversations = { deleted: conversationsDeleted || 0 };\n  } catch (error) {\n    results.conversations = { error: error.message };\n  }\n  \n  // Delete old completed/cancelled appointments (90+ days)\n  try {\n    const { count: appointmentsDeleted, error: apptError } = await supabase\n      .from('appointments')\n      .delete()\n      .in('status', ['completed', 'cancelled'])\n      .lt('created_at', ninetyDaysAgo.toISOString());\n      \n    if (apptError) throw apptError;\n    results.appointments = { deleted: appointmentsDeleted || 0 };\n  } catch (error) {\n    results.appointments = { error: error.message };\n  }\n  \n  return results;\n}\n\nconst cleanup = await cleanupData();\n\nreturn {\n  json: {\n    cleanup,\n    timestamp: new Date().toISOString(),\n    status: 'completed'\n  }\n};"
      },
      "id": "supabase-cleanup",
      "name": "Supabase Cleanup",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [600, 300]
    },
    {
      "parameters": {
        "fromEmail": "{{ $env.SMTP_USER }}",
        "toEmail": "admin@aiquimia.com",
        "subject": "Daily Backup & Cleanup Report",
        "emailType": "text",
        "message": "Daily maintenance completed at {{ $json.timestamp }}\n\nBackup Status: {{ $('Supabase Daily Backup').first().json.status }}\nCleanup Status: {{ $json.status }}\n\nBackup Summary:\n{{ JSON.stringify($('Supabase Daily Backup').first().json.backup, null, 2) }}\n\nCleanup Summary:\n{{ JSON.stringify($json.cleanup, null, 2) }}",
        "credentials": {
          "smtp": {
            "id": "smtp-credentials",
            "name": "SMTP Credentials"
          }
        }
      },
      "id": "email-backup-report",
      "name": "Email Backup Report",
      "type": "n8n-nodes-base.emailSend",
      "typeVersion": 2,
      "position": [800, 300]
    }
  ],
  "connections": {
    "daily-backup-trigger": {
      "main": [
        [
          {
            "node": "supabase-backup",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "webhook-chat": {
      "main": [
        [
          {
            "node": "language-detection",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "language-detection": {
      "main": [
        [
          {
            "node": "get-conversation-history",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "get-conversation-history": {
      "main": [
        [
          {
            "node": "get-company-knowledge",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "get-company-knowledge": {
      "main": [
        [
          {
            "node": "prompt-builder",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "prompt-builder": {
      "main": [
        [
          {
            "node": "llama-ai-call",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "llama-ai-call": {
      "main": [
        [
          {
            "node": "check-ai-error",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "check-ai-error": {
      "main": [
        [
          {
            "node": "process-ai-response",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "error-handler",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "error-handler": {
      "main": [
        [
          {
            "node": "email-admin-error",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "email-admin-error": {
      "main": [
        [
          {
            "node": "webhook-error-response",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "process-ai-response": {
      "main": [
        [
          {
            "node": "save-user-message",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "save-user-message": {
      "main": [
        [
          {
            "node": "save-ai-response",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "save-ai-response": {
      "main": [
        [
          {
            "node": "webhook-response",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "supabase-backup": {
      "main": [
        [
          {
            "node": "supabase-cleanup",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "supabase-cleanup": {
      "main": [
        [
          {
            "node": "email-backup-report",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "active": true,
  "settings": {
    "timezone": "Europe/Lisbon"
  },
  "versionId": "1.0.0",
  "meta": {
    "instanceId": "aiquimia-chatbot"
  }
}
