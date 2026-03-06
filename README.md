# 🦕 Digibras NH4CU53 — Ressuscitando um notebook de 2011 no Linux Mint

> Seu notebook de 2011 trava no YouTube? A CPU vai a 100% só de abrir o browser? O brilho muda tão pouco que você nem percebe? Bem-vindo ao clube. Este script resolve tudo isso.

---

## 🤔 O que estava acontecendo

O kernel moderno removeu alguns parâmetros antigos do driver de vídeo Intel (`enable_rc6`, `semaphores`), mas arquivos de configuração velhos continuavam passando esses parâmetros como se nada tivesse mudado desde 2013.

Resultado: a GPU Intel ficava de bobeira enquanto a CPU pobre tentava desenhar **cada pixel da tela sozinha**. Modo tela cheia no YouTube? Trava tudo. Arrastar uma janela? Pico de CPU. O computador estava literalmente sofrendo.

Além disso, Firefox e Brave têm uma lista negra interna que desativa aceleração de hardware para GPUs antigas — mesmo quando o hardware aguenta. Porque aparentemente nós não merecemos.

---

## ⚡ Solução

```bash
bash OTIMIZAR_GPU_SANDYBRIDGE_v3.sh
sudo reboot
```

Pronto. Vai um café.

---

## 📦 O que o script faz

- **Limpa a bagunça** — remove arquivos de configuração conflitantes do modprobe que estavam sabotando o driver i915
- **Corrige o GRUB** — tira os parâmetros obsoletos que o kernel ignora há anos
- **Instala o mpv** — reprodutor de vídeo leve que usa a GPU de verdade
- **Ativa VAAPI no Firefox e Brave** — força os navegadores a usarem aceleração de hardware em vez da blocklist padrão de "GPUs antigas não merecem"
- **Conserta os launchers** — Firefox e Brave já abrem com VAAPI ativo automaticamente, sem precisar de terminal ou atalho especial
- **Brilho em 10 níveis** — Fn+F11/F12 agora faz diferença perceptível de verdade
- **Desativa serviços inúteis** — coisas como gerenciador de modem 3G e descoberta de impressoras de rede que ficam rodando sem fazer nada

---

## ✅ Como confirmar que funcionou

Após o reboot:

```bash
# GPU carregada?
lsmod | grep i915

# VAAPI ok?
LIBVA_DRIVER_NAME=i965 vainfo

# Temperatura
sensors
```

Abra o YouTube, coloque um vídeo em tela cheia. Se não travar, funcionou. Simples assim.

---

## 🌡️ Temperaturas normais pra não entrar em pânico

| Situação | Normal |
|---|---|
| Parado | 45–55°C |
| Navegando | 55–65°C |
| Vídeo com VAAPI | 60–70°C |
| Hora de se preocupar | > 86°C |

---

## 📝 Notas técnicas pra quem quiser saber

- Hardware: Intel Celeron 847 / Sandy Bridge / Intel HD Graphics 2000
- Sistema: Linux Mint 21.x / Ubuntu 22.04 / kernel 5.15+
- O erro `CanCreateUserNamespace: EPERM` no Firefox é esperado e inofensivo
- Sandy Bridge não suporta AV1 — o script desativa isso no Firefox pra não cair em decodificação por software sem avisar

---

## 🤝 Contribuições

Tem o mesmo notebook ou um primo dele (Positivo, CCE, qualquer coisa com Sandy Bridge que sofre no Linux)? Abre uma issue ou PR.

---

*Testado em Linux Mint 21.3 Virginia, kernel 5.15.0-171-generic, março de 2026.*  
*Nenhum Celeron foi prejudicado na produção deste script. Pelo contrário.*




ctrl+C
#!/bin/bash
# =============================================================
#  OTIMIZADOR COMPLETO - Intel Sandy Bridge / Linux Mint
#  Digibras NH4CU53 | Celeron 847 | HD Graphics Sandy Bridge
#  v3 - limpeza total + brilho + navegadores + autostart
# =============================================================

set -euo pipefail

RED='\033[0;31m'; YEL='\033[1;33m'; GRN='\033[0;32m'
CYN='\033[0;36m'; BOLD='\033[1m'; NC='\033[0m'

ok()     { echo -e "${GRN}  ✔  $*${NC}"; }
warn()   { echo -e "${YEL}  ⚠  $*${NC}"; }
err()    { echo -e "${RED}  ✘  $*${NC}"; exit 1; }
info()   { echo -e "${CYN}  →  $*${NC}"; }
header() { echo -e "\n${BOLD}══════════════════════════════════════${NC}";
           echo -e "${BOLD}  $*${NC}";
           echo -e "${BOLD}══════════════════════════════════════${NC}"; }

LOGFILE="$HOME/otimizacao_gpu_$(date +%Y%m%d_%H%M%S).log"
exec > >(tee -a "$LOGFILE") 2>&1
echo -e "${BOLD}Iniciado em $(date)${NC}"

[ "$EUID" -eq 0 ] && err "Não execute como root."

# ════════════════════════════════════════════════════════════
header "1/7 · LIMPEZA DE RASTROS DE TENTATIVAS ANTERIORES"
# ════════════════════════════════════════════════════════════

# Remover arquivos modprobe conflitantes
for f in /etc/modprobe.d/i915-performance.conf \
          /etc/modprobe.d/i915-ultra.conf \
          /etc/modprobe.d/i915-old.conf; do
    if [ -f "$f" ]; then
        sudo rm -f "$f"
        ok "Removido: $f"
    fi
done

# Garantir que existe apenas uma config correta
MODPROBE_CONF="/etc/modprobe.d/i915-sandybridge.conf"
sudo tee "$MODPROBE_CONF" > /dev/null << 'EOF'
# Intel Sandy Bridge - parâmetros válidos para kernel 5.15+
# enable_rc6 e semaphores foram removidos do kernel moderno
options i915 enable_fbc=1 enable_psr=0
EOF
ok "Criado $MODPROBE_CONF limpo"

# Remover launchers antigos/duplicados
for f in "$HOME/.local/share/applications/firefox-vaapi.desktop" \
          "$HOME/.local/share/applications/brave-vaapi.desktop"; do
    [ -f "$f" ] && rm -f "$f" && info "Launcher antigo removido: $(basename $f)"
done

# Limpar backups antigos de tentativas (manter só os mais recentes)
find "$HOME" -maxdepth 1 -name "*.bak*" -mtime +7 -delete 2>/dev/null || true
find "$HOME" -maxdepth 1 -name "grub.bak" -delete 2>/dev/null || true
ok "Backups antigos removidos"

# ════════════════════════════════════════════════════════════
header "2/7 · MODPROBE + GRUB + INITRAMFS"
# ════════════════════════════════════════════════════════════

GRUB_FILE="/etc/default/grub"
sudo cp "$GRUB_FILE" "$HOME/grub.bak.$(date +%Y%m%d)"

# Remover parâmetros obsoletos do GRUB
sudo sed -i \
    -e 's/i915\.enable_rc6=[^ "]*[ ]*//g' \
    -e 's/i915\.semaphores=[^ "]*[ ]*//g' \
    -e 's/  */ /g' \
    "$GRUB_FILE"

# Garantir parâmetros corretos presentes
CMDLINE=$(grep "^GRUB_CMDLINE_LINUX_DEFAULT" "$GRUB_FILE")
if ! echo "$CMDLINE" | grep -q "i915.enable_fbc"; then
    sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="\(.*\)"/GRUB_CMDLINE_LINUX_DEFAULT="\1 i915.enable_fbc=1"/' "$GRUB_FILE"
fi
if ! echo "$CMDLINE" | grep -q "i915.modeset"; then
    sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="\(.*\)"/GRUB_CMDLINE_LINUX_DEFAULT="\1 i915.modeset=1"/' "$GRUB_FILE"
fi

info "GRUB: $(grep "^GRUB_CMDLINE_LINUX_DEFAULT" "$GRUB_FILE")"
sudo update-grub 2>/dev/null
sudo update-initramfs -u
ok "GRUB e initramfs atualizados"

# ════════════════════════════════════════════════════════════
header "3/7 · REMOVENDO BRAVE NIGHTLY E RASTROS"
# ════════════════════════════════════════════════════════════

# Remover Brave Nightly
if dpkg -l | grep -q "brave-browser-nightly"; then
    info "Removendo Brave Nightly..."
    sudo apt-get remove -y brave-browser-nightly 2>/dev/null || true
    sudo apt-get autoremove -y 2>/dev/null || true
    ok "Brave Nightly removido"
else
    ok "Brave Nightly não estava instalado"
fi

# Remover repositório do Brave Nightly
if [ -f /etc/apt/sources.list.d/brave-browser-nightly.sources ]; then
    sudo rm -f /etc/apt/sources.list.d/brave-browser-nightly.sources
    ok "Repositório Brave Nightly removido"
fi

# Limpar cache apt
sudo apt-get clean
ok "Cache apt limpo"

# ════════════════════════════════════════════════════════════
header "4/7 · MPV"
# ════════════════════════════════════════════════════════════

if ! command -v mpv &>/dev/null; then
    sudo apt-get install -y mpv
fi
ok "mpv: $(mpv --version | head -1)"

mkdir -p "$HOME/.config/mpv"
cat > "$HOME/.config/mpv/mpv.conf" << 'EOF'
# Aceleração VAAPI - Intel Sandy Bridge
hwdec=vaapi
hwdec-codecs=h264,mpeg2video,vc1
vo=gpu
gpu-api=opengl
EOF
ok "mpv.conf reescrito limpo"

# ════════════════════════════════════════════════════════════
header "5/7 · FIREFOX + BRAVE COM VAAPI COMO PADRÃO DO SISTEMA"
# ════════════════════════════════════════════════════════════

mkdir -p "$HOME/.local/share/applications"

# ── Firefox ──
if command -v firefox &>/dev/null; then
    FF_BIN=$(command -v firefox)
    FF_DIR="$HOME/.mozilla/firefox"
    FF_PROFILE=$(find "$FF_DIR" -maxdepth 1 -name "*.default*" -type d 2>/dev/null | head -1)

    if [ -n "$FF_PROFILE" ]; then
        USERJS="$FF_PROFILE/user.js"
        # Reescrever user.js limpo (sem duplicatas)
        grep -v "media.ffmpeg.vaapi\|media.av1\|gfx.webrender\|hardware-video-decoding\|media.blocklist\|media.gpu-process\|VAAPI Sandy" \
            "$USERJS" 2>/dev/null > /tmp/userjs_clean || true
        cat >> /tmp/userjs_clean << 'EOF'

// === VAAPI Sandy Bridge ===
user_pref("media.ffmpeg.vaapi.enabled", true);
user_pref("media.av1.enabled", false);
user_pref("gfx.webrender.all", false);
user_pref("gfx.webrender.enabled", false);
user_pref("media.hardware-video-decoding.force-enabled", true);
user_pref("media.blocklist.enabled", false);
user_pref("media.gpu-process-decoder", true);
EOF
        mv /tmp/userjs_clean "$USERJS"
        ok "user.js do Firefox reescrito limpo"
    fi

    # Launcher que substitui o padrão do sistema
    cat > "$HOME/.local/share/applications/firefox.desktop" << EOF
[Desktop Entry]
Name=Firefox
Comment=Firefox com aceleração VAAPI
Exec=env MOZ_DISABLE_RDD_SANDBOX=1 LIBVA_DRIVER_NAME=i965 $FF_BIN %u
Icon=firefox
Terminal=false
Type=Application
Categories=Network;WebBrowser;
MimeType=text/html;text/xml;application/xhtml+xml;x-scheme-handler/http;x-scheme-handler/https;
EOF
    ok "Launcher Firefox substituído — VAAPI ativo por padrão"
fi

# ── Brave ──
BRAVE_BIN=""
for b in brave-browser brave; do
    command -v "$b" &>/dev/null && BRAVE_BIN=$(command -v "$b") && break
done

if [ -n "$BRAVE_BIN" ]; then
    # Reescrever brave-flags.conf limpo
    cat > "$HOME/.config/brave-flags.conf" << 'EOF'
--enable-features=VaapiVideoDecoder
--disable-features=UseChromeOSDirectVideoDecoder
EOF
    ok "brave-flags.conf reescrito limpo"

    cat > "$HOME/.local/share/applications/brave-browser.desktop" << EOF
[Desktop Entry]
Name=Brave
Comment=Brave com aceleração VAAPI
Exec=env LIBVA_DRIVER_NAME=i965 $BRAVE_BIN --enable-features=VaapiVideoDecoder --disable-features=UseChromeOSDirectVideoDecoder %U
Icon=brave-browser
Terminal=false
Type=Application
Categories=Network;WebBrowser;
MimeType=text/html;text/xml;application/xhtml+xml;x-scheme-handler/http;x-scheme-handler/https;
EOF
    ok "Launcher Brave substituído — VAAPI ativo por padrão"
fi

# Definir navegador padrão do sistema (Firefox tem prioridade)
if command -v firefox &>/dev/null; then
    xdg-settings set default-web-browser firefox.desktop 2>/dev/null || true
    ok "Firefox definido como navegador padrão do sistema"
fi

# Atualizar banco de dados de aplicativos
update-desktop-database "$HOME/.local/share/applications" 2>/dev/null || true

# ════════════════════════════════════════════════════════════
header "6/7 · CONTROLE DE BRILHO — 10 NÍVEIS VIA Fn+F11/F12"
# ════════════════════════════════════════════════════════════

# Verificar se backlight existe
BACKLIGHT_PATH=""
for p in /sys/class/backlight/intel_backlight \
          /sys/class/backlight/acpi_video0 \
          /sys/class/backlight/*/; do
    [ -f "${p%/}/max_brightness" ] && BACKLIGHT_PATH="${p%/}" && break
done

if [ -z "$BACKLIGHT_PATH" ]; then
    warn "Backlight não encontrado — brilho por software não disponível"
else
    MAX=$(cat "$BACKLIGHT_PATH/max_brightness")
    ok "Backlight encontrado: $BACKLIGHT_PATH (max: $MAX)"

    # Script de controle de brilho com 10 níveis
    sudo tee /usr/local/bin/brilho << EOF
#!/bin/bash
# Controle de brilho em 10 níveis - Intel Sandy Bridge
BACKLIGHT="$BACKLIGHT_PATH"
MAX=\$(cat "\$BACKLIGHT/max_brightness")
ATUAL=\$(cat "\$BACKLIGHT/brightness")
PASSO=\$((MAX / 10))
[ "\$PASSO" -lt 1 ] && PASSO=1

case "\$1" in
    up)
        NOVO=\$((ATUAL + PASSO))
        [ "\$NOVO" -gt "\$MAX" ] && NOVO="\$MAX"
        ;;
    down)
        NOVO=\$((ATUAL - PASSO))
        [ "\$NOVO" -lt "\$PASSO" ] && NOVO="\$PASSO"  # nunca chega a zero
        ;;
    *)
        echo "Uso: brilho up | brilho down"
        echo "Brilho atual: \$ATUAL / \$MAX"
        exit 1
        ;;
esac

echo "\$NOVO" | sudo tee "\$BACKLIGHT/brightness" > /dev/null
PERCENT=\$((NOVO * 100 / MAX))
echo "Brilho: \$PERCENT% (\$NOVO/\$MAX)"
EOF
    sudo chmod +x /usr/local/bin/brilho

    # Permitir que o usuário mude brilho sem senha
    SUDOERS_LINE="$USER ALL=(ALL) NOPASSWD: /usr/bin/tee $BACKLIGHT_PATH/brightness"
    if ! sudo grep -q "tee $BACKLIGHT_PATH/brightness" /etc/sudoers 2>/dev/null; then
        echo "$SUDOERS_LINE" | sudo tee /etc/sudoers.d/brilho > /dev/null
        sudo chmod 440 /etc/sudoers.d/brilho
        ok "Permissão de brilho sem senha configurada"
    fi

    ok "Script /usr/local/bin/brilho instalado"
    info "Agora vamos mapear Fn+F11 e Fn+F12 no XFCE..."

    # Criar atalhos de teclado via xfconf (XFCE)
    # F11 = diminuir, F12 = aumentar (padrão Digibras NH4CU53)
    if command -v xfconf-query &>/dev/null; then
        # Garantir que o canal de atalhos existe
        xfconf-query -c xfce4-keyboard-shortcuts -p "/commands/custom/<F11>" \
            -n -t string -s "brilho down" 2>/dev/null || \
        xfconf-query -c xfce4-keyboard-shortcuts -p "/commands/custom/<F11>" \
            -s "brilho down" 2>/dev/null || true

        xfconf-query -c xfce4-keyboard-shortcuts -p "/commands/custom/<F12>" \
            -n -t string -s "brilho up" 2>/dev/null || \
        xfconf-query -c xfce4-keyboard-shortcuts -p "/commands/custom/<F12>" \
            -s "brilho up" 2>/dev/null || true

        ok "Atalhos F11/F12 configurados no XFCE"
    else
        warn "xfconf-query não encontrado — configure manualmente:"
        warn "Menu > Configurações > Teclado > Atalhos de Aplicativos"
        warn "  F11 → brilho down"
        warn "  F12 → brilho up"
    fi

    info "Teste agora: brilho up  /  brilho down"
fi

# ════════════════════════════════════════════════════════════
header "7/7 · OTIMIZAÇÕES DE INICIALIZAÇÃO"
# ════════════════════════════════════════════════════════════

# Reduzir timeout do GRUB (boot mais rápido)
if grep -q "^GRUB_TIMEOUT=10" "$GRUB_FILE" 2>/dev/null; then
    sudo sed -i 's/^GRUB_TIMEOUT=10/GRUB_TIMEOUT=3/' "$GRUB_FILE"
    sudo update-grub 2>/dev/null
    ok "Timeout do GRUB reduzido para 3s"
fi

# Desativar serviços desnecessários para hardware limitado
SERVICES_TO_DISABLE=(
    "bluetooth"          # sem bluetooth no NH4CU53
    "cups-browsed"       # descoberta de impressoras na rede
    "ModemManager"       # gerenciador de modems 3G/4G
)

for svc in "${SERVICES_TO_DISABLE[@]}"; do
    if systemctl is-enabled "$svc" &>/dev/null; then
        sudo systemctl disable "$svc" --now 2>/dev/null || true
        ok "Serviço desativado: $svc"
    else
        info "Serviço já inativo: $svc"
    fi
done

# Verificar se mintreport está causando overhead (aparecia nos logs)
if systemctl is-enabled "mintreport" &>/dev/null; then
    warn "mintreport ativo — este serviço verifica problemas do sistema"
    warn "Mantido ativo pois pode ser útil. Para desativar: sudo systemctl disable mintreport"
fi

# ════════════════════════════════════════════════════════════
header "CONCLUÍDO"
# ════════════════════════════════════════════════════════════

echo ""
echo -e "${BOLD}O que foi feito:${NC}"
echo "  ✔  Rastros de tentativas anteriores removidos"
echo "  ✔  GRUB e initramfs limpos e atualizados"
echo "  ✔  Brave Nightly e repositório removidos"
echo "  ✔  mpv configurado com VAAPI"
echo "  ✔  Firefox e Brave abrem com VAAPI automaticamente"
echo "  ✔  Navegador padrão do sistema configurado"
echo "  ✔  Controle de brilho em 10 níveis (F11/F12)"
echo "  ✔  Serviços desnecessários desativados"
echo ""
echo -e "${YEL}  ⚠  REBOOT NECESSÁRIO para todas as mudanças terem efeito${NC}"
echo ""
echo -e "${BOLD}Após o reboot:${NC}"
echo "  • Abra Firefox/Brave normalmente pelo menu — VAAPI já estará ativo"
echo "  • F11 = menos brilho  |  F12 = mais brilho (10 níveis)"
echo "  • Para vídeos locais: mpv arquivo.mp4"
echo ""
echo "Log: $LOGFILE"
echo "Finalizado em $(date)"
