# AWS-SecOps
AWS SecOps: Auditoria Contínua e Monitoramento Reativo contra Evasão de Defesa

# AWS SecOps: Auditoria Contínua e Monitoramento Reativo contra Evasão de Defesa

## Cenário e Contexto de Segurança
Em um cenário real de invasão em ambientes cloud, uma das primeiras técnicas utilizadas por atacantes para apagar seus rastros é a **Evasão de Defesa** (mapeada no framework MITRE ATT&CK como *Defense Evasion*). Ao obter acesso administrativo a uma conta AWS, o agente malicioso frequentemente tenta desativar, deletar ou modificar as trilhas de auditoria para operar às cegas e evitar detecção pela equipe de Blue Team.

### O Desafio
Como garantir visibilidade imediata e uma reação em tempo real se um usuário ou atacante tentar interromper os logs globais de auditoria da infraestrutura?

### A Solução
Este projeto implementa uma arquitetura orientada a eventos (*Event-Driven Security*) utilizando serviços nativos da AWS para monitorar, filtrar e alertar instantaneamente a equipe de segurança caso qualquer alteração crítica seja realizada no core de logs da conta.

---

## Arquitetura do Projeto
O fluxo de detecção e notificação foi desenhado utilizando recursos contidos no **AWS Free Tier**:

1. **AWS CloudTrail:** Atuando como a espinha dorsal de auditoria, registrando todas as chamadas de API globais da conta com validação de integridade criptográfica ativa.
2. **Amazon EventBridge:** Funciona como o motor de regras e filtro analítico, interceptando eventos específicos em tempo real.
3. **Amazon SNS (Simple Notification Service):** O barramento de mensagens responsável por processar o payload do evento e disparar o alerta via e-mail (SMTP) para os analistas.

---

## Passo a Passo da Implementação Técnica

### Passo 1: Baseline de Auditoria com AWS CloudTrail
* Criação de uma trilha global (`Trilha-Seguranca-Global`) configurada para capturar todos os eventos de gerenciamento (*Management Events*) de escrita (`Write`).
* Ativação obrigatória da opção **Log File Integrity Validation** (Validação de Integridade do Arquivo de Log). Isso garante que qualquer tentativa posterior de modificar ou deletar os arquivos de log no bucket S3 gere uma quebra na assinatura hash (SHA-256/RSA), invalidando o log para fins forenses e gerando evidência de adulteração.

### Passo 2: Configuração do Canal de Alerta com Amazon SNS
* Criação de um tópico do tipo *Standard* denominado `Alertas-Seguranca-Cloud`.
* Criação de uma assinatura de protocolo `Email` apontando para a caixa de correio da equipe de SecOps.
* Validação do fluxo de opt-in por meio de confirmação do token de subscrição enviado pela AWS para evitar o descarte de alertas por políticas de SPAM.

### Passo 3: Engenharia de Regras no Amazon EventBridge
* Construção de uma regra de monitoramento customizada baseada em padrão de evento (*Event Pattern*).
* O filtro JSON foi estruturado especificamente para ignorar eventos de leitura comuns e capturar de forma cirúrgica as APIs mutáveis e destrutivas do CloudTrail:

```json
{
  "source": ["aws.cloudtrail"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["cloudtrail.amazonaws.com"],
    "eventName": ["StopLogging", "DeleteTrail", "UpdateTrail"]
  }
}
```
## Teste
* Agora é preciso ir no Cloudtrail > trilhas > stop logging
* E pronto, recebemos um aviso via e-mail:
<img width="1080" height="72" alt="image" src="https://github.com/user-attachments/assets/7647279a-c248-499d-8ff0-8ebeb70a4ec1" />
