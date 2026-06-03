## Guia Completo: Hardening e Segurança Avançada no Ubuntu 26.04

Este documento consolida as melhores práticas de hardening para ambientes acadêmicos e profissionais,
unificando os scripts de auditoria passiva, defesas de rede locais, integração de alertas via email,
implantação do IPS Suricata no modo Drop, e
finalizando com a robustez da autenticação via hardware (Chaves FIDO2/U2F).

# Fase 1: Configuração do Webmail (Gmail)

Para que o servidor consiga enviar relatórios de segurança de forma automatizada e invisível na inicialização,
é necessário gerar uma "Senha de Aplicativo". O Google exige isso para conexões via linha de comando.

1. Acesse sua conta Google no navegador: `myaccount.google.com`.
2. No menu lateral esquerdo, clique em **Segurança**.
3. Ative a **Verificação em duas etapas** (caso ainda não esteja ativada).
4. Role a página da Verificação em duas etapas até o final e clique em **Senhas de app**.
5. No campo "Nome do app", digite algo para identificar (ex: Hardening Ubuntu) e clique em Criar.
6. Copie a senha de 16 letras gerada (na hora de inserir no script abaixo, utilize-a **sem espaços**).

# Fase 2: O Script Unificado de Instalação e Hardening


> **ATENÇÃO:** Antes de executar o script, altere os campos `MEU_EMAIL` e `MINHA_SENHA_APP` nas primeiras linhas,
>  do código colocando seus dados correspondentes. Note que os possíveis erros de sintaxe nos parâmetros de kernel foram rigorosamente corrigidos para esta versão.

# !/bin/bash

MEU_EMAIL="SEU_EMAIL_AQUI@gmail.com"
MINHA_SENHA_APP="SUA_SENHA_DE_16_LETRAS_AQUI_SEM_ESPACOS"

Copie o bloco de código abaixo, editando o email e cole em um editor de texto simples (como o Nano no terminal),
e salve o arquivo como `hardening.sh`.

sudo su
sudo nano hardeing.sh #Copie e cole tudo que está abaixo.


echo "Iniciando Hardening para Ubuntu 26.04..."

# 1. Instalação de Ferramentas de Auditoria e Segurança
apt update && full-upgrade -y && auto-remove
apt install libpam-tmpdir apt-listchanges needrestart debsums rkhunter auditd wtmpdb sysvinit-utils msmtp msmtp-mta mailutils ufw -y

# 2. Ativação de Serviços de Auditoria
systemctl enable auditd
systemctl start auditd
sed -i 's/CRON_CHECK="never"/CRON_CHECK="weekly"/' /etc/default/debsums

# 3. Banners Legais de Acesso
echo "Configurando banners de segurança..."
echo "Acesso restrito. O uso nao autorizado e proibido." > /etc/issue
echo "Acesso restrito. O uso nao autorizado e proibido." > /etc/issue.net

# 4. Políticas de Senhas de Usuários
echo "Ajustando politicas de senhas locais..."
sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS 90/' /etc/login.defs
sed -i 's/^PASS_MIN_DAYS.*/PASS_MIN_DAYS 1/' /etc/login.defs
sed -i 's/^PASS_WARN_AGE.*/PASS_WARN_AGE 7/' /etc/login.defs
sed -i 's/^UMASK.*/UMASK 027/' /etc/login.defs

# 5. Hardening de Kernel (Focado em Rede)
echo "Aplicando proteção de Sysctl (Kernel)..."
cat > /etc/sysctl.d/99-security.conf <<EOF
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1
kernel.dmesg_restrict = 1
kernel.sysrq = 0
fs.suid_dumpable = 0
kernel.core_uses_pid = 1
EOF
sysctl -p /etc/sysctl.d/99-security.conf

# 6. Configuração de Rede (UFW + CrowdSec)
echo "Configurando Firewall e CrowdSec..."
ufw default deny incoming
ufw default allow outgoing
ufw enable
curl -s [https://install.crowdsec.net](https://install.crowdsec.net) | sh
apt install crowdsec crowdsec-firewall-bouncer-iptables -y

# 7. Configuração de Aliases de E-mail
echo "Centralizando alertas do sistema no Gmail..."
cat > /etc/aliases <<EOF
root: $MEU_EMAIL
default: $MEU_EMAIL
EOF

# 8. Configuração Global do MSMTP
echo "Configurando credenciais do servidor de email local..."
cat > /etc/msmtprc <<EOF
defaults
auth on
tls on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile ~/.msmtp.log
aliases /etc/aliases
account padrao
host smtp.gmail.com
port 587
from $MEU_EMAIL
user $MEU_EMAIL
password $MINHA_SENHA_APP
account default : padrao
EOF
chown root:root /etc/msmtprc
chmod 600 /etc/msmtprc

# 9. Script de Coleta de Logs de Boot
echo "Criando script de auditoria de inicializacao..."
cat > /usr/local

# 10. Guia Prático: Segurança Avançada (Suricata IPS) no Ubuntu 26.04

#**Objetivo:** Implementar um Sistema de Prevenção de Intrusões (IPS) no modo "Drop" (Bloqueio Ativo),
operando nativamente no sistema, protegendo a navegação e os dados sem interferir em;
 outras ferramentas gráficas de firewall (como o UFW).


## Fase 10.1: Atualização e Instalação Base

A segurança começa com um sistema atualizado.
Nesta etapa, preparamos o terreno e instalamos o motor do Suricata a partir do seu repositório oficial.
Abra o terminal e execute os comandos abaixo:

# Atualizar listas e pacotes do sistema (Upgrades)
sudo apt update && sudo apt full-upgrade -y && sudo auto-remove

# Adicionar o repositório oficial do Suricata
sudo add-apt-repository ppa:oisf/suricata-stable -y

# Atualizar repositórios novamente e instalar os pacotes necessários
sudo apt update
sudo apt install suricata jq -y

## **Copie para colar até a linha acima.**

Ctrl +o # para salvar
Ctrl +x # para sair
sudo chmod +x ./hardening.sh
sudo ./hardening.sh

$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

Abaixo tudo; comando a comando, aqui você mantem seu sistema rodando, mesmo sobre ataques "zero day",
portanto se estiveres com sono ou pressa, deixe para outra hora, leva 15 minutos, mas um erro e nada funciona.

Para ir acima deste ponto gratutito, fornecido abaixo, voce precisará gastar cerca de U$20,000.00 em mensalidades,
fora hardware, portanto, é para médias e grandes empresas com patentes.

A referência é PaloAlto. O tipo de cliente no Brasil, acho que apenas bancos o governo federal.

$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

## Fase 10.2: Calibragem do Motor e Rede Local

Para que o Suricata funcione perfeitamente dentro da própria máquina e não bloqueie o seu tráfego legítimo,
por falsos-positivos (como downloads de atualizações do sistema), precisamos desativar a checagem de integridade local.

# 1. Abra o arquivo principal de configuração:

sudo nano /etc/suricata/suricata.yaml

Edite... com calma, e bem acordado.

No meu caso na linha baixo mudou de /16 para /24, se não tens nenhuma ideia do que fazes mude para /24.

2. **Definição da Rede:** Verifique se a variável `HOME_NET` (próxima ao topo) contém as faixas `"[192.168.0.0/16,10.0.0.0/8,172.16.0.0/12]"`. Isso garante que sua rede local e túneis de VPN estejam cobertos.
3. **Desativação do Checksum:** Pressione `Ctrl + W` para abrir a pesquisa, digite `stream:` e pressione Enter. Imediatamente abaixo dessa linha, adicione a instrução `checksum-validation: no`, respeitando o recuo estrutural do YAML:

O arquivo tem muitas linhas e esta do meio para o final esta parte, tenha calma. tem coisa parecida bem no começo,e não é, vai dar merda.

stream:
  memcap: 64 MiB
  #memcap-policy: ignore
  checksum-validation: no      # reject incorrect csums
  #midstream: false
  #midstream-policy: ignore


4. Salve e feche o arquivo (`Ctrl + O`, `Enter`, `Ctrl + X`).

## Fase 10,3: A Lista de Extermínio (Regras de Bloqueio)

Nesta etapa, criamos o arquivo que orienta a inteligência do sistema sobre quais categorias de tráfego malicioso devem ser ativamente destruídas.

1. Copie o bloco inteiro abaixo e cole no terminal para gerar o arquivo de regras de forma automática:

sudo tee /etc/suricata/drop.conf > /dev/null <<EOF
re:classtype:trojan-activity
re:classtype:attempted-admin
re:classtype:attempted-user
re:classtype:web-application-attack
re:classtype:network-scan
re:classtype:bad-unknown
2100498
EOF

# 2. Baixe as assinaturas de segurança mundiais atualizadas e aplique a nossa inteligência de bloqueio:

sudo suricata-update --drop-conf /etc/suricata/drop.conf

## Fase 10.4: Modo de Interceptação de Fila (NFQUEUE)

O Ubuntu gerencia os serviços de retaguarda de forma rigorosa via `systemd`. Vamos alterar a inicialização de fábrica do Suricata,
para que ele intercepte diretamente o tráfego do kernel (Fila 0), em vez de apenas atuar como um observador passivo.
Copie e cole o bloco completo no terminal:

sudo systemctl stop suricata
sudo mkdir -p /etc/systemd/system/suricata.service.d
sudo tee /etc/systemd/system/suricata.service.d/override.conf > /dev/null <<EOF
[Service]
ExecStart=
ExecStart=/usr/bin/suricata -c /etc/suricata/suricata.yaml --pidfile /run/suricata.pid --user suricata --group suricata -q 0
Type=simple
EOF
sudo systemctl daemon-reload
sudo systemctl start suricata

## Fase 10.5: Automação e Redirecionamento de Tráfego

Para garantir estabilidade e evitar conflitos com a interface gráfica do firewall do Ubuntu (UFW), utilizaremos o agendador de tarefas do sistema (`cron`). Ele ativará a proteção automaticamente ao ligar o computador e atualizará as vacinas contra novas ameaças quatro vezes ao dia.

1. Abra o painel de agendamento do superusuário:

sudo crontab -e

*(Caso seja solicitada a escolha de um editor, digite o número correspondente ao `nano`).*

2. Navegue até a última linha do arquivo e cole estas instruções:
   
@reboot sleep 10 && iptables -I INPUT -j NFQUEUE --queue-bypass
@reboot sleep 10 && iptables -I OUTPUT -j NFQUEUE --queue-bypass
0 1,3,5,7,9,11,13,15,17,19,21,23 * * * /usr/bin/suricata-update --drop-conf /etc/suricata/drop.conf && systemctl reload suricata

4. Salve e saia do editor (`Ctrl + O`, `Enter`, `Ctrl + X`).
5. Para ativar a proteção imediatamente, sem a necessidade de reiniciar a máquina, execute:

sudo iptables -I INPUT -j NFQUEUE --queue-bypass
sudo iptables -I OUTPUT -j NFQUEUE --queue-bypass

## Fase 10.6: Homologação e Teste de Resposta

O seu escudo está armado e operacional. Para homologar a instalação, simularemos uma requisição maliciosa padronizada para sistemas de detecção.

1. No terminal, execute o disparo de teste:

sudo snap install curl
curl http://testmynids.org/uid/index.html

> **Nota de Comportamento:** O comando deve apresentar lentidão extrema (congelamento) ou retornar uma mensagem de falha/tempo limite esgotado. A página não deve carregar.

2. Para comprovar visualmente o bloqueio, em novo terminal, leia o log de eventos de segurança:

sudo cat /var/log/suricata/fast.log | tail -n 2

Deve aparecer coisa do tipo abaixo:
06/03/2026-09:51:02.564453  [Drop] [**] [1:2100498:7] GPL ATTACK_RESPONSE id check returned root [**] [Classification: Potentially Bad Traffic] [Priority: 2] {TCP} 13.227.207.16:80 -> 

# Parte foi ocultada para minha segurança, mas o computador está sofrendo ataques,
sem parar em varias portas, tendando root ou admin em outros sistemas.
como o ataque é controlado, se ele conseguir, apenas não faz nada, mas se fosse um zero day real,
poderia ser um ranwsomeware com travamento imediato e pedido de resgate.
Já existiram, governos pagando para regatar servidores, e de mais dinheiro que o Brasil. É com este tipo de coisa que estais a mexer.

Não recomento ires mais longe que isto, usando TOR, por exemplo, se não fores reporter ou policial. A PF vem de certeza, se não veier, vem a interpool, mandando na PF.

Tem gente na dark-web, que não é este público, e não vai preso, mas trabalham para militares inderetamente, da merda até para policial federal usando de casa,
lembre da história da interpool.

Existe um limite, que voce deve se permitir ser observado, se voce some na internet, pode sumir na vida real. Até hoje a DEA, só prendeu duas pessoas.
Pablo Escobar, e Maduro. E oficialmente, nunca mataram ninguem, porque não há corpos.
É disto que estou falando, sumir na internet, significa para quem não pode, que a pessoa será tele-transportada por ETs, digamos assim.

Neste tipo de ET acretido, passa nas TVs locais ao vivo e todos, menos "Pablo Escobar" e "Nicolas Maduro" entraram na nave espacial.
Apesar de que os ETs, levaram os dois depois de comerem o dinheiro deles.
E os ETs, ainda repartem os lucros com os moradores locais, nem que seja de um País inteiro.

# **Resultado esperado:** O sistema retornará o registro do ataque com a marcação `[Drop]` no início da linha,
confirmando que o pacote malicioso foi neutralizado e descartado antes de alcançar o seu ambiente de trabalho.

Que Deus nos proteja !!!

EOF
