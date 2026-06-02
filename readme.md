# Guia Completo: Hardening e Segurança Avançada no Ubuntu 26.04

Este documento consolida as melhores práticas de hardening para ambientes acadêmicos e profissionais, unificando os scripts de auditoria passiva, defesas de rede locais, integração de alertas via email, implantação do IPS Suricata no modo Drop, e finalizando com a robustez da autenticação via hardware (Chaves FIDO2/U2F).

## Fase 1: Configuração do Webmail (Gmail)

Para que o servidor consiga enviar relatórios de segurança de forma automatizada e invisível na inicialização, é necessário gerar uma "Senha de Aplicativo". O Google exige isso para conexões via linha de comando.

1. Acesse sua conta Google no navegador: `myaccount.google.com`.
2. No menu lateral esquerdo, clique em **Segurança**.
3. Ative a **Verificação em duas etapas** (caso ainda não esteja ativada).
4. Role a página da Verificação em duas etapas até o final e clique em **Senhas de app**.
5. No campo "Nome do app", digite algo para identificar (ex: Hardening Ubuntu) e clique em Criar.
6. Copie a senha de 16 letras gerada (na hora de inserir no script abaixo, utilize-a **sem espaços**).

## Fase 2: O Script Unificado de Instalação e Hardening

Copie o bloco de código abaixo, cole em um editor de texto simples (como o Nano no terminal) e salve o arquivo como `hardening.sh`.

> **ATENÇÃO:** Antes de executar o script, altere os campos `MEU_EMAIL` e `MINHA_SENHA_APP` nas primeiras linhas do código colocando seus dados correspondentes. Note que os possíveis erros de sintaxe nos parâmetros de kernel foram rigorosamente corrigidos para esta versão.

```bash
#!/bin/bash
MEU_EMAIL="SEU_EMAIL_AQUI@gmail.com"
MINHA_SENHA_APP="SUA_SENHA_DE_16_LETRAS_AQUI_SEM_ESPACOS"

echo "Iniciando Hardening para Ubuntu 26.04..."

# 1. Instalação de Ferramentas de Auditoria e Segurança
apt update
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
