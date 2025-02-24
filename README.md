# n8n + AI Starter Stack for Asustor NAS (No SSH Required) 🚀

A **production-ready template** to run n8n workflow automation with Ollama (AI), Qdrant (vector DB), and PostgreSQL on your Asustor NAS using Portainer's web interface.

![Asustor NAS Compatible](https://img.shields.io/badge/Asustor-ADM_4.0%2B-blue?logo=asustor&style=flat-square)
![Portainer Required](https://img.shields.io/badge/Requires-Portainer_CE_2.18%2B-13bdfd?logo=portainer&style=flat-square)

## Features ✨
- **100% Web Interface Setup** - No terminal/SSH required
- **Asustor ADM 4.0+ Optimized** - Verified on AS53/54/67 series
- **Secure Defaults** - Pre-configured with encryption & access controls
- **LLM Ready** - Ollama + Llama3 integration out-of-the-box

## Installation Guide 📥

### Prerequisites
1. Asustor NAS with [Portainer CE](https://www.asustor.com/en-gb/online/College_topic?topic=350) installed
2. (Optional) [Cloudflare Tunnel](https://www.asustor.com/en-gb/online/College_topic?topic=349) for secure remote access

### Step 1 - Prepare Environment File
1. In Portainer, create new **Environment Variables**:
```env
# .env file template
POSTGRES_USER=admin
POSTGRES_PASSWORD=your_strong_db_password
POSTGRES_DB=n8n_production

N8N_ENCRYPTION_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
N8N_USER_MANAGEMENT_JWT_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxx
TUNNEL_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxx
```

### Step 2 - Deploy Stack
1. In Portainer, go to **Stacks** → **Add Stack**
2. Name: `n8n-ai-stack`
3. Build Method: **Web editor**
4. Paste this compose file:
```yaml
version: '3.8'

volumes:
  n8n_storage:
  postgres_storage:
  ollama_storage:
  qdrant_storage:

networks:
#  work-net:
#    external: true # Uses an existing network and sets it as static

x-n8n-environment: &service-n8n-environment
  DB_TYPE: postgresdb
  DB_POSTGRESDB_HOST: postgres
  DB_POSTGRESDB_USER: ${POSTGRES_USER}
  DB_POSTGRESDB_PASSWORD: ${POSTGRES_PASSWORD}
  N8N_DIAGNOSTICS_ENABLED: false
  N8N_PERSONALIZATION_ENABLED: false
  N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}
  N8N_USER_MANAGEMENT_JWT_SECRET: ${N8N_USER_MANAGEMENT_JWT_SECRET}
  OLLAMA_HOST: ollama:11434

x-n8n: &service-n8n
  image: n8nio/n8n:latest
  networks: ['work-net']
  environment: &service-n8n-env
    <<: *service-n8n-environment

x-ollama: &service-ollama
  image: ollama/ollama:latest
  container_name: ollama
  networks: ['work-net']
  restart: unless-stopped
  ports:
    - 11434:11434
  volumes:
    - ollama_storage:/root/.ollama

x-init-ollama: &init-ollama
  image: ollama/ollama:latest
  networks: ['work-net']
  container_name: ollama-pull-llama
  volumes:
    - ollama_storage:/root/.ollama
  entrypoint: /bin/sh
  environment:
    - OLLAMA_HOST=ollama:11434
  command:
    - "-c"
    - "sleep 3; ollama pull llama3"

services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    networks: ['work-net']
    restart: unless-stopped
    command: tunnel --url http://n8n:5678 --no-autoupdate run
    environment:
      - TUNNEL_TOKEN=${TUNNEL_TOKEN}

  postgres:
    image: postgres:16-alpine
    hostname: postgres
    networks: ['work-net']
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_storage:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -h localhost -U ${POSTGRES_USER} -d ${POSTGRES_DB}']
      interval: 5s
      timeout: 5s
      retries: 10

  n8n:
    <<: *service-n8n
    hostname: n8n
    container_name: n8n
    restart: unless-stopped
    environment:
      <<: *service-n8n-env
      DB_POSTGRESDB_DATABASE: ${POSTGRES_DB}
      N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS: "true"
    ports:
      - 5678:5678
    volumes:
      - n8n_storage:/home/node/.n8n
      - ./shared:/data/shared
    depends_on:
      postgres:
        condition: service_healthy

  qdrant:
    image: qdrant/qdrant
    hostname: qdrant
    container_name: qdrant
    networks: ['work-net']
    restart: unless-stopped
    ports:
      - 6333:6333
    volumes:
      - qdrant_storage:/qdrant/storage

  ollama-cpu:
    <<: *service-ollama

  ollama-pull-llama-cpu:
    <<: *init-ollama
    depends_on:
      - ollama-cpu
```
5. Enable **Auto-update** (recommended)

### Step 3 - First Run Setup
1. After deployment completes (2-5 mins), access:
   - Local: `http://your-nas-ip:5678`
   - Remote: `https://n8n.your-domain.com` (if using Cloudflare)
2. **First-time login:**
   - Username: `your@email.com`
   - Password: 'whateverpasswordyouwant'

## Updating 🔄
Portainer will auto-update images when "Auto-update" is enabled. To manually update:

1. In Portainer → Stacks → `n8n-ai-stack`
2. Click **Update**
3. Check "Pull latest image"
4. Click **Update the stack**

## Security Checklist 🔒
- [ ] Change default admin credentials
- [ ] Enable Cloudflare Access Policies
- [ ] Rotate encryption keys quarterly
- [ ] Monitor Portainer audit logs

## ☕ Weekend Warrior Appreciation

After spending my **entire Saturday night** and **Sunday** debugging (including multiple Portainer wipes and NAS resets! 😅), this stack finally works seamlessly with Cloudflared. If this setup saves you from similar headaches:

[![Cash App](https://img.shields.io/badge/Support_My_Late_Night_Coding-00C244?style=for-the-badge&logo=cashapp&label=Cash%20App&logoColor=white)](https://cash.app/$JohnEricHuggins89)  
**[→ Direct Cash App Link](https://cash.app/$JohnEricHuggins89)**

Your support helps fund:  
🔥 More Asustor-specific optimizations  
📚 Documentation improvements  
🛠️ Future-proof maintenance (so you don't have to wipe your NAS like I did!)

*"The only thing I wiped more than my NAS this weekend was my brow!"* 💦

---

**Maintained with ❤️ by Eric** 
