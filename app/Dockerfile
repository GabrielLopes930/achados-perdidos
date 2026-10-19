# Imagem base oficial e enxuta do Python 3.12.
# A variante "slim" traz apenas o necessario para executar Python,
# reduzindo o tamanho final da imagem e a superficie de ataque.
FROM python:3.12-slim

# Nao gravar arquivos .pyc e nao usar buffer na saida padrao.
# Sem o buffer, os logs aparecem imediatamente em "docker compose logs".
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Diretorio de trabalho dentro da imagem.
WORKDIR /app

# Copiar e instalar as dependencias ANTES do codigo da aplicacao.
# Assim a camada do "pip install" so e refeita quando o requirements.txt
# muda; alteracoes no app.py reaproveitam o cache (ver Q5 do relatorio).
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Criar o usuario sem privilegios e o diretorio de uploads JA pertencendo
# a ele (RNF04). A ordem importa: o volume nomeado "fotos" herda, na sua
# primeira criacao, o dono deste diretorio. Se ele fosse do root, o
# usuario appuser nao conseguiria gravar as fotos no volume.
RUN useradd --create-home --shell /bin/bash appuser \
    && mkdir -p /app/uploads \
    && chown -R appuser:appuser /app

# Copiar o codigo da aplicacao ja com o dono correto.
COPY --chown=appuser:appuser app.py .

# A partir daqui o container roda sem privilegios de root.
USER appuser

# Documenta a porta em que a aplicacao escuta (porta interna do container).
EXPOSE 5000

# Forma "exec" (com colchetes): o gunicorn vira o processo PID 1 e recebe
# diretamente os sinais do Docker, permitindo encerramento limpo.
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
