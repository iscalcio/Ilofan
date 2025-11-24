sudo bash -c 'cat > /usr/local/bin/ilofan << "EOF"
#!/bin/bash
# ilofan – Full fan control for HP DL380 Gen8
# ilofan – Controle total dos fans do DL380 Gen8
#
# Usage / Uso:
#   ilofan 15                → set fans to 15% right now / define fans em 15% agora
#   ilofan 22:30 10          → schedule 10% every day at 22:30 / agenda 10% todo dia às 22:30
#   ilofan auto              → return to iLO automatic mode / volta pro modo automático do iLO

# =================================== CONFIGURATION ===================================
LXC_IP="192.168.15.202"   # ← CHANGE IF YOUR LXC HAS A DIFFERENT IP
                          # ← MUDE SE O IP DO SEU LXC FOR DIFERENTE
PORTA="8000"
# ====================================================================================

# Function to apply speed immediately / Função para aplicar velocidade agora
apply_now() {
    local speed=$1
    curl -s -X POST http://$LXC_IP:$PORTA/index.php \
         -H "Content-Type: application/json" \
         -d "{\"action\":\"fans\",\"fans\":$speed}" >/dev/null
    echo "Fans set to $speed% now / Fans definidos em $speed% agora"
}

# Function to return to iLO auto / Função para voltar ao automático do iLO
apply_auto() {
    curl -s -X POST http://$LXC_IP:$PORTA/index.php \
         -H "Content-Type: application/json" \
         -d "{\"action\":\"auto\"}" >/dev/null
    echo "Fans returned to iLO AUTOMATIC mode / Fans voltaram ao modo AUTOMÁTICO do iLO"
}

# ========== APPLY NOW (only one argument) / APLICA AGORA (apenas um argumento) ==========
if [ $# -eq 1 ]; then
    case "$1" in
        auto)
            apply_auto
            ;;
        [0-9]*)
            if [ "$1" -ge 10 ] && [ "$1" -le 100 ]; then
                apply_now "$1"
            else
                echo "Speed must be between 10 and 100 / Velocidade deve ser entre 10 e 100"
                exit 1
            fi
            ;;
        *)
            echo "Usage: ilofan [10-100] or ilofan auto / Uso: ilofan [10-100] ou ilofan auto"
            exit 1
            ;;
    esac
    exit 0
fi

# ========== SCHEDULE (two arguments: time speed) / AGENDAMENTO (dois argumentos) ==========
if [ $# -eq 2 ]; then
    TIME="$1"
    SPEED="$2"

    # Validate speed / Valida velocidade
    if ! [[ $SPEED =~ ^[0-9]+$ ]] || [ "$SPEED" -lt 10 ] || [ "$SPEED" -gt 100 ]; then
        echo "Speed must be 10–100 / Velocidade deve ser 10–100"
        exit 1
    fi

    # Convert HH:MM to cron format / Converte HH:MM para formato cron
    MIN=$(echo "$TIME" | cut -d: -f2)
    HOUR=$(echo "$TIME" | cut -d: -f1)

    # Remove old entry with same time (prevents duplicates) / Remove entrada antiga com mesmo horário
    (crontab -l 2>/dev/null | grep -v "ilofan $TIME ") | crontab -
    
    # Add new schedule / Adiciona novo agendamento
    (crontab -l 2>/dev/null ; echo "$MIN $HOUR * * * /usr/local/bin/ilofan $SPEED >/dev/null 2>&1") | crontab -

    echo "Scheduled: every day at $TIME → fans $SPEED% / Agendado: todo dia às $TIME → fans $SPEED%"
    exit 0
fi

# ========== HELP / AJUDA ==========
echo "Usage / Uso:"
echo "  ilofan 15                → set now / define agora"
echo "  ilofan 22:30 10          → schedule every day / agenda todo dia"
echo "  ilofan auto              → iLO automatic / automático do iLO"
EOF

# Give execution permission / Dá permissão de execução
chmod +x /usr/local/bin/ilofan

# Final message / Mensagem final
echo ""
echo "✓ ilofan installed successfully! / ilofan instalado com sucesso!"
echo ""
echo "Examples now / Exemplos agora:"
echo "  ilofan 10"
echo "  ilofan 22:00 10"
echo "  ilofan 08:00 25"
echo "  ilofan auto"
echo ""
ilofan 10   # leaves the server quiet right after installation / deixa silencioso logo de cara
'
