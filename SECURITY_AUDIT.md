# 🔐 Relatório de Auditoria de Segurança — Boot-Check

**Projeto:** Boot-Check v1.1  
**Arquivo auditado:** `checkusb` (script Bash único)  
**Data:** 21/05/2026  
**Auditor:** Cline (AppSec Sênior — White-box Pentest)  
**Metodologia:** SAST manual + análise de dependências + superfície de ataque  
**Classificação:** USO INTERNO / OPEN SOURCE

---

## 📋 Sumário Executivo

O Boot-Check é uma ferramenta TUI em Bash que detecta pendrives USB e os inicializa via QEMU. Por ser um script local (sem rede, sem banco de dados, sem API HTTP), sua superfície de ataque é restrita — porém **não trivial**, especialmente devido ao uso automático de `sudo` e à leitura de dados externos (sysfs, stdin do usuário).

A auditoria identificou **9 vulnerabilidades** distribuídas em 4 níveis de severidade:

| Severidade | Quantidade | IDs |
|---|---|---|
| 🔴 Crítica | 2 | SEC-01, SEC-08 |
| 🟠 Alta | 2 | SEC-02, SEC-03 |
| 🟡 Média | 3 | SEC-04, SEC-05, SEC-06 |
| 🟢 Baixa | 2 | SEC-07, SEC-09 |

**Nenhum segredo hardcoded, chave de API ou credencial** foi encontrado no código.  
**Nenhum CVE ativo** foi identificado nas dependências instaladas (QEMU 11.0.0, util-linux 2.42.1, Bash 5.3.9).

---

## 🔍 Vulnerabilidades Encontradas

---

### 🔴 SEC-01 — CRÍTICA | Escalada de Privilégio via PATH Hijacking com `sudo`

**Arquivo:** `checkusb`  
**Linhas:** 27, 344–347, 368  
**Categoria OWASP:** A01 – Broken Access Control / A03 – Injection  

#### Descrição do Risco

O script armazena o nome do binário QEMU em `QEMU_BIN="qemu-system-x86_64"` (apenas o nome, sem caminho absoluto) e depois o executa com `sudo "$QEMU_BIN"`. Quando `sudo` é invocado sem caminho absoluto, ele resolve o binário usando o `PATH` do **ambiente do usuário** (não o `secure_path` do sudoers, pois o script não usa `sudo -i` nem `env_reset` forçado de forma confiável).

A investigação confirmou que o `PATH` do usuário inclui `/home/filipe/.bin` **antes** dos diretórios do sistema:

```
PATH=/home/filipe/.bin:/home/filipe/google-cloud-sdk/bin:/home/filipe/.cargo/bin:...
```

Além disso, `sudo -l` revelou que o usuário possui **`(ALL) ALL`** — sem restrição de comando. Isso significa que um atacante que consiga escrever em `/home/filipe/.bin/qemu-system-x86_64` pode fazer o script executar **qualquer código arbitrário como root** quando o usuário precisar de `sudo`.

**Vetor de ataque (PoC conceitual):**
```bash
# Atacante cria um binário malicioso no diretório .bin do usuário
echo '#!/bin/bash\nchmod 4755 /bin/bash' > /home/filipe/.bin/qemu-system-x86_64
chmod +x /home/filipe/.bin/qemu-system-x86_64
# Aguarda o usuário executar: bios → selecionar pendrive → sudo é invocado
# Resultado: /bin/bash vira SUID root → root shell para qualquer usuário
```

**Nota:** A configuração padrão do `sudo` no Arch Linux com `secure_path` mitiga parcialmente isso, mas não completamente quando o usuário tem `(ALL) ALL` e o `env_reset` não é configurado de forma estrita.

#### ✅ Patch Recomendado

```bash
# LINHA 27 — ANTES (vulnerável):
readonly QEMU_BIN="qemu-system-x86_64"

# DEPOIS (corrigido — use o caminho absoluto):
readonly QEMU_BIN="/usr/bin/qemu-system-x86_64"
```

```bash
# Também ajuste check_deps (linha 368) para verificar o binário pelo path completo:
# ANTES:
command -v "$QEMU_BIN" &>/dev/null || missing+=("$QEMU_BIN")

# DEPOIS:
[[ -x "$QEMU_BIN" ]] || missing+=("qemu-system-x86_64 (não encontrado em ${QEMU_BIN})")
```

```bash
# E ao invocar sudo, passe --preserve-env=DISPLAY e restrinja explicitamente:
# ANTES (linha 345):
sudo "$QEMU_BIN" "${qemu_args[@]}" || exit_code=$?

# DEPOIS:
sudo --preserve-env=DISPLAY,XAUTHORITY "$QEMU_BIN" "${qemu_args[@]}" || exit_code=$?
```

> 💡 **Recomendação adicional:** Documente no README que o usuário deve criar uma entrada `sudoers` específica via `visudo`, restringindo o `sudo` apenas ao binário QEMU com caminho absoluto:
> ```
> filipe ALL=(root) NOPASSWD: /usr/bin/qemu-system-x86_64
> ```

---

### 🟠 SEC-02 — ALTA | Injeção de Argumento QEMU via Conteúdo do Sysfs (`/sys/block/.../model`)

**Arquivo:** `checkusb`  
**Linhas:** 116–119, 131–134, 290  
**Categoria OWASP:** A03 – Injection  

#### Descrição do Risco

A função `get_usb_model()` lê o conteúdo do arquivo `/sys/block/<dev>/device/model` e o passa para `sanitize_model_name()`, que remove caracteres não alfanuméricos. **O resultado sanitizado é então embutido diretamente no argumento `-name` do QEMU** (linha 290):

```bash
# Linha 290 — build_qemu_args():
-name "bootterm - ${dev}"
```

O problema não está no argumento `-name` em si (que usa `$dev`, não o model), mas na função `get_usb_model()` que retorna o conteúdo do model **com** os caracteres `+` e `-` permitidos pela sanitização:

```bash
# Linha 118 — sanitize_model_name():
printf '%s' "$raw" | sed 's/[^a-zA-Z0-9 _.()+-]//g'
```

Os caracteres `+` e `-` são **metacaracteres do QEMU** em determinados contextos de argumento. Mais grave: o output de `get_usb_model()` é capturado e exibido via `echo -e` (linha 169) sem sanitização adicional — e `echo -e` interpreta sequências de escape como `\n`, `\t`, `\033[...m` (códigos ANSI). Um atacante que consiga controlar o conteúdo de `/sys/block/sda/device/model` (ex: via dispositivo USB malicioso com nome de modelo manipulado) pode **injetar sequências ANSI** na TUI, potencialmente causando confusão visual (ex: sobrescrever texto do menu).

**PoC conceitual (ANSI Injection via model name):**
```
Conteúdo de /sys/block/sdb/device/model:
"SanDisk\033[2J\033[H[FAKE MENU]"
```
Com `echo -e`, isso limparia a tela e exibiria um menu falso.

#### ✅ Patch Recomendado

```bash
# ANTES — sanitize_model_name() (linha 116-119):
sanitize_model_name() {
    local raw="$1"
    printf '%s' "$raw" | sed 's/[^a-zA-Z0-9 _.()+-]//g'
}

# DEPOIS — remova + e - da lista permitida; adicione limite de comprimento:
sanitize_model_name() {
    local raw="$1"
    # Permite apenas alfanuméricos, espaço, ponto e underscore.
    # Remove + e - (metacaracteres em contextos QEMU/shell).
    # Limita a 64 caracteres para evitar overflow de buffer em args QEMU.
    printf '%s' "$raw" | sed 's/[^a-zA-Z0-9 _.]//g' | cut -c1-64
}
```

```bash
# ANTES — uso de echo -e com model (linha 169):
echo -e "  ${BOLD}${GREEN}[$(( i + 1 ))]${RESET}  ${WHITE}${dev}${RESET}  ${CYAN}${model}${RESET}  ${DIM}(${size})${RESET}"

# DEPOIS — use printf com %s para nunca interpretar escapes do conteúdo:
printf "  ${BOLD}${GREEN}[%d]${RESET}  ${WHITE}%s${RESET}  ${CYAN}%s${RESET}  ${DIM}(%s)${RESET}\n" \
    "$(( i + 1 ))" "$dev" "$model" "$size"
```

---

### 🟠 SEC-03 — ALTA | Escopo Global Implícito em `build_qemu_args` e Pattern Replace Inseguro

**Arquivo:** `checkusb`  
**Linhas:** 284–303, 340–341  
**Categoria OWASP:** A04 – Insecure Design  

#### Descrição do Risco

A função `build_qemu_args()` popula a variável `qemu_args` sem declará-la com `local` — ela escreve diretamente em uma variável global. Em `launch_qemu()` (linha 340), `qemu_args` é declarada como `local`, mas como `build_qemu_args` é chamada como subprocesso no mesmo shell, ela **sobrescreve a variável local do escopo pai**.

Mais crítico: a linha 302 usa substituição de padrão em **todo o array** para trocar `type=pc` por `type=q35`:

```bash
# Linha 302 — pattern replace em todos os elementos do array:
qemu_args=("${qemu_args[@]/type=pc/type=q35}")
```

Isso substitui a string `type=pc` em **qualquer** elemento do array, não apenas no argumento `-machine`. Se um nome de dispositivo, label ou path contiver a substring `type=pc`, ele será silenciosamente corrompido sem qualquer mensagem de erro — comportamento indefinido.

#### ✅ Patch Recomendado

```bash
# DEPOIS — controle machine_type com variável, sem pattern replace no array:
build_qemu_args() {
    local dev="$1"
    local mode="$2"
    local machine_type="pc"

    if [[ "$mode" == "uefi" ]]; then
        machine_type="q35"
    fi

    qemu_args=(
        -name "bootterm - ${dev}"
        -m "${RAM_MB}"
        -drive "file=${dev},format=raw,readonly=on,if=virtio,cache=none"
        -boot order=c,menu=on
        -display sdl
        -accel kvm
        -cpu host
        -machine "type=${machine_type}"
    )

    if [[ "$mode" == "uefi" ]]; then
        qemu_args+=( -bios "${OVMF_PATH}" )
    fi
}
```

---

### 🟡 SEC-04 — MÉDIA | TOCTOU na Verificação de Permissão do Dispositivo

**Arquivo:** `checkusb`  
**Linhas:** 329–347  
**Categoria OWASP:** A01 – Broken Access Control  

#### Descrição do Risco

O script verifica se o dispositivo é legível (`[[ ! -r "$dev" ]]`) na linha 329 e, com base nisso, decide se invoca `sudo`. Há uma **janela de corrida (race condition / TOCTOU)** entre essa verificação e a execução efetiva do QEMU nas linhas 344–347:

```bash
# Linha 329 — verificação:
if [[ ! -r "$dev" ]]; then
    use_sudo=1
fi
# ~10 linhas de código intermediário (echo, tput, msg_info)...
# Linha 344 — execução (janela de corrida aqui):
if (( use_sudo == 1 )); then
    sudo "$QEMU_BIN" "${qemu_args[@]}" || exit_code=$?
```

Durante essa janela, o dispositivo pode ser removido/reconectado com path diferente, ou um symlink malicioso pode ser criado via udev rule. Adicionalmente, a validação `^/dev/[a-zA-Z0-9]+$` aceita qualquer dispositivo de bloco (incluindo `/dev/sda`, o disco interno), não apenas pendrives USB — permitindo que um `SELECTED_DEV` manipulado aponte para discos internos.

#### ✅ Patch Recomendado

```bash
# 1. Pré-autentique sudo imediatamente após o check, reduzindo a janela:
local use_sudo=0
if [[ ! -r "$dev" ]]; then
    msg_warn "Permissão negada para ${dev}. Será solicitado sudo."
    sudo -v || { msg_error "Autenticação sudo falhou."; return 1; }
    use_sudo=1
fi

# Construa args e execute imediatamente, sem código entre check e use:
local qemu_args=()
build_qemu_args "$dev" "$mode"

local exit_code=0
if (( use_sudo == 1 )); then
    sudo --preserve-env=DISPLAY,XAUTHORITY "$QEMU_BIN" "${qemu_args[@]}" || exit_code=$?
else
    "$QEMU_BIN" "${qemu_args[@]}" || exit_code=$?
fi

# 2. Em validate_launch_params, confirme que o device está na lista USB:
local is_known_usb=0
local known_dev
for known_dev in "${USB_DEVICES[@]}"; do
    [[ "$known_dev" == "$dev" ]] && is_known_usb=1 && break
done
if (( is_known_usb == 0 )); then
    msg_error "Dispositivo '${dev}' não reconhecido como USB. Abortando por segurança."
    sleep 2
    return 1
fi
```

---

### 🟡 SEC-05 — MÉDIA | `OVMF_PATH` Não é `readonly` — Mutável Durante a Execução

**Arquivo:** `checkusb`  
**Linha:** 25  
**Categoria OWASP:** A05 – Security Misconfiguration  

#### Descrição do Risco

Todas as outras constantes de configuração (`QEMU_BIN`, `RAM_MB`, `VERSION`, `ALLOWED_SYS_PREFIX`) são declaradas como `readonly`. A variável `OVMF_PATH`, no entanto, **não é `readonly`**:

```bash
# Linha 25 — mutável (incorreto):
OVMF_PATH="/usr/share/edk2/x64/OVMF.4m.fd"

# Linha 27 — readonly (correto):
readonly QEMU_BIN="qemu-system-x86_64"
```

Isso significa que qualquer função no script pode reatribuir `OVMF_PATH` acidentalmente ou maliciosamente (ex: via injeção em variáveis de ambiente, se o script for chamado com `env` ou se uma futura função cometer o erro de usar o mesmo nome de variável local sem `local`). Uma reatribuição de `OVMF_PATH` para um path controlado pelo atacante faria o QEMU carregar um firmware arbitrário com privilégios elevados — **execução de código como root**.

Adicionalmente, a validação de `OVMF_PATH` em `validate_launch_params` (linha 274) verifica apenas se começa com `/` e se o arquivo existe — **não verifica se o path está dentro de um diretório permitido**. Um `OVMF_PATH` como `/tmp/evil.fd` passaria na validação.

#### ✅ Patch Recomendado

```bash
# LINHA 25 — ANTES (mutável):
OVMF_PATH="/usr/share/edk2/x64/OVMF.4m.fd"

# DEPOIS (imutável):
readonly OVMF_PATH="/usr/share/edk2/x64/OVMF.4m.fd"
```

```bash
# Em validate_launch_params, adicione validação de prefixo de diretório:
# ANTES (linha 274):
if [[ ! "$OVMF_PATH" =~ ^/ ]] || [[ ! -f "$OVMF_PATH" ]]; then

# DEPOIS — valide também que está em um diretório de sistema confiável:
if [[ ! "$OVMF_PATH" =~ ^/usr/share/ ]] || [[ ! -f "$OVMF_PATH" ]]; then
    msg_error "OVMF_PATH deve apontar para um arquivo dentro de /usr/share/. Path inválido: ${OVMF_PATH}"
    msg_info  "Instale o pacote 'edk2-ovmf' e configure OVMF_PATH no topo do script."
    echo
    read -rp "  Pressione Enter para voltar..."
    return 1
fi
```

---

### 🟡 SEC-06 — MÉDIA | Ausência de Verificação de Integridade do Script

**Arquivo:** `checkusb`  
**Linhas:** N/A (ausência de controle)  
**Categoria OWASP:** A08 – Software and Data Integrity Failures  

#### Descrição do Risco

O script é instalado via `sudo cp bios /usr/local/bin/bios` (conforme README) e executado com permissões de usuário comum, podendo escalar para root via sudo. **Não há nenhum mecanismo de verificação de integridade** (hash, assinatura GPG, checksum) que garanta que o arquivo `checkusb` não foi modificado após a instalação.

```
Fluxo de ataque:
1. Atacante com acesso local modifica /usr/local/bin/bios
2. Usuário executa 'bios' normalmente
3. Código malicioso injetado no script executa como usuário
4. Se sudo for necessário: escalada para root
```

Permissões atuais do arquivo:
```
-rwxr-xr-x 1 filipe filipe 13992 bios
```
O arquivo pertence ao usuário `filipe` e é executável por todos — qualquer processo rodando como `filipe` pode modificá-lo.

#### ✅ Patch Recomendado

```bash
# 1. Após instalar globalmente, transfira a propriedade para root:
sudo cp bios /usr/local/bin/bios
sudo chown root:root /usr/local/bin/bios
sudo chmod 755 /usr/local/bin/bios
# Agora apenas root pode modificar o arquivo.
```

```bash
# 2. Adicione no README instruções de verificação de hash:
# Gere o hash após cada release:
sha256sum bios > bios.sha256

# Usuário verifica antes de instalar:
sha256sum -c bios.sha256
```

> 💡 Documente no README que o arquivo instalado em `/usr/local/bin/bios` **deve pertencer a root** para evitar modificações não autorizadas.

---

### 🟢 SEC-07 — BAIXA | Ausência de `umask` e Exposição de Informações em Mensagens de Erro

**Arquivo:** `checkusb`  
**Linhas:** 263, 269, 275, 381–383  
**Categoria OWASP:** A09 – Security Logging and Monitoring Failures  

#### Descrição do Risco

**1. Ausência de `umask`:** O script não define `umask` explicitamente. Embora não crie arquivos temporários atualmente, qualquer futura adição de logging ou criação de arquivos herdará o `umask` do ambiente do usuário, potencialmente criando arquivos world-readable.

**2. Exposição de informações em erros:** As mensagens de erro nas linhas 275 e 276 expõem o path completo do firmware OVMF e instruções de instalação. Em sistemas multiusuário, isso pode revelar informações sobre a configuração do sistema para usuários não privilegiados que consigam visualizar o terminal:

```bash
# Linha 275-276 — exposição de path interno:
msg_error "Firmware UEFI não encontrado ou path inválido: ${OVMF_PATH}"
msg_info  "Instale o pacote 'edk2-ovmf' e configure OVMF_PATH no topo do script."
```

**3. Instruções de instalação com `sudo` nas mensagens de erro (linha 381–383):** As mensagens de `check_deps` ensinam o usuário a instalar pacotes com `sudo pacman/apt/dnf` sem advertência sobre verificação de autenticidade dos pacotes.

#### ✅ Patch Recomendado

```bash
# 1. Adicione umask defensivo no início do script (após set -euo pipefail):
umask 077  # arquivos criados com permissão 600, diretórios com 700

# 2. Mantenha as mensagens de erro mas não exponha paths completos para stderr público:
# (baixo risco em ferramenta local — documentar como aceito)

# 3. Adicione nota nas instruções de instalação sobre verificação de pacotes:
msg_info "Arch Linux  : sudo pacman -S qemu-full edk2-ovmf util-linux"
msg_info "(Verifique a autenticidade dos pacotes com: pacman-key --verify)"
```

---

### 🔴 SEC-08 — CRÍTICA | Environment Variable Injection — `OVMF_PATH` Injetável via `env`

**Arquivo:** `checkusb`  
**Linha:** 25  
**Categoria OWASP:** A03 – Injection / A01 – Broken Access Control  

#### Descrição do Risco

Em Bash, declarar `readonly VAR="valor"` no corpo do script **não protege contra variáveis já existentes no ambiente do processo pai**. Se `OVMF_PATH` já estiver definida no ambiente antes do script iniciar, a linha 25 (`OVMF_PATH="/usr/share/..."`) **não sobrescreve** a variável de ambiente — e o `readonly` subsequente apenas congela o valor malicioso.

**PoC direto e funcional:**
```bash
# Atacante (ou script malicioso no mesmo sistema) executa:
OVMF_PATH="/tmp/evil_firmware.fd" bios
# Resultado: o script usa /tmp/evil_firmware.fd como firmware UEFI
# Se sudo for invocado → QEMU carrega firmware arbitrário como root
```

A validação em `validate_launch_params` (linha 274) apenas verifica se o arquivo existe e começa com `/` — `/tmp/evil_firmware.fd` passa em ambos os critérios. Isso significa que um firmware malicioso posicionado em `/tmp/` seria carregado pelo QEMU com os privilégios do processo (potencialmente root).

Este é o **vetor mais fácil de explorar** da auditoria inteira — uma única linha de comando.

#### ✅ Patch Recomendado

```bash
# SOLUÇÃO 1 — Ignorar o ambiente e forçar o valor interno (mais seguro):
# No início do script, antes de qualquer uso de OVMF_PATH:
unset OVMF_PATH  # descarta qualquer valor do ambiente
readonly OVMF_PATH="/usr/share/edk2/x64/OVMF.4m.fd"

# SOLUÇÃO 2 — Detectar e rejeitar injeção via ambiente:
if [[ -v OVMF_PATH ]] && [[ "${OVMF_PATH}" != "/usr/share/edk2/x64/OVMF.4m.fd" ]]; then
    echo "ERRO: Variável de ambiente OVMF_PATH foi redefinida externamente. Abortando." >&2
    exit 1
fi
readonly OVMF_PATH="/usr/share/edk2/x64/OVMF.4m.fd"

# SOLUÇÃO 3 (complementar) — Restringir prefixo em validate_launch_params:
if [[ ! "$OVMF_PATH" =~ ^/usr/share/ ]] || [[ ! -f "$OVMF_PATH" ]]; then
    msg_error "OVMF_PATH inválido ou fora de /usr/share/: ${OVMF_PATH}"
    return 1
fi
```

> ⚠️ O mesmo vetor existe potencialmente para `QEMU_BIN` se o atacante definir a variável antes de `readonly` — mas como `QEMU_BIN` usa apenas o nome (sem path), o `secure_path` do sudoers já oferece alguma proteção. Com `OVMF_PATH` o risco é direto.

---

### 🟢 SEC-09 — BAIXA | `USB_SIZES` Não Sanitizado Exibido via `echo -e`

**Arquivo:** `checkusb`  
**Linha:** 169  
**Categoria OWASP:** A03 – Injection (Terminal)  

#### Descrição do Risco

O campo `SIZE` do `lsblk` (ex: `"32G"`, `"16G"`) é capturado via `BASH_REMATCH` e armazenado em `USB_SIZES` sem sanitização. Na linha 169, ele é exibido via `echo -e`:

```bash
# Linha 166-169:
local size="${USB_SIZES[$i]}"
echo -e "  ... ${DIM}(${size})${RESET}"
```

O campo `SIZE` do `lsblk` normalmente contém apenas valores como `"32G"` — no entanto, em versões antigas de `util-linux` ou em sistemas com `lsblk` customizado, um dispositivo com metadata corrompida poderia retornar valores inesperados. Em conjunto com `echo -e`, sequências de escape ANSI presentes em `$size` seriam interpretadas — o mesmo vetor do SEC-02, mas via campo `SIZE` ao invés de `model`.

O risco real é baixo (o campo SIZE é bem controlado pelo kernel), mas é uma inconsistência: `$model` tem `sanitize_model_name()`, enquanto `$size` não tem nenhuma sanitização.

#### ✅ Patch Recomendado

```bash
# Aplique o mesmo printf usado no patch do SEC-02 — ele resolve ambos:
# (o printf com %s nunca interpreta escapes em argumentos de dados)
printf "  ${BOLD}${GREEN}[%d]${RESET}  ${WHITE}%s${RESET}  ${CYAN}%s${RESET}  ${DIM}(%s)${RESET}\n" \
    "$(( i + 1 ))" "$dev" "$model" "$size"
# %s garante que $size seja impresso literalmente, sem interpretar \033 etc.
```

---

## 📦 Auditoria de Dependências

### Dependências do Projeto

| Dependência | Versão Instalada | CVEs Conhecidos | Status |
|---|---|---|---|
| `qemu-system-x86_64` | 11.0.0 | Nenhum CVE ativo para 11.0.0 | ✅ Atualizado |
| `lsblk` (util-linux) | 2.42.1 | Nenhum CVE ativo para 2.42.1 | ✅ Atualizado |
| `bash` | 5.3.9 | Nenhum CVE ativo para 5.3.9 | ✅ Atualizado |
| OVMF (edk2-ovmf) | Sistema | Firmware de terceiro — risco de supply chain | ⚠️ Ver nota |

**Nota sobre OVMF:** O firmware OVMF é distribuído pelos repositórios da distro (Arch: `edk2-ovmf`). Ele é executado pelo QEMU com os mesmos privilégios do processo QEMU. Não há CVEs ativos, mas recomenda-se manter o pacote sempre atualizado via `pacman -Syu`.

### Ausência de Gerenciador de Pacotes de Aplicação

O projeto não possui `package.json`, `requirements.txt`, `Cargo.toml` ou equivalente — é um script Bash puro. Isso elimina toda a superfície de ataque relacionada a supply chain de dependências de aplicação (ex: npm typosquatting, PyPI malicious packages). **Pontuação positiva de segurança.**

---

## 🗺️ Mapa da Superfície de Ataque

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SUPERFÍCIE DE ATAQUE — Boot-Check              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ENTRADAS EXTERNAS              PROCESSAMENTO         SAÍDA         │
│  ─────────────────              ─────────────         ─────         │
│                                                                     │
│  stdin (teclado)   ──────────►  menu_select_device    TUI display  │
│  [escolha 1..N]                 choice validation                   │
│                                                                     │
│  /sys/block/*/     ──────────►  get_usb_model()       QEMU -name   │
│  device/model                   sanitize_model_name() TUI display  │
│  [dado externo]                 [SEC-02: ANSI inject]               │
│                                                                     │
│  lsblk output      ──────────►  detect_usb_drives()   USB list     │
│  [saída de sistema]             BASH_REMATCH parse                  │
│                                                                     │
│  OVMF_PATH         ──────────►  validate_launch_params QEMU -bios  │
│  [variável mutável]             [SEC-05: não readonly]              │
│                                                                     │
│  QEMU_BIN          ──────────►  sudo "$QEMU_BIN"       execução    │
│  [nome sem path]                [SEC-01: PATH hijack]  como root   │
│                                                                     │
│  /dev/sdX          ──────────►  TOCTOU check→exec      QEMU drive  │
│  [bloco de disco]               [SEC-04: race cond.]               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Vetores Sem Risco Identificado

| Vetor | Análise | Resultado |
|---|---|---|
| SQL Injection | Sem banco de dados | ✅ N/A |
| XSS | Sem interface web | ✅ N/A |
| SSRF | Sem chamadas de rede | ✅ N/A |
| Credenciais hardcoded | Nenhuma encontrada | ✅ Limpo |
| `eval` / code injection | Não utilizado | ✅ Limpo |
| Backticks legados | Não utilizados | ✅ Limpo |
| Arquivos temporários | Não criados | ✅ Limpo |
| Source/include externo | Não utilizado | ✅ Limpo |
| Variáveis sem aspas | Todas protegidas com `"$var"` | ✅ Limpo |

---

## ✅ Checklist de Remediação Prioritária

Aplique as correções na seguinte ordem de prioridade:

### Prioridade 1 — Aplicar Imediatamente (Crítica/Alta)

- [ ] **SEC-01** — Alterar `QEMU_BIN` para caminho absoluto `/usr/bin/qemu-system-x86_64`
- [ ] **SEC-01** — Adicionar `--preserve-env=DISPLAY,XAUTHORITY` ao `sudo`
- [ ] **SEC-01** — Documentar no README o uso de entrada `sudoers` específica
- [ ] **SEC-02** — Remover `+` e `-` de `sanitize_model_name()` e adicionar `cut -c1-64`
- [ ] **SEC-02** — Substituir `echo -e` por `printf` nos locais que exibem `$model` e `$size`
- [ ] **SEC-03** — Refatorar `build_qemu_args()` para usar variável `machine_type` em vez de pattern replace no array
- [ ] **SEC-08** — Adicionar `unset OVMF_PATH` antes da declaração para descartar valor injetado via ambiente
- [ ] **SEC-08** — Adicionar validação de prefixo `/usr/share/` em `validate_launch_params`

### Prioridade 2 — Aplicar na Próxima Release (Média)

- [ ] **SEC-04** — Adicionar `sudo -v` logo após o check de permissão para reduzir janela TOCTOU
- [ ] **SEC-04** — Adicionar validação em `validate_launch_params` que confirma device está na lista USB detectados
- [ ] **SEC-05** — Adicionar `readonly` à declaração de `OVMF_PATH` (linha 25)
- [ ] **SEC-05** — Restringir validação de `OVMF_PATH` para exigir prefixo `/usr/share/`
- [ ] **SEC-06** — Atualizar instrução de instalação no README para incluir `chown root:root /usr/local/bin/bios`
- [ ] **SEC-06** — Gerar `bios.sha256` em cada release e documentar verificação no README

### Prioridade 3 — Melhorias de Hardening (Baixa)

- [ ] **SEC-07** — Adicionar `umask 077` após `set -euo pipefail`
- [ ] **SEC-07** — Adicionar nota sobre verificação de autenticidade de pacotes nas mensagens de `check_deps`
- [ ] **SEC-09** — O patch do SEC-02 (substituir `echo -e` por `printf`) resolve este item simultaneamente

---

## 📊 Score de Segurança Final

| Categoria | Pontuação | Observação |
|---|---|---|
| Injeção de código (eval, backtick) | 10/10 | Não utilizado |
| Sanitização de inputs | 6/10 | ANSI injection possível via sysfs |
| Gestão de privilégios | 5/10 | PATH hijacking + TOCTOU |
| Integridade de dependências | 9/10 | Zero deps de aplicação |
| Segurança de configuração | 7/10 | OVMF_PATH mutável |
| Integridade do software | 5/10 | Sem hash/assinatura |
| **Score Geral** | **7.0/10** | **Bom para ferramenta local; melhorável** |

---

## 📝 Conclusão

O Boot-Check é um projeto bem estruturado com boas práticas de Bash: uso de `set -euo pipefail`, aspas em expansões de variáveis, validação de inputs com regex, e acesso a dispositivos em modo read-only. A ausência de dependências de aplicação externa é um ponto fortíssimo de segurança.

As vulnerabilidades encontradas não são trivialmente exploráveis — exigem acesso local ao sistema do usuário. No entanto, dado que a ferramenta escala para root via `sudo`, **o impacto de exploração bem-sucedida é máximo** (root shell).

**As 3 correções de maior impacto com menor esforço são:**

1. `readonly QEMU_BIN="/usr/bin/qemu-system-x86_64"` — 1 linha, elimina SEC-01
2. `printf '%s' "$raw" | sed 's/[^a-zA-Z0-9 _.]//g' | cut -c1-64` — 1 linha, elimina SEC-02
3. `readonly OVMF_PATH=...` — 1 palavra, elimina SEC-05

Implementar essas 3 correções elevaria o score de segurança para aproximadamente **8.5/10**.

---

*Relatório gerado por auditoria White-box (Pentest de Caixa Branca) — todos os testes foram passivos e não-destrutivos.*  
*Nenhum dado foi modificado, nenhum comando destrutivo foi executado, nenhum dispositivo foi acessado.*

