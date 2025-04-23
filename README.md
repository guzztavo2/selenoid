# Selenoid, Docker & Docker Compose

Este projeto demonstra como instalar e configurar o **Selenoid**, um “hub” de Selenium que executa navegadores em containers Docker, orquestrado pelo **Docker-Compose**. O processo consiste em:

1. Criar uma **rede Docker** dedicada (`selenoid-net`).  
2. Gerar o arquivo de configuração de browsers (`browsers.json`) via **Configuration Manager (CM)**.  
3. Subir os containers de Selenoid, Selenoid-UI e manter logs e vídeos de sessão.  
4. Controlar sessões, timeouts e encerramento com comandos simples.  

---

## 1. Pré-requisitos

- **Docker** instalado (Engine ≥ 20.x) 
- **Docker-Compose** instalado (versão 1.29+ ou Compose V2)

---

## 2. Criar rede Docker dedicada

Para isolar o tráfego entre Selenoid e os containers de browser, criamos uma _user-defined bridge network_:

```bash
docker network create selenoid-net
```
- `selenoid-net` é o nome lógico da rede.
- Isso garante que todos os containers conectados troquem dados diretamente, sem expor portas extras.  

---

## 3. Gerar `browsers.json` com Configuration Manager

O CM (imagem `aerokube/cm`) automatiza o download de imagens de browser e produz o JSON de configuração:

```bash
docker-compose run --rm config-manager
```

No `docker-compose.yml`, o serviço **config-manager** está definido como:

```yaml
config-manager:
  image: aerokube/cm:latest-release
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock     
    - ./config:/etc/selenoid                        
  networks:
    - selenoid-net
  command:                                     
    - "selenoid" "configure" "--browsers" "chrome;firefox" "--last-versions" "2" "--config-dir" "/etc/selenoid" "--force"
```

Ao executar, você verá:

```
✔ Configuration saved to /etc/selenoid/browsers.json
```

E, no host:

```bash
ls config
# → browsers.json
```

---

## 4. Subir Selenoid e Selenoid-UI

Após gerar o JSON, iniciamos o **Selenoid** (servidor de Selenium) e a **UI** (painel gráfico):

```bash
docker-compose up --build -d
```

Trecho relevante do `docker-compose.yml`:

```yaml
selenoid:
  image: aerokube/selenoid:latest
  depends_on:
    - config-manager
  network_mode: bridge                           
  ports:
    - "4444:4444"                                 
  volumes:
    - ./config:/etc/selenoid:ro                   
    - /var/run/docker.sock:/var/run/docker.sock
    - ./logs:/opt/selenoid/logs
    - ./video:/opt/selenoid/video
  command:
    [
      "-conf", "/etc/selenoid/browsers.json",
      "-limit", "10",
      "-timeout", "15m",                          
      "-video-output-dir", "/opt/selenoid/video"
    ]

selenoid-ui:
  image: aerokube/selenoid-ui:latest
  depends_on:
    - selenoid
  network_mode: bridge
  ports:
    - "8080:8080"                                
  command:
    - "--selenoid-uri" "http://localhost:4444"
```

Após alguns segundos:

- Acesse o painel em <http://localhost:8080> para monitorar sessões.  
- Use `curl http://localhost:4444/status` para ver o JSON de status.  

---

## 5. Parar e limpar tudo

Para encerrar os containers e remover volumes anônimos:

```bash
docker-compose down -v
```

Isso para Selenoid, UI e remove rede _ephemeral_ criada pelo Compose, mas **não** remove a rede `selenoid-net` criada manualmente.

---

## 6. Estrutura dos arquivos de configuração

### 6.1 `config/browsers.json`

Define quais navegadores e versões o Selenoid deve usar. Exemplo mínimo:

```json
{
  "chrome": {
    "default": "128.0",
    "versions": {
      "128.0": {
        "image": "selenoid/vnc_chrome:128.0",
        "port": "4444",
        "path": "/"
      }
    }
  },
  "firefox": {
    "default": "125.0",
    "versions": {
      "125.0": {
        "image": "selenoid/vnc_firefox:125.0",
        "port": "4444",
        "path": "/wd/hub"
      }
    }
  }
}
```

- `image`: container Docker com browser + VNC citeturn0search12  
- `path`: endpoint WebDriver (Chrome `/`, Firefox `/wd/hub`)   

### 6.2 `docker-compose.yml`

Controla sessões, limites e bind-mounts:

- **volumes**: monta configurações e logs.  
- **ports**: publica API WebDriver (4444) e UI (8080).  
- **network_mode** ou **networks**: garante comunicação interna.  

---

## 7. Considerações finais

- Cada robô é **um container** isolado, facilitando escalabilidade e debug.  
- Usar `--timeout 15m` evita que sessões sejam finalizadas durante depuração.  
- Habilitar VNC ou gravação de vídeo (`enableVideo`) permite revisitar o que ocorreu na sessão.  

Este setup demonstra como integrar Docker, Docker-Compose, CM e Selenoid para criar um ambiente robusto de automação de testes em massa.
