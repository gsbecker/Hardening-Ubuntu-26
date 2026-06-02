<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
</head>
<body>
    <h1>Guia Completo: Hardening e Segurança Avançada no Ubuntu 26.04</h1>
    <p>Este documento consolida as melhores práticas de hardening para ambientes acadêmicos e profissionais, unificando os scripts de auditoria passiva, defesas de rede locais, integração de alertas via email, implantação do IPS Suricata no modo Drop, e finalizando com a robustez da autenticação via hardware (Chaves FIDO2/U2F).</p>
    
    <h2>Fase 1: Configuração do Webmail (Gmail)</h2>
    <p>Para que o servidor consiga enviar relatórios de segurança de forma automatizada e invisível na inicialização, é necessário gerar uma "Senha de Aplicativo". O Google exige isso para conexões via linha de comando.</p>
    <ol>
        <li>Acesse sua conta Google no navegador: <code>myaccount.google.com</code>.</li>
        <li>No menu lateral esquerdo, clique em <strong>Segurança</strong>.</li>
        <li>Ative a <strong>Verificação em duas etapas</strong> (caso ainda não esteja ativada).</li>
        <li>Role a página da Verificação em duas etapas até o final e clique em <strong>Senhas de app</strong>.</li>
        <li>No campo "Nome do app", digite algo para identificar (ex: Hardening Ubuntu) e clique em Criar.</li>
        <li>Copie a senha de 16 letras gerada (na hora de inserir no script abaixo, utilize-a <strong>sem espaços</strong>).</li>
    </ol>

    <h2>Fase 2: O Script Unificado de Instalação e Hardening</h2>
    <p>Copie o bloco de código abaixo, cole em um editor de texto simples (como o Nano no terminal) e salve o arquivo como <code>hardening.sh</code>.</p>
    <div class="warning">
        <strong>ATENÇÃO:</strong> Antes de executar o script, altere os campos <code>MEU_EMAIL</code> e <code>MINHA_SENHA_APP</code> nas primeiras linhas do código colocando seus dados correspondentes. Note que os possíveis erros de sintaxe nos parâmetros de kernel foram rigorosamente corrigidos para esta versão.
    </div>
<pre><code>#!/bin/bash
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
cat > /etc/sysctl.d/99-security.conf &lt;&lt;EOF
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
curl -s https://install.crowdsec.net | sh
apt install crowdsec crowdsec-firewall-bouncer-iptables -y

# 7. Configuração de Aliases de E-mail
echo "Centralizando alertas do sistema no Gmail..."
cat > /etc/aliases &lt;&lt;EOF
root: $MEU_EMAIL
default: $MEU_EMAIL
EOF

# 8. Configuração Global do MSMTP
echo "Configurando credenciais do servidor de email local..."
cat > /etc/msmtprc &lt;&lt;EOF
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
cat > /usr/local/bin/boot_audit.sh &lt;&lt;'EOF'
#!/bin/bash
EMAIL="SEU_EMAIL_AQUI@gmail.com"
ASSUNTO="[Auditoria] Ubuntu 26.04 Iniciado \$(date +'%d/%m/%Y %H:%M')"
LOG_FALHAS=\$(journalctl -p 4 -b | grep -i "failed\|error" | tail -n 20)
LOG_AUTENTICACAO=\$(last -a | head -n 10)
UPTIME=\$(uptime -p)
MENSAGEM="O sistema foi ligado.

Tempo de atividade inicial: \$UPTIME

Últimos logins:
\$LOG_AUTENTICACAO

Últimos erros críticos do boot atual:
\$LOG_FALHAS"
echo -e "Subject: \$ASSUNTO

\$MENSAGEM" | msmtp --file=/etc/msmtprc -a padrao "\$EMAIL"
EOF
chmod +x /usr/local/bin/boot_audit.sh

# 10. Serviço Systemd para Envio Automático
echo "Ativando serviço automatizado..."
cat > /etc/systemd/system/boot-email.service &lt;&lt;EOF
[Unit]
Description=Envia log de auditoria por email no boot
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/boot_audit.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable boot-email.service
systemctl start boot-email.service

echo "Hardening finalizado! Verifique sua caixa de entrada para confirmar o teste."
</code></pre>
    <p>Para aplicar e executar esta estrutura, use os seguintes comandos no terminal:</p>
<pre><code>chmod +x hardening.sh
sudo bash hardening.sh</code></pre>

    <h2>Fase 3: Atualização e Instalação Base do Suricata</h2>
    <p>Com as barreiras padrão erguidas pelo script anterior, preparamos o terreno para o IPS. Execute linha por linha:</p>
<pre><code>sudo apt update && sudo apt upgrade -y
sudo add-apt-repository ppa:oisf/suricata-stable -y
sudo apt update
sudo apt install suricata jq -y</code></pre>

    <h2>Fase 4: Calibragem do Motor e Rede Local</h2>
    <p>Para evitar que o Suricata bloqueie tráfego local por falsos-positivos de validação, desativaremos a checagem de integridade interna:</p>
    <ol>
        <li>Abra a configuração: <code>sudo nano /etc/suricata/suricata.yaml</code></li>
        <li>Verifique se <code>HOME_NET</code> (próximo ao topo) engloba sua rede adequadamente.</li>
        <li>Procure por <code>stream:</code> e adicione <code>checksum-validation: no</code> logo abaixo, com recuo exato de 2 espaços:</li>
    </ol>
<pre><code>stream:
  checksum-validation: no</code></pre>

    <h2>Fase 5: A Lista de Extermínio (Regras de Bloqueio)</h2>
    <p>Aqui geramos a inteligência para que o tráfego classificado como malicioso seja derrubado (Dropped).</p>
<pre><code>sudo tee /etc/suricata/drop.conf > /dev/null &lt;&lt;EOF
re:classtype: trojan-activity
re:classtype: attempted-admin
re:classtype: attempted-user
re:classtype: web-application-attack
re:classtype: network-scan
re:classtype: bad-unknown
EOF

# Aplicando as assinaturas ativas:
sudo suricata-update -drop-conf /etc/suricata/drop.conf</code></pre>

    <h2>Fase 6: Modo de Interceptação de Fila (NFQUEUE)</h2>
    <p>Altera-se o serviço para interceptar diretamente o tráfego no kernel usando Fila 0.</p>
<pre><code>sudo systemctl stop suricata
sudo mkdir -p /etc/systemd/system/suricata.service.d
sudo tee /etc/systemd/system/suricata.service.d/override.conf > /dev/null &lt;&lt;EOF
[Service]
ExecStart=
ExecStart=/usr/bin/suricata -c /etc/suricata/suricata.yaml --pidfile /run/suricata.pid -user suricata -group suricata -q 0
Type=simple
EOF
sudo systemctl daemon-reload
sudo systemctl start suricata</code></pre>

    <h2>Fase 7: Automação e Redirecionamento de Tráfego</h2>
    <p>Agendando a proteção automática junto ao firewall local e atualização das vacinas.</p>
    <p>Execute <code>sudo crontab -e</code> e cole no final do arquivo:</p>
<pre><code>@reboot sleep 10 && iptables -I INPUT -j NFQUEUE --queue-bypass
@reboot sleep 10 && iptables -I OUTPUT -j NFQUEUE --queue-bypass
0 3,9,15,23 * * * /usr/bin/suricata-update -drop-conf /etc/suricata/drop.conf && systemctl reload suricata</code></pre>
    <p>Ative na sessão atual para uso imediato:</p>
<pre><code>sudo iptables -I INPUT -j NFQUEUE --queue-bypass
sudo iptables -I OUTPUT -j NFQUEUE --queue-bypass</code></pre>

    <h2>Fase 8: Homologação e Teste de Resposta</h2>
    <p>Para comprovar que a blindagem está operante e rejeitando requisições ilícitas:</p>
<pre><code>curl http://testmynids.org/uid/index.html</code></pre>
    <p>O terminal deve apresentar lentidão e o site não carregará. Confirme no log:</p>
<pre><code>sudo cat /var/log/suricata/fast.log | tail -n 2</code></pre>
    <p>Você deverá enxergar a tag <code>[Drop]</code> atestando a destruição do pacote.</p>

    <h2 style="background-color: #f8eceb; color: #c0392b; border-left: 5px solid #c0392b;">Anexo Especial: Implementação de Chaves U2F (FIDO2)</h2>
    <p>Para docentes que possuem chaves criptográficas físicas (como Yubico ou Identiv uTrust), podemos exigi-las durante ações críticas do sistema.</p>
    
    <div class="warning">
        <strong>AVISO DE SEGURANÇA CRÍTICO:</strong> Antes de começar, abra um terminal e digite <code>sudo -s</code>. Mantenha-o aberto durante todo o processo. Ele será sua rota de emergência caso haja erros de digitação nas configurações do PAM, evitando que você perca acesso root!
    </div>
    
    <h3>Passo 1: Instalação e Associação</h3>
<pre><code># Instalar o módulo PAM (padrão da indústria compatível com Yubico e Identiv):
sudo apt install libpam-u2f -y

# Criar a pasta oculta de registro na sua home:
mkdir -p ~/.config/Yubico

# Realizar o mapeamento (Conecte a chave, rode o comando e TOQUE NELA quando piscar):
pamu2fcfg > ~/.config/Yubico/u2f_keys</code></pre>

    <h3>Passo 2: Configurar o Sudo para Exigir a Chave</h3>
    <p>No terminal, execute <code>sudo nano /etc/pam.d/sudo</code> e busque pela linha <code>@include common-auth</code>.</p>
    <ul>
        <li><strong>Opção A (Alta Segurança: Senha + Chave):</strong> Adicione <code>auth required pam_u2f.so</code> <em>imediatamente abaixo</em> da linha encontrada.</li>
        <li><strong>Opção B (Conveniência: Apenas Chave):</strong> Adicione <code>auth sufficient pam_u2f.so</code> <em>imediatamente acima</em> da linha encontrada.</li>
    </ul>
    <p>Salve e feche. Abra uma <strong>nova aba</strong> de terminal (sem fechar a do root) e execute um teste rápido como <code>sudo ls</code>.</p>

    <h3>Passo 3: Configurar a Tela de Login Gráfica (GDM)</h3>
    <p>Opcionalmente, edite <code>sudo nano /etc/pam.d/gdm-password</code>.</p>
    <p>Procure por <code>@include common-auth</code> e aplique exatamente a mesma regra do passo anterior conforme sua preferência.</p>

    <h3>Dica: Cadastrando uma Chave de Backup</h3>
    <p>Se adquirir uma segunda chave, registre-a preservando a primeira com este comando (note o uso de <code>&gt;&gt;</code>):</p>
<pre><code>pamu2fcfg -n >> ~/.config/Yubico/u2f_keys</code></pre>

    <p style="text-align: center; margin-top: 40px; font-weight: bold; color: #7f8c8d;">--- Fim da Apostila de Operações de Segurança ---</p>
</body>
</html>
