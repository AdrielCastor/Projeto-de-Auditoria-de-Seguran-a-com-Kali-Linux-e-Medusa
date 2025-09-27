# Projeto-de-Auditoria-de-Seguran-a-com-Kali-Linux-e-Medusa

Introdução
Olá! Este repositório documenta minha jornada no desafio de segurança cibernética da DIO, focando em ataques simulados de força bruta usando a ferramenta Medusa no Kali Linux. O objetivo é demonstrar o entendimento de conceitos como brute force em serviços como FTP, formulários web (via DVWA) e SMB, em um ambiente controlado e ético. Tudo foi realizado em VMs isoladas (Kali Linux como atacante e Metasploitable 2 como alvo vulnerável), sem impacto em sistemas reais.

Aviso Ético: Este projeto é puramente educacional e para fins de aprendizado. Ataques de força bruta são ilegais em ambientes não autorizados. Sempre obtenha permissão explícita e use apenas em labs controlados. Recomendo seguir as melhores práticas de cibersegurança, como o uso de senhas fortes e autenticação multifator (MFA).

Eu assisti a todas as vídeo-aulas, explorei os recursos fornecidos (como a documentação do Medusa e DVWA) e adaptei o desafio ligeiramente: criei wordlists personalizadas simples e inclui reflexões sobre mitigação de vulnerabilidades.

Configuração do Ambiente
Ferramentas e Requisitos
VirtualBox: Para criar VMs.
Kali Linux: VM atacante (versão 2023.4, baixada do site oficial: kali.org).
Metasploitable 2: VM alvo vulnerável (baixada de SourceForge). Inclui serviços como FTP (porta 21), SMB (porta 445) e DVWA (porta 80).
Rede: Configurada como "Host-Only Adapter" no VirtualBox para isolar o tráfego (sem acesso à internet externa). IP do Kali: 192.168.56.101; IP do Metasploitable: 192.168.56.102.
Medusa: Pré-instalada no Kali (versão 2.5). Comando de verificação: medusa -h.
DVWA: Instalado no Metasploitable via Apache (acesso via http://192.168.56.102/dvwa). Nível de segurança configurado para "Low" para simulação.
Passos de Configuração
Instale o VirtualBox e crie duas VMs:
Kali: 2GB RAM, 20GB disco, rede Host-Only.
Metasploitable: 1GB RAM, 10GB disco, rede Host-Only.
Inicie as VMs e configure IPs estáticos:
No Kali: sudo ifconfig eth0 192.168.56.101 netmask 255.255.255.0.
No Metasploitable: IPs já configurados por padrão.
Verifique conectividade: ping 192.168.56.102 do Kali.
Instale DVWA no Metasploitable (se não pré-instalado): Baixe via wget e configure o banco MySQL com credenciais padrão (user: admin, pass: password).
Capturas de Tela: Veja a pasta /images para screenshots da configuração de rede e login no DVWA.

Ataques Simulados
Usei wordlists simples para demonstrar brute force. Criei duas wordlists personalizadas:

users.txt: Lista de usuários comuns (ex: msfadmin, root, admin).
passwords.txt: Senhas fracas (ex: password, 123456, admin).
Esses arquivos estão no repositório na pasta /wordlists.

1. Força Bruta em FTP (Porta 21)
Serviço FTP no Metasploitable usa credenciais padrão (user: msfadmin, pass: msfadmin).

Comando Utilizado:


Run
Copy code
medusa -h 192.168.56.102 -u users.txt -P passwords.txt -M ftp -T 10 -t 4 -f -F
-h: Host alvo.
-u e -P: Arquivos de usuários e senhas.
-M ftp: Módulo FTP.
-T 10: Timeout de 10s por tentativa.
-t 4: 4 threads para paralelismo.
-f -F: Para ao encontrar credenciais válidas.
Resultados:

Credenciais encontradas: msfadmin:msfadmin.
Tempo aproximado: 2 minutos (com wordlist pequena).
Validação: ftp 192.168.56.102 e login bem-sucedido.
Screenshot: /images/ftp_bruteforce_success.png.

2. Força Bruta em Formulário Web (DVWA - Login)
DVWA simula um formulário de login vulnerável. Configurei o nível para "Low" (sem CAPTCHA ou rate limiting).

Comando Utilizado (Medusa com módulo HTTP):


Run
Copy code
medusa -h 192.168.56.102 -u admin -P passwords.txt -M http -m DIR:/dvwa/login.php -T 1 -t 4
-m DIR:/dvwa/login.php: Caminho do formulário.
Parâmetros POST simulados: username e password (Medusa injeta automaticamente).
Resultados:

Credenciais encontradas: admin:password.
Validação: Acesso ao dashboard do DVWA.
Nota: Em produção, isso seria bloqueado por OWASP protections.
Screenshot: /images/dvwa_login_success.png.

3. Password Spraying em SMB (Porta 445) com Enumeração de Usuários
Primeiro, enumerei usuários com enum4linux (ferramenta auxiliar no Kali):


Run
Copy code
enum4linux -U 192.168.56.102 > users_enum.txt
Identificou usuários: msfadmin, user, etc.
Em seguida, password spraying com Medusa (testa uma senha comum contra múltiplos usuários):


Run
Copy code
medusa -h 192.168.56.102 -U users_enum.txt -p password -M smbnt -t 4
-p password: Senha única testada em todos usuários (spraying para evitar lockouts).
-M smbnt: Módulo SMB NTLM.
Resultados:

Acesso bem-sucedido a msfadmin:password.
Validação: smbclient //192.168.56.102/msfadmin -U msfadmin%password.
Screenshot: /images/smb_spraying_success.png.

Análise e Reflexões
Vulnerabilidades Identificadas
Força Bruta Geral: Wordlists pequenas funcionaram devido a credenciais fracas. Em cenários reais, isso pode levar a breaches (ex: ataque ao LinkedIn em 2012).
FTP: Sem autenticação segura (use FTPS ou SFTP).
DVWA/Web: Falta de rate limiting e CAPTCHA permite automação.
SMB: Enumeração fácil; spraying evade detecções baseadas em falhas.
Medidas de Mitigação Recomendadas
Senhas Fortas: Use gerenciadores como LastPass; mínimo 12 caracteres, com entropia alta.
Autenticação Multifator (MFA): Implemente via Google Authenticator ou hardware keys.
Rate Limiting e CAPTCHA: No web, use Fail2Ban ou Cloudflare.
Monitoramento: Ferramentas como OSSEC ou Splunk para detectar tentativas de brute force.
Atualizações e Configurações Seguras: Desabilite serviços desnecessários (ex: FTP legacy); use VPN para SMB.
Defesa em Profundidade: Firewalls (iptables no Linux: iptables -A INPUT -p tcp --dport 21 -m limit --limit 3/min -j ACCEPT), IDS como Snort.
Estudos Adicionais: Li a doc do Medusa (sourceforge.net/projects/medusa) e explorei Nmap para scanning inicial (nmap -sV 192.168.56.102). Reflexão: Entendi como brute force é ineficiente contra defesas modernas, mas devastador em setups legacy.

Arquivos no Repositório
/wordlists/users.txt e passwords.txt: Listas usadas.
/scripts/enum_smb.sh: Script simples para automação de enumeração (bash):
bash

Run
Copy code
#!/bin/bash
enum4linux -U $1 > users_enum.txt
medusa -h $1 -U users_enum.txt -p password -M smbnt -t 4
/images/: Capturas de tela (FTP, DVWA, SMB).
README.md: Este arquivo.
Conclusão e Aprendizado
Este projeto reforçou minha compreensão de ataques de força bruta e a importância da prevenção. Aprendi a usar Medusa de forma ética, documentar processos e versionar com Git (commit inicial: setup; commits subsequentes por ataque). Próximos passos: Explorar Hydra como alternativa e testar em DVWA com níveis mais altos.

Se você tiver dúvidas ou sugestões, abra uma issue! Link para o curso DIO: Desafio de Segurança.

GitHub Stats: Repositório criado em [data], com branches main e dev.

Autor: [Seu Nome]
Data: [Data Atual]
Licença: MIT (para fins educacionais).
