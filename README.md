# Assistente de Delivery com AWS Step Functions e Bedrock

Este projeto demonstra como usar AWS Step Functions e AWS Bedrock para criar uma sequência de tarefas interativas com um modelo de linguagem, gerando uma experiência de diálogo contínuo.

## Descrição do Fluxo de Trabalho

Usamos o AWS Step Functions para montar um fluxo com três perguntas relacionadas a um jantar romântico. Cada resposta é baseada na anterior, criando uma conversa fluida.

### Como Funciona

1. **Sugestão de Experiência Gastronômica para Jantar Romântico**  
   O usuário pede sugestões de experiências para um jantar romântico com macarrão. A resposta é guardada para continuar a conversa.

2. **Sugestão de Bebidas para Jantar Romântico**  
   Baseado na primeira resposta, o fluxo agora pede duas sugestões de bebidas que combinem com o jantar romântico. A resposta é novamente salva para a próxima etapa.

3. **Indicação de Lugar para Jantar Romântico em Paris**  
   Com o histórico das duas etapas anteriores, o modelo sugere um local ideal em Paris para o jantar romântico.

O AWS Step Functions armazena cada resposta e as adiciona ao histórico, permitindo um fluxo encadeado de informações.

## Requisitos

Para rodar este exemplo, você precisará de:
- **Conta AWS** com permissões para o AWS Bedrock e AWS Step Functions.
- **Modelo Bedrock** configurado com o ARN do modelo Bedrock Claude 3.

## Estrutura do Arquivo JSON

```json
{
  "Comment": "An example of using Bedrock to chain prompts and their responses together.",
  "StartAt": "Sugestão de Experiência Gastronômica para Jantar Romântico.",
  "States": {
    "Sugestão de Experiência Gastronômica para Jantar Romântico.": {
      "Type": "Task",
      "Resource": "arn:aws:states:::bedrock:invokeModel",
      "Parameters": {
        "ModelId": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-haiku-20240307-v1:0",
        "Body": {
          "anthropic_version": "bedrock-2023-05-31",
          "max_tokens": 2000,
          "messages": [
            {
              "role": "user",
              "content": [
                {
                  "type": "text",
                  "text": "Estou planejando um jantar romântico, e neste jantar servirei um macarrão. Sugira três experiências que combinam para criar uma experiência gastronômica completa."
                }
              ]
            }
          ]
        },
        "ContentType": "application/json",
        "Accept": "*/*"
      },
      "Next": "Add first result to conversation history",
      "ResultPath": "$.result_one",
      "ResultSelector": {
        "result_one.$": "$.Body.content[0].text"
      }
    },
    "Add first result to conversation history": {
      "Type": "Pass",
      "Next": "Sugestão de Bebidas para Jantar Romântico.",
      "Parameters": {
        "convo_one.$": "States.Format('{}\n{}', $.prompt_one, $.result_one.result_one)"
      },
      "ResultPath": "$.convo_one"
    },
    "Sugestão de Bebidas para Jantar Romântico.": {
      "Type": "Task",
      "Resource": "arn:aws:states:::bedrock:invokeModel",
      "Parameters": {
        "ModelId": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-haiku-20240307-v1:0",
        "Body": {
          "anthropic_version": "bedrock-2023-05-31",
          "max_tokens": 200,
          "messages": [
            {
              "role": "user",
              "content": [
                {
                  "type": "text",
                  "text": "string"
                }
              ]
            }
          ]
        },
        "ContentType": "application/json",
        "Accept": "*/*"
      },
      "Next": "Add second result to conversation history",
      "ResultSelector": {
        "result_two.$": "$.Body.content[0].text"
      },
      "ResultPath": "$.result_two"
    },
    "Add second result to conversation history": {
      "Type": "Pass",
      "Next": "Indicação de Lugar para Jantar Romântico em Paris.",
      "Parameters": {
        "convo_two.$": "States.Format('{}\n{}\n{}', $.convo_one.convo_one, $.prompt_two, $.result_two.result_two)"
      },
      "ResultPath": "$.convo_two"
    },
    "Indicação de Lugar para Jantar Romântico em Paris.": {
      "Type": "Task",
      "Resource": "arn:aws:states:::bedrock:invokeModel",
      "Parameters": {
        "ModelId": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-haiku-20240307-v1:0",
        "Body": {
          "prompt.$": "States.Format('{}\n{}', $.convo_two.convo_two, $.prompt_three)",
          "max_tokens": 250
        },
        "ContentType": "application/json",
        "Accept": "*/*"
      },
      "End": true,
      "ResultSelector": {
        "result_three.$": "$.Body.content[0].text"
      }
    }
  }
}
