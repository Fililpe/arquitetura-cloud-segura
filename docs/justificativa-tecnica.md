# Justificativa técnica

## Visão geral

A aplicação web é implantada na AWS com um princípio central: somente o balanceador de carga é alcançável pela internet. Todo o resto fica em sub-redes privadas e só pode ser acessado pelos componentes que realmente precisam dele. A segurança é construída em camadas (rede, controle de acesso, criptografia e auditoria), para que a falha de uma camada não exponha os dados.

## 1. Por que cada componente foi incluído

- **VPC customizada (10.0.0.0/16):** delimita o ambiente da aplicação. Usamos uma VPC criada por nós e não a default da AWS, porque na default todas as sub-redes já nascem públicas e na customizada, só é público o que recebe rota explícita para a internet.
- **Seis sub-redes em duas zonas de disponibilidade:** separam as camadas de entrada, aplicação e banco de dados, e permitem que a aplicação continue funcionando se uma zona falhar.
- **Internet Gateway:** única porta de entrada e saída da VPC para a internet. Fica ligado à VPC, e não a uma sub-rede.
- **WAF:** filtra as requisições antes de chegarem à aplicação, com regras gerenciadas da AWS contra ataques comuns como SQL injection e XSS e limite de requisições por IP. Sem ele, qualquer requisição maliciosa chegaria à API.
- **ALB com certificado do ACM:** recebe os clientes por HTTPS (443), distribui as requisições entre as instâncias saudáveis e é o único componente da aplicação exposto. A porta 80 existe apenas para redirecionar para a 443.
- **EC2 (API):** As instâncias EC2 processam as requisições da aplicação e ficam em sub-redes privadas, sem IP público.
- **RDS Multi-AZ:** armazena os dados da aplicação, com uma instância primária e uma standby em outra zona, com failover automático. Criptografado com chave do KMS.
- **S3 de arquivos:** serviço apropriado para os arquivos enviados pelos usuários, com Block Public Access, criptografia em repouso e acesso restrito.
- **Gateway Endpoint do S3:** liga a VPC ao S3 por um caminho privado.
- **NAT Gateway (um por zona):** permite que as instâncias privadas iniciem conexões para a internet para atualizações e comunicação com o SSM, sem aceitar conexões vindas de fora.
- **Security Groups:** firewalls por recurso, que limitam quem pode se comunicar com quem e em quais portas.
- **IAM:** controla o que cada serviço e cada pessoa pode fazer, seguindo o princípio do menor privilégio.
- **Secrets Manager:** guarda a senha do banco fora do código da aplicação.
- **KMS:** gerencia as chaves de criptografia do banco de dados.
- **SSM Session Manager:** acesso administrativo sem abrir portas (ver seção 4).
- **CloudTrail, VPC Flow Logs, logs do WAF e do ALB, CloudWatch e um bucket de logs separado:** registram e permitem analisar eventos para auditoria e segurança.

## 2. Por que determinados recursos estão em sub-redes públicas ou privadas

Só fica em sub-rede pública o que precisa ser alcançado diretamente pela internet, e isso é reduzido ao mínimo.

- **Sub-redes públicas (ALB e NAT Gateway):** o ALB precisa receber os clientes. O NAT precisa de rota para o Internet Gateway para dar saída às instâncias privadas. Nenhum servidor de aplicação e nenhum banco ficam aqui.
- **Sub-redes privadas de aplicação (EC2):** a API só precisa ser alcançada pelo ALB. Sem IP público e sem rota de entrada da internet, ela não pode ser acessada diretamente, mesmo que alguém descubra seu endereço. A saída para a internet é feita pelo NAT da mesma zona.
- **Sub-redes privadas de dados (RDS):** o banco só precisa falar com a API. Essas sub-redes não têm rota para a internet, nem pelo NAT, e o banco não inicia nenhuma conexão. Separar dados e aplicação em sub-redes diferentes cria uma barreira extra: se uma EC2 for comprometida, o banco continua atrás de outra camada de controle.

Sub-redes, tabelas de rota e Security Groups não têm custo adicional, então essa separação aumenta o isolamento sem aumentar a conta.

## 3. Quais conexões são permitidas e quais são bloqueadas

Os Security Groups são encadeados: cada recurso aceita tráfego somente do Security Group do componente anterior da cadeia, nunca de faixas de IP abertas. A única exceção é o ALB, que precisa receber a internet. Todo tráfego que não está listado abaixo é negado por padrão, porque Security Groups só têm regras de permissão.

### Conexões necessárias

- **Acesso dos clientes:** o ALB recebe as requisições da aplicação por HTTPS. A porta HTTP existe somente para redirecionar os clientes para HTTPS.
- **Encaminhamento para a aplicação:** o ALB precisa se comunicar com as instâncias EC2 para encaminhar as requisições e verificar se estão saudáveis. Essa comunicação é restrita ao Security Group do ALB.
- **Acesso ao banco de dados:** a API precisa consultar o RDS para processar as operações da aplicação. O acesso é limitado à porta do banco e ao Security Group da EC2.
- **Acesso aos arquivos:** a aplicação precisa gravar e ler arquivos no S3. O Gateway Endpoint permite essa comunicação por um caminho privado, sem passar pela internet.
- **Leitura do segredo do banco:** a EC2 precisa obter a senha armazenada no Secrets Manager para se conectar ao banco, sem manter credenciais fixas no código.
- **Atualizações das instâncias:** a EC2 precisa iniciar conexões de saída para obter atualizações. O NAT permite essa saída sem tornar as instâncias acessíveis pela internet.
- **Gerenciamento pelo SSM:** o agente na EC2 precisa manter comunicação de saída com o Systems Manager para possibilitar sessões administrativas sem abrir portas de entrada.
- **Acesso administrativo:** administradores autenticados com MFA usam o SSM para iniciar sessões apenas nas instâncias necessárias, sem acesso direto por SSH.
- **Manutenção do banco:** o encaminhamento de porta pelo SSM permite que o administrador acesse o RDS por meio da EC2, que já é a origem autorizada pelo Security Group do banco.
- **Envio de logs:** a EC2 precisa enviar os registros da aplicação ao CloudWatch Logs para monitoramento e auditoria.
- **Replicação do RDS:** a instância primária precisa replicar os dados para a instância standby em outra zona, permitindo o failover do Multi-AZ.

### Conexões bloqueadas

- **Internet para EC2:** as sub-redes de aplicação não têm IP público nem rota de entrada, e o Security Group só aceita o ALB.
- **Internet para RDS:** a sub-rede de dados não tem rota para a internet, e o Security Group só aceita a EC2.
- **ALB para RDS:** o Security Group do RDS só aceita origem do Security Group da EC2.
- **Qualquer origem para as portas 22 e 3389:** nenhuma regra libera essas portas.
- **Internet para o S3:** o Block Public Access impede acesso público.
- **Acesso ao S3 fora do endpoint:** a bucket policy nega qualquer pedido que não venha do Gateway Endpoint.
- **RDS para a internet:** o banco não tem rota de saída e não inicia conexões.

## 4. Como o acesso administrativo é protegido

As máquinas estão em sub-redes privadas, então o administrador não consegue entrar por SSH, e abrir a porta 22 para a internet seria um risco. A solução adotada é o SSM Session Manager:

- **Nenhuma porta de entrada aberta:** não há SSH, IP público nem bastion host.
- **Conexão iniciada de dentro:** o SSM Agent instalado na EC2 conecta para fora pelo NAT ao serviço Systems Manager. O administrador autenticado no IAM é ligado a essa conexão.
- **Autenticação forte:** o grupo de administradores precisa de MFA.
- **Menor privilégio:** a policy permite `ssm:StartSession` (para fazer uma conexão via shell) somente nas instâncias necessárias.
- **Manutenção do banco:** o administrador abre uma sessão SSM com port forwarding pela EC2 até o RDS. Não é preciso criar nenhuma regra nova, porque a EC2 já é a única origem permitida na porta 5432.
- **Rastreabilidade:** cada sessão fica registrada no CloudTrail.

## 5. Como as permissões dos serviços e usuários são limitadas

### Serviços

- A EC2 usa uma role, ligada a ela por um instance profile com credenciais temporárias e renovadas automaticamente. Não existem chaves de acesso fixas no código.
- A role inclui AmazonSSMManagedInstanceCore, para que a instância possa ser gerenciada pelo SSM.
- No S3, a role só tem `s3:PutObject` e `s3:GetObject`, no bucket e no prefixo da aplicação. Não pode apagar, listar nem acessar outros buckets.
- No Secrets Manager, a role só pode ler o segredo do banco (GetSecretValue naquele segredo).
- O banco só aceita conexões do Security Group da EC2.

### Usuários

- **Administradores:** MFA obrigatório e `ssm:StartSession` apenas nas instâncias necessárias.
- **Desenvolvedores:** sem acesso ao ambiente de produção.
- **Auditoria:** somente leitura nos logs.

### Duas camadas de permissão no S3

A policy da role define o que a EC2 pode fazer no bucket. A bucket policy define de onde o pedido pode vir: ela nega qualquer acesso que não passe pelo Gateway Endpoint pela condição `aws:SourceVpce`. O Block Public Access impede acesso público. Uma credencial roubada e usada fora da VPC é recusada pelo S3.

## 6. Como a arquitetura reduz os riscos de exposição de dados e de acesso não autorizado

### Exposição de dados

- O banco e a API não têm nenhum caminho direto a partir da internet.
- O bucket de arquivos não é público e só aceita acessos pelo endpoint da VPC.
- Os dados são criptografados em repouso (RDS com KMS e S3). Em trânsito o TLS termina no ALB. O trecho interno ALB para EC2 usa HTTP dentro da VPC, limitado pelo Security Group.
- A senha do banco fica no Secrets Manager, e não no código.

### Acesso não autorizado

- O WAF e o limite de requisições por IP barram ataques comuns e abusos antes de chegarem à API.
- Security Groups encadeados impedem que qualquer origem fale com um componente além do que o fluxo permite.
- Não há SSH aberto e todo acesso administrativo exige identidade IAM e MFA.
- O menor privilégio limita o estrago de uma conta ou serviço comprometido.

### Defesa em camadas

Se um atacante comprometer a EC2, ele continua limitado a gravar e ler no prefixo da aplicação e a falar com o banco na porta 5432. Isso reduz o alcance do incidente, mas não o elimina, e é por isso que o registro de eventos faz parte do projeto.

## 7. Como os registros de auditoria auxiliam na identificação de atividades suspeitas

- **CloudTrail:** registra quem fez cada chamada de API na conta, seja logins de administradores, sessões SSM ou até alterações em Security Groups ou permissões. Permite identificar, por exemplo, a abertura indevida da porta 22 ou um login sem MFA.
- **VPC Flow Logs:** registram o tráfego de rede aceito e rejeitado. Muitas rejeições vindas de um mesmo IP, ou tentativas nas portas 22 e 5432, indicam varredura ou tentativa de ataque.
- **Logs do WAF e do ALB:** mostram requisições bloqueadas, padrões de ataque e picos de erros.
- **CloudWatch:** reúne os logs e dispara alarmes, por exemplo para aumento de requisições bloqueadas, mudanças em Security Groups e uso da conta root.
- **Bucket de logs separado:** com versionamento e escrita permitida só aos serviços de log, para que um invasor não consiga apagar seus rastros. A equipe de auditoria tem acesso de leitura.

Os registros se complementam: o Flow Logs mostra a tentativa de conexão, o CloudTrail mostra quem alterou a configuração e o WAF mostra o que foi tentado contra a aplicação.

## 8. VPC Endpoint, NAT Gateway e SSM: qual problema cada um resolve

### Gateway Endpoint do S3

**Problema:** a EC2 está em sub-rede privada e o S3 é um serviço público da AWS. Sem endpoint, o tráfego até o S3 sairia pelo NAT e pela internet pública, sendo mais exposto e cobrado por volume de dados.

**Solução:** o endpoint cria uma rota privada, pela rede interna da AWS, sem custo adicional. Ele também permite que a bucket policy aceite apenas acessos vindos da VPC. É adequado ao cenário porque a API troca arquivos com o S3 o tempo todo, e o requisito pede acesso controlado.

### NAT Gateway

**Problema:** as instâncias privadas precisam iniciar conexões para a internet (atualizações e SSM), mas não podem ser alcançáveis de fora.

**Solução:** o NAT permite a saída e bloqueia conexões iniciadas externamente. Mantemos um por zona para evitar um ponto único de falha.

### SSM em vez de VPN ou bastion host

**Problema:** o administrador precisa chegar a recursos privados com segurança. Uma VPN exige gerenciar gateway, clientes e custos fixos, e normalmente dá acesso à rede inteira. Um bastion host é uma máquina exposta e ligada o tempo todo.

**Solução:** o SSM dá acesso por identidade (IAM com MFA), sem porta aberta, com registro das sessões e apenas nas instâncias autorizadas. É adequado a uma equipe pequena e atende aos requisitos de acesso seguro e menor privilégio.

### Decisões e trade-offs assumidos

- **Porta 80:** aberta somente para redirecionar para HTTPS.
- **TLS até o ALB:** o TLS termina no ALB e o trecho até a EC2 usa HTTP dentro da VPC, protegido pelo Security Group. Isso evita gerenciar certificado na EC2. A alternativa seria HTTPS ponta a ponta.
- **Download de arquivos:** como a bucket policy nega acessos fora do endpoint, os downloads passam pela API. Links pré-assinados do S3 seriam negados. Priorizamos a segurança sobre a carga da EC2.
- **Disponibilidade e custo:** um NAT por zona e o RDS Multi-AZ aumentam o custo em troca de resiliência a falhas de zona.
