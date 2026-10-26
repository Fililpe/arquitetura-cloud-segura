# Fluxos de comunicação

Todas as regras são aplicadas por Security Groups (SG). Sempre que possível a origem permitida é outro SG, e não uma faixa de IP.

## Conexões permitidas

| # | Origem | Destino | Protocolo | Porta | Finalidade |
|---|---|---|---|---|---|
| 1 | Internet (cliente) | ALB | HTTPS | 443 | Acesso do cliente à aplicação, com WAF e certificado do ACM |
| 2 | Internet (cliente) | ALB | HTTP | 80 | Apenas redireciona para 443 |
| 3 | ALB (SG do ALB) | EC2 (SG da EC2) | TCP | 8080 | Encaminhar requisições e health check |
| 4 | EC2 (SG da EC2) | RDS (SG do RDS) | TCP | 5432 | Consultas ao banco |
| 5 | EC2 | S3 via Gateway Endpoint | HTTPS | 443 | Gravar e ler arquivos enviados pelos usuários, sem passar pela internet |
| 6 | EC2 | Secrets Manager via NAT | HTTPS | 443 | Ler a senha do banco |
| 7 | EC2 | Internet via NAT | HTTPS | 443 | Atualizações de pacotes (somente saída) |
| 8 | EC2 (agente SSM) | SSM via NAT | HTTPS | 443 | Manter o canal da sessão administrativa |
| 9 | Admin (MFA) | SSM | HTTPS | 443 | Abrir sessão de manutenção na EC2 |
| 10 | Admin via SSM port forwarding pela EC2 | RDS | TCP | 5432 | Manutenção do banco |
| 11 | EC2 (agente CloudWatch) | CloudWatch Logs via NAT | HTTPS | 443 | Envio de logs da aplicação |
| 12 | RDS primário | RDS standby (outra AZ) | Interno AWS | - | Replicação Multi-AZ |

## Conexões bloqueadas

| Origem | Destino | Porta | Como é bloqueado |
|---|---|---|---|
| Internet | EC2 | qualquer | EC2 em subnet privada, sem IP público. SG da EC2 só aceita o SG do ALB |
| Internet | RDS | 5432 | RDS em subnet privada sem rota para internet. SG do RDS só aceita o SG da EC2 |
| Internet | Bucket S3 | 443 | Block Public Access ligado e bucket policy que nega acesso fora do Gateway Endpoint |
| Qualquer IP (0.0.0.0/0) | EC2 | 22 (SSH) / 3389 (RDP) | Nenhuma porta de administração aberta. Manutenção só por SSM |
| ALB | RDS | 5432 | SG do RDS não aceita o SG do ALB |
| Internet | EC2 (conexão iniciada de fora) | qualquer | NAT Gateway só permite conexões iniciadas de dentro para fora |
| RDS | Internet | qualquer | Subnets de dados sem rota para internet. RDS não inicia conexões |
