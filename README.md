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
