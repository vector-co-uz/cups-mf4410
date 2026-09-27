FROM debian:bookworm-slim

# --- Базовые пакеты ---
RUN apt-get update && apt-get install -y --no-install-recommends \
        cups \
        cups-client \
        cups-bsd \
        printer-driver-gutenprint \
        wget \
        tar \
        udev \
        usbutils \
        gnupg2 \
        sudo \
    && rm -rf /var/lib/apt/lists/*

# --- Драйвер Canon UFR II LT — 32-битный, поэтому подключаем multiarch ---
RUN dpkg --add-architecture i386 && \
    apt-get update && apt-get install -y --no-install-recommends \
        libc6:i386 \
        libstdc++6:i386 \
        libxml2:i386 \
        libpng16-16:i386 \
        zlib1g:i386 \
    && rm -rf /var/lib/apt/lists/*

# --- Копируем драйвер, скачанный вручную с сайта Canon ---
# Положите архив (например Linux_UFRII_LT_PrinterDriver_V*_uk_EN.tar.gz)
# в папку ./drivers рядом с Dockerfile перед сборкой.
COPY drivers/*.tar.gz /tmp/canon-driver.tar.gz

RUN mkdir -p /tmp/canon && \
    tar -xzf /tmp/canon-driver.tar.gz -C /tmp/canon --strip-components=1 && \
    DEBDIR=$(find /tmp/canon -type d -iname "*32-bit*Debian*" -o -type d -iname "Debian" | head -n1) && \
    echo "Using deb dir: $DEBDIR" && \
    dpkg -i ${DEBDIR}/cndrvcups-common*.deb || apt-get install -f -y && \
    dpkg -i ${DEBDIR}/cndrvcups-ufr2*.deb || apt-get install -f -y && \
    rm -rf /tmp/canon /tmp/canon-driver.tar.gz

# --- Настройка CUPS для доступа снаружи контейнера ---
RUN sed -i 's/^Listen localhost:631/Listen 0.0.0.0:631/' /etc/cups/cupsd.conf && \
    sed -i 's/Browsing Off/Browsing On/' /etc/cups/cupsd.conf 2>/dev/null || true && \
    printf '<Location />\n  Order allow,deny\n  Allow all\n</Location>\n' >> /etc/cups/cupsd.conf && \
    printf '<Location /admin>\n  Order allow,deny\n  Allow all\n</Location>\n' >> /etc/cups/cupsd.conf && \
    printf '<Location /admin/conf>\n  AuthType Default\n  Require user @SYSTEM\n  Order allow,deny\n  Allow all\n</Location>\n' >> /etc/cups/cupsd.conf

# --- Пользователь-администратор для веб-интерфейса CUPS (localhost:631) ---
RUN useradd -m -G lpadmin -s /bin/bash printadmin && \
    echo "printadmin:printadmin" | chpasswd

EXPOSE 631

# Запуск CUPS на переднем плане
CMD ["/usr/sbin/cupsd", "-f"]
