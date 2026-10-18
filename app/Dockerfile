# Imagem oficial enxuta do Python 3.12 (Debian slim)
FROM python:3.12-slim

# Logs sem buffer (aparecem no "docker compose logs") e sem gerar .pyc
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

# Usuário sem privilégios e diretório de uploads pertencente a ele (RNF04).
# Esta camada quase nunca muda, por isso fica antes das dependências.
RUN groupadd --system --gid 10001 app \
 && useradd --system --uid 10001 --gid app --no-create-home --shell /usr/sbin/nologin app \
 && mkdir -p /app/uploads \
 && chown app:app /app/uploads

# Dependências antes do código: só são reinstaladas se o requirements.txt mudar
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Código da aplicação (pertence ao root: o processo pode ler, mas não alterar)
COPY app.py .

USER app

EXPOSE 5000

# Saúde da aplicação, sem curl na imagem slim: usa o próprio Python
HEALTHCHECK --interval=15s --timeout=3s --start-period=10s --retries=3 \
    CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:5000/health', timeout=2)"]

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "--access-logfile", "-", "app:app"]
