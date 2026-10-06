# Dev AI — OVA de desenvolvimento com IA

Este repositório documenta como importar e usar a imagem `.ova` **Dev AI**. A imagem é uma base Ubuntu para desenvolvimento, com Docker, Python, Node.js, Claude Code e Codex já instalados.

> A OVA não inclui modelos locais, imagens Docker, projetos, chaves, tokens, sessões ou credenciais de terceiros.

## Conteúdo da imagem

| Componente | Situação |
| --- | --- |
| Ubuntu | Base da VM |
| Docker | Instalado, sem containers, volumes ou imagens |
| Node.js e Python | Instalados |
| Claude Code | Instalado; autenticação é individual |
| Codex CLI | Instalado; autenticação é individual |
| SSH | Habilitado para configuração após o primeiro boot |

## 1. Importar a OVA

1. Baixe o arquivo `.ova` por um canal confiável e, se fornecido, confira o SHA-256.
2. No VirtualBox, use **Arquivo → Importar Appliance**. No VMware Workstation, use **File → Open** e escolha a OVA.
3. Como ponto de partida, atribua:
   - 4 vCPUs;
   - 8 GB de RAM no mínimo; 16 GB ou mais para fluxos de IA;
   - 80 GB ou mais de disco se for baixar modelos locais ou imagens Docker.
4. Em **Rede**, escolha uma das opções:
   - **Bridged / Adaptador em ponte**: recomendada para SSH a partir de outros dispositivos da rede. A VM recebe um IP próprio.
   - **NAT**: use se a VM só será acessada pelo host. Crie uma regra de redirecionamento: porta do host `2222` → porta convidada `22`.

Não reutilize snapshots da VM de origem. No primeiro boot, a imagem gera novas chaves de host SSH e uma nova identidade de máquina.

## 2. Primeiro acesso e segurança da conta

A conta padrão é `ai-agent`. O administrador que distribui a OVA deve fornecer um método seguro de primeiro acesso — nunca publique uma senha fixa junto do arquivo.

Ao abrir o console da VM pela primeira vez:

```bash
passwd
sudo apt update
sudo apt upgrade
```

Descubra o endereço IP quando a rede estiver em ponte:

```bash
ip -br addr
hostname -I
```

Confirme que o servidor SSH está ativo:

```bash
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

Se o firewall UFW estiver ativo, permita somente SSH:

```bash
sudo ufw allow OpenSSH
sudo ufw status
```

## 3. Criar uma chave SSH no Windows (host principal)

Abra o **Prompt de Comando (CMD)** no computador host. Gere uma chave Ed25519 protegida por frase secreta:

```cmd
ssh-keygen -t ed25519 -a 100 -f "%USERPROFILE%\.ssh\dev-ai_ed25519" -C "seu-nome@host-dev-ai"
```

Serão criados:

- chave privada: `%USERPROFILE%\.ssh\dev-ai_ed25519` — **não compartilhe**;
- chave pública: `%USERPROFILE%\.ssh\dev-ai_ed25519.pub` — esta será enviada à VM.

## 4. Enviar a chave pública à VM pelo terminal

Substitua `192.168.1.50` pelo IP da VM em modo ponte. Este comando pede a senha inicial apenas uma vez e adiciona somente a **chave pública**:

```cmd
type "%USERPROFILE%\.ssh\dev-ai_ed25519.pub" | ssh ai-agent@192.168.1.50 "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys; chmod 700 ~/.ssh; chmod 600 ~/.ssh/authorized_keys"
```

Com NAT e a regra `2222 → 22`, use:

```cmd
type "%USERPROFILE%\.ssh\dev-ai_ed25519.pub" | ssh -p 2222 ai-agent@localhost "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys; chmod 700 ~/.ssh; chmod 600 ~/.ssh/authorized_keys"
```

Se o SSH por senha não estiver disponível, entre pelo console da VM e cole o conteúdo do arquivo `.pub` em `~/.ssh/authorized_keys`:

```bash
install -d -m 700 ~/.ssh
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

## 5. Conectar via CMD

Teste a chave antes de desabilitar senhas:

```cmd
ssh -i "%USERPROFILE%\.ssh\dev-ai_ed25519" ai-agent@192.168.1.50
```

Para evitar informar IP e chave sempre, crie/edite `%USERPROFILE%\.ssh\config`:

```text
Host dev-ai
    HostName 192.168.1.50
    User ai-agent
    IdentityFile ~/.ssh/dev-ai_ed25519
    IdentitiesOnly yes
    ServerAliveInterval 30
```

Então conecte com:

```cmd
ssh dev-ai
```

Para NAT, inclua `HostName localhost` e `Port 2222`.

Na primeira conexão, confira a impressão digital apresentada pelo SSH. Dentro da VM, ela pode ser consultada com:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

## 6. Desabilitar login por senha

Só faça isto **depois** de confirmar que a conexão por chave funciona em outra janela de terminal.

Na VM:

```bash
sudo tee /etc/ssh/sshd_config.d/99-dev-ai.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
EOF

sudo systemctl restart ssh
```

Teste novamente `ssh dev-ai`. Mantenha o console do hypervisor aberto até validar o acesso.

## 7. Conectar pelo VS Code

1. Instale o [Visual Studio Code](https://code.visualstudio.com/) no host.
2. Instale a extensão **Remote - SSH**.
3. Abra a paleta de comandos (`Ctrl+Shift+P`) e escolha **Remote-SSH: Connect to Host…**.
4. Selecione `dev-ai`, definido no arquivo `%USERPROFILE%\.ssh\config`.
5. Escolha a plataforma Linux quando solicitado e abra uma pasta em `/home/ai-agent`.

O VS Code instalará seu servidor remoto na conta da VM. Isso é esperado e não deve ser incluído novamente em uma OVA-base.

## 8. Usar Claude Code e Codex

Cada pessoa deve autenticar a própria conta; nunca coloque tokens, chaves ou arquivos de sessão na imagem distribuída.

```bash
claude
codex
```

Claude Code requer conexão com a internet e uma conta/assinatura ou credenciais de API compatíveis. Codex também solicitará seu próprio login.

## 9. Docker e modelos locais

Docker está instalado, mas a imagem começa sem imagens e sem containers:

```bash
docker run --rm hello-world
```

Para modelos locais, instale o runtime e modelos somente após expandir o disco conforme sua necessidade. Modelos e caches podem ocupar dezenas de gigabytes.

## Checklist para o mantenedor da OVA

- [ ] Distribuir a OVA e checksum por canal confiável.
- [ ] Definir um método seguro de primeiro acesso para `ai-agent`.
- [ ] Não publicar senhas, chaves privadas, tokens ou endereços internos.
- [ ] Confirmar que a VM inicializa e gera uma nova impressão digital SSH.
- [ ] Tirar um snapshot limpo antes de testes ou atualizações.
- [ ] Atualizar o sistema periodicamente e gerar uma nova OVA após mudanças relevantes.

## Solução de problemas

| Sintoma | Verificação |
| --- | --- |
| `Connection refused` | `sudo systemctl status ssh` e a regra NAT/ponte |
| Timeout | IP incorreto, firewall do host ou rede da VM |
| `Permission denied (publickey)` | permissões: `700 ~/.ssh`, `600 ~/.ssh/authorized_keys` |
| VS Code não conecta | teste `ssh dev-ai` no CMD antes de usar a extensão |
| Aviso de chave de host alterada | esperado para uma VM clonada; valide a nova impressão digital antes de remover a entrada antiga |

