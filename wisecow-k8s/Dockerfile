FROM ubuntu:22.04

RUN apt-get update && \
    apt-get install -y cowsay fortune-mod netcat-openbsd && \
    rm -rf /var/lib/apt/lists/*

ENV PATH="/usr/games:${PATH}"

WORKDIR /app
COPY . .

EXPOSE 4499

CMD ["bash", "wisecow.sh"]
