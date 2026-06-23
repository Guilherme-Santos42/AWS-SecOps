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

AWS:
As 4 Features da AWS mais utilizadas no dia a dia do SOC.

1. AWS Security Hub (Sua "Tela Principal" / SIEM)O Security Hub será a sua primeira tela ao entrar no turno. Como ele centraliza e padroniza todos os alertas no formato ASFF (AWS Security Finding Format), você não precisa abrir serviço por serviço para caçar incidentes.  Feature diária: Insights e Custom Actions. Você utilizará os painéis de Insights para identificar quais recursos têm o maior volume de falhas de segurança e usará as Custom Actions para enviar alertas críticos diretamente para o sistema de tickets do SOC (como Jira ou ServiceNow) ou disparar remediações pelo EventBridge.  

2. Amazon GuardDuty (Seu Detector de Ameaças em Tempo Real)É o motor que vai ditar suas prioridades de incidentes gerando findings comportamentais com severidades de 0.1 a 8.0+.  Feature diária: Análise de Findings por Severidade. Você passará o dia triando alertas como CryptoCurrency:EC2/BitcoinTool (indicação de mineração), UnauthorizedAccess:EC2/SSHBruteForce (ataques externos) ou Stealth:IAMUser/CloudTrailLoggingDisabled (tentativa de evasão de defesa).

3. CloudWatch Logs Insights (Sua Ferramenta de Busca / Threat Hunting)Quando um alerta genérico disparar, você precisará investigar os logs brutos. O Logs Insights possui uma linguagem de busca rápida em massa que você usará constantemente.  Feature diária: Rodar queries para correlacionar eventos. Exemplo: pesquisar nos VPC Flow Logs quais IPs externos receberam o maior volume de tráfego de saída (REJECT vs ACCEPT) de uma instância alvo, ou auditar requisições de DNS suspeitas nos logs do Route 53.
 
4. Amazon Detective (Sua Ferramenta de Análise Forense)Sempre que um alerta do GuardDuty indicar um comprometimento real, o Detective será usado para o processo de Deep Dive (investigação profunda).  Feature diária: Gráficos de Correlação de Entidades e TTP Mapping. O Detective usa teoria de grafos para te mostrar visualmente, em uma linha do tempo de até 1 ano, quais chamadas de API aquele usuário suspeito fez, quais instâncias ele tocou e de onde veio o ataque.  

## Teste
* Agora é preciso ir no Cloudtrail > trilhas > stop logging
* E pronto, recebemos um aviso via e-mail:
<img width="1080" height="72" alt="image" src="https://github.com/user-attachments/assets/7647279a-c248-499d-8ff0-8ebeb70a4ec1" />
