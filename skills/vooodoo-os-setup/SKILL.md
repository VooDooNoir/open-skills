---
name: vooodoo-os-setup
description: "Initialize and configure VOODOO OS architecture with core kernel services, agent registry, and knowledge graph."
---

# VOODOO OS Setup

Complete guide for setting up the VOODOO OS architecture including core kernel services, Docker deployment, and first agent registration.

## Quick quality checklist

- `name` matches folder name exactly (kebab-case)
- All examples are tested and runnable
- Includes both Bash and Node.js examples
- Uses free/public tools (Docker, Postgres, NATS)
- No secrets, API keys, or personal data in examples

## When to use

- Use case 1: When initializing a new VOODOO OS deployment
- Use case 2: When setting up agent infrastructure from scratch
- Use case 3: When integrating Opencode and Obsidian into the stack

## Required tools / APIs

- Docker and Docker Compose
- Node.js 18+ and npm
- PostgreSQL (via Docker)
- Git for version control

Install options:

```bash
# macOS
brew install docker node postgresql

# Ubuntu/Debian
sudo apt-get install -y docker.io docker-compose nodejs postgresql

# Verify installations
docker --version
node --version
psql --version
```

## Skills

### basic_usage

Deploy core kernel services with Docker Compose.

```bash
# Create project directory
mkdir -p ~/voodoo/os-kernel
cd ~/voodoo/os-kernel

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: "3.9"

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: voodoo
      POSTGRES_PASSWORD: voodoo_dev_password
      POSTGRES_DB: voodoo
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U voodoo"]
      interval: 10s
      timeout: 5s
      retries: 5

  nats:
    image: nats:2.10
    ports:
      - "4222:4222"
      - "8222:8222"
    healthcheck:
      test: ["CMD", "nats", "server", "check"]
      interval: 10s
      timeout: 5s
      retries: 5

  keycloak:
    image: quay.io/keycloak/keycloak:24.0.2
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports:
      - "8080:8080"
    command: start-dev
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8080/health/live || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  postgres_data:
EOF

# Start services
docker compose up -d

# Verify all services
echo "=== Service Health Check ==="
curl -s http://localhost:5432 >/dev/null && echo "✓ PostgreSQL" || echo "✗ PostgreSQL"
curl -s http://localhost:4222 >/dev/null && echo "✓ NATS" || echo "✗ NATS"
curl -s http://localhost:8080/health/live >/dev/null && echo "✓ Keycloak" || echo "✗ Keycloak"
```

**Node.js:**

```javascript
import { execSync } from 'child_process';
import { writeFileSync, mkdirSync } from 'fs';
import { join } from 'path';

class VoodooOSSetup {
  static async initialize(projectRoot = '~/voodoo/os-kernel') {
    // Create directories
    const dirs = [
      join(projectRoot, 'src'),
      join(projectRoot, 'src', 'registry'),
      join(projectRoot, 'src', 'mcp'),
      join(projectRoot, 'data')
    ];
    
    for (const dir of dirs) {
      mkdirSync(dir, { recursive: true });
    }
    
    // Create docker-compose.yml
    const composeYml = `version: "3.9"

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: voodoo
      POSTGRES_PASSWORD: voodoo_dev_password
      POSTGRES_DB: voodoo
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  nats:
    image: nats:2.10
    ports:
      - "4222:4222"
      - "8222:8222"

  keycloak:
    image: quay.io/keycloak/keycloak:24.0.2
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports:
      - "8080:8080"
    command: start-dev

volumes:
  postgres_data:
`;
    
    writeFileSync(join(projectRoot, 'docker-compose.yml'), composeYml);
    
    console.log(`✓ Created project structure at ${projectRoot}`);
    console.log('Run: cd ' + projectRoot + ' && docker compose up -d');
  }
}

// Usage
// VoodooOSSetup.initialize().then(() => console.log('Setup complete'));
```

### robust_usage

Production deployment with health checks and monitoring.

```bash
#!/bin/bash
# deploy-production.sh

set -euo pipefail

PROJECT_DIR="${1:-$HOME/voodoo/os-kernel}"
COMPOSE_FILE="$PROJECT_DIR/docker-compose.prod.yml"

echo "🚀 Deploying VOODOO OS to $PROJECT_DIR"

# Create production compose file
cat > "$COMPOSE_FILE" << 'EOF'
version: "3.9"

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-voodoo}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: voodoo
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-voodoo}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  nats:
    image: nats:2.10
    ports:
      - "4222:4222"
      - "8222:8222"
    volumes:
      - nats_data:/data
    restart: unless-stopped

  keycloak:
    image: quay.io/keycloak/keycloak:24.0.2
    environment:
      KEYCLOAK_ADMIN: ${KEYCLOAK_ADMIN:-admin}
      KEYCLOAK_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD}
      KC_HOSTNAME: ${KEYCLOAK_HOSTNAME:-localhost}
      KC_HTTP_PORT: "8080"
    ports:
      - "8080:8080"
    command: start
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

volumes:
  postgres_data:
  nats_data:
EOF

# Check prerequisites
echo "✓ Checking prerequisites..."
command -v docker >/dev/null 2>&1 || { echo "❌ Docker not found"; exit 1; }
command -v docker-compose >/dev/null 2>&1 || { echo "❌ Docker Compose not found"; exit 1; }

# Start services
echo "✓ Starting services..."
cd "$PROJECT_DIR"
docker compose -f "$COMPOSE_FILE" up -d

# Wait for health
echo "⏳ Waiting for services to be healthy..."
sleep 10

# Health checks
echo "✓ Running health checks..."
curl -sf http://localhost:5432 >/dev/null && echo "  ✓ PostgreSQL" || echo "  ✗ PostgreSQL"
curl -sf http://localhost:4222 >/dev/null && echo "  ✓ NATS" || echo "  ✗ NATS"
curl -sf http://localhost:8080/health/live >/dev/null && echo "  ✓ Keycloak" || echo "  ✗ Keycloak"

echo ""
echo "✅ VOODOO OS deployed successfully!"
echo "   PostgreSQL:    localhost:5432"
echo "   NATS:          localhost:4222"
echo "   Keycloak:      http://localhost:8080"
echo ""
echo "   Default Keycloak credentials:"
echo "   Username: admin"
echo "   Password: ${KEYCLOAK_ADMIN_PASSWORD:-admin}"
EOF

chmod +x deploy-production.sh

# Usage
# ./deploy-production.sh ~/voodoo/os-kernel
```

**Node.js:**

```javascript
import { execSync } from 'child_process';
import { writeFileSync, mkdirSync, chmodSync } from 'fs';
import { join } from 'path';

class ProductionDeployer {
  static deploy(projectRoot = process.env.VOOODOO_PROJECT || '~/voodoo/os-kernel') {
    console.log(`🚀 Deploying VOODOO OS to ${projectRoot}`);
    
    // Create directories
    const dirs = [
      join(projectRoot, 'src', 'registry'),
      join(projectRoot, 'src', 'mcp'),
      join(projectRoot, 'src', 'ingestion'),
      join(projectRoot, 'data'),
      join(projectRoot, 'logs')
    ];
    
    for (const dir of dirs) {
      mkdirSync(dir, { recursive: true });
    }
    
    // Create production compose
    const composeYml = `version: "3.9"

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: \${POSTGRES_USER:-voodoo}
      POSTGRES_PASSWORD: \${POSTGRES_PASSWORD}
      POSTGRES_DB: voodoo
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U \${POSTGRES_USER:-voodoo}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  nats:
    image: nats:2.10
    ports:
      - "4222:4222"
      - "8222:8222"
    volumes:
      - nats_data:/data
    restart: unless-stopped

  keycloak:
    image: quay.io/keycloak/keycloak:24.0.2
    environment:
      KEYCLOAK_ADMIN: \${KEYCLOAK_ADMIN:-admin}
      KEYCLOAK_ADMIN_PASSWORD: \${KEYCLOAK_ADMIN_PASSWORD}
    ports:
      - "8080:8080"
    command: start
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

volumes:
  postgres_data:
  nats_data:
`;
    
    writeFileSync(join(projectRoot, 'docker-compose.prod.yml'), composeYml);
    
    // Create .env file template
    const envTemplate = `# VOODOO OS Environment Variables
POSTGRES_USER=voodoo
POSTGRES_PASSWORD=changeme_in_production
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=changeme_in_production
`;
    writeFileSync(join(projectRoot, '.env.template'), envTemplate);
    
    console.log('✅ Production deployment ready');
    console.log('   1. Copy .env.template to .env and set secrets');
    console.log('   2. Run: docker compose -f docker-compose.prod.yml up -d');
  }
}

// Usage
// ProductionDeployer.deploy();
```

### agent_registry_setup

Create the Agent Registry service for managing agent identities and permissions.

```bash
# Create registry service
mkdir -p ~/voodoo/os-kernel/src/registry
cat > ~/voodoo/os-kernel/src/registry/main.py << 'PYEOF'
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import sqlite3
from datetime import datetime

app = FastAPI(title="VOODOO Agent Registry")

DATABASE = "agents.db"

class Agent(BaseModel):
    id: str
    email: str
    display_name: str
    scopes: List[str] = ["read"]
    api_key: Optional[str] = None
    created_at: Optional[str] = None

class AgentCreate(BaseModel):
    email: str
    display_name: str
    scopes: List[str] = ["read"]

def get_db():
    conn = sqlite3.connect(DATABASE)
    conn.row_factory = sqlite3.Row
    return conn

@app.on_event("startup")
def startup():
    conn = get_db()
    conn.execute("""
        CREATE TABLE IF NOT EXISTS agents (
            id TEXT PRIMARY KEY,
            email TEXT UNIQUE NOT NULL,
            display_name TEXT NOT NULL,
            scopes TEXT NOT NULL,
            api_key TEXT,
            created_at TEXT NOT NULL,
            updated_at TEXT NOT NULL
        )
    """)
    conn.commit()
    conn.close()

@app.get("/agents", response_model=List[Agent])
def list_agents():
    conn = get_db()
    agents = conn.execute("SELECT * FROM agents ORDER BY created_at DESC").fetchall()
    conn.close()
    return [dict(a) for a in agents]

@app.post("/agents", response_model=Agent)
def create_agent(agent: AgentCreate):
    import uuid
    agent_id = str(uuid.uuid4())
    api_key = f"voodoo_{uuid.uuid4().hex}"
    now = datetime.utcnow().isoformat()
    
    conn = get_db()
    conn.execute(
        """INSERT INTO agents (id, email, display_name, scopes, api_key, created_at, updated_at)
           VALUES (?, ?, ?, ?, ?, ?, ?)""",
        (agent_id, agent.email, agent.display_name, 
         str(agent.scopes), api_key, now, now)
    )
    conn.commit()
    conn.close()
    
    return Agent(
        id=agent_id,
        email=agent.email,
        display_name=agent.display_name,
        scopes=agent.scopes,
        api_key=api_key,
        created_at=now
    )

@app.get("/agents/{agent_id}")
def get_agent(agent_id: str):
    conn = get_db()
    agent = conn.execute("SELECT * FROM agents WHERE id = ?", (agent_id,)).fetchone()
    conn.close()
    
    if not agent:
        raise HTTPException(status_code=404, detail="Agent not found")
    
    return dict(agent)

@app.delete("/agents/{agent_id}")
def delete_agent(agent_id: str):
    conn = get_db()
    cursor = conn.execute("DELETE FROM agents WHERE id = ?", (agent_id,))
    conn.commit()
    conn.close()
    
    if cursor.rowcount == 0:
        raise HTTPException(status_code=404, detail="Agent not found")
    
    return {"deleted": agent_id}
PYEOF

# Install dependencies
pip install fastapi uvicorn pydantic

# Start registry service
uvicorn src.registry.main:app --host 0.0.0.0 --port 9000

# Test the registry
curl -X POST http://localhost:9000/agents \
  -H "Content-Type: application/json" \
  -d '{"email": "alice@voodoo.ai", "display_name": "Alice", "scopes": ["read", "write"]}'

curl http://localhost:9000/agents
```

### knowledge_graph_setup

Initialize the knowledge graph schema in Postgres.

```bash
# Connect to Postgres and set up schema
psql -h localhost -U voodoo -d voodoo << 'SQL'
-- Enable vector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Events table for knowledge graph
CREATE TABLE IF NOT EXISTS events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type VARCHAR(50) NOT NULL,
    source VARCHAR(100) NOT NULL,
    agent_id VARCHAR(255),
    content JSONB NOT NULL,
    embedding vector(1536),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Create index for vector similarity search
CREATE INDEX IF NOT EXISTS idx_events_embedding 
ON events USING ivfflat (embedding vector_l2_ops)
WITH (lists = 100);

-- Create GIN index for JSONB queries
CREATE INDEX IF NOT EXISTS idx_events_content 
ON events USING gin (content);

-- View recent events
SELECT type, source, content->>'title' as title, created_at
FROM events
ORDER BY created_at DESC
LIMIT 10;
SQL

echo "✓ Knowledge graph schema initialized"
```

**Node.js:**

```javascript
import { Client } from 'pg';

class KnowledgeGraph {
  constructor(config) {
    this.client = new Client(config);
  }
  
  async connect() {
    await this.client.connect();
    await this.initializeSchema();
  }
  
  async initializeSchema() {
    await this.client.query(`
      CREATE EXTENSION IF NOT EXISTS vector;
      
      CREATE TABLE IF NOT EXISTS events (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        type VARCHAR(50) NOT NULL,
        source VARCHAR(100) NOT NULL,
        agent_id VARCHAR(255),
        content JSONB NOT NULL,
        embedding vector(1536),
        created_at TIMESTAMP DEFAULT NOW(),
        updated_at TIMESTAMP DEFAULT NOW()
      );
      
      CREATE INDEX IF NOT EXISTS idx_events_embedding 
      ON events USING ivfflat (embedding vector_l2_ops)
      WITH (lists = 100);
      
      CREATE INDEX IF NOT EXISTS idx_events_content 
      ON events USING gin (content);
    `);
  }
  
  async storeEvent(type, source, content, agentId = null, embedding = null) {
    return this.client.query(
      `INSERT INTO events (type, source, agent_id, content, embedding)
       VALUES ($1, $2, $3, $4, $5)
       RETURNING id, created_at`,
      [type, source, agentId, JSON.stringify(content), embedding]
    );
  }
  
  async searchSimilar(queryEmbedding, limit = 10) {
    return this.client.query(
      `SELECT id, type, source, content, 
              1 - (embedding <-> $1::vector) AS similarity
       FROM events
       WHERE embedding IS NOT NULL
       ORDER BY embedding <-> $1::vector
       LIMIT $2`,
      [queryEmbedding, limit]
    );
  }
  
  async getEventsByType(type, limit = 20) {
    return this.client.query(
      `SELECT * FROM events 
       WHERE type = $1 
       ORDER BY created_at DESC 
       LIMIT $2`,
      [type, limit]
    );
  }
  
  async close() {
    await this.client.end();
  }
}

// Usage
// const kg = new KnowledgeGraph({ connectionString: 'postgresql://voodoo:voodoo_dev_password@localhost:5432/voodoo' });
// await kg.connect();
// await kg.storeEvent('slack_msg', 'slack', { text: 'Hello world' });
// const results = await kg.searchSimilar([...embedding...]);
```

## Output format

Define exactly what the agent should return.

- **Setup complete:** Object with `{ services: [...], endpoints: {...}, next_steps: [...] }`
- **Agent created:** `{ id, email, display_name, api_key, scopes }`
- **Event stored:** `{ id, created_at }`
- **Error shape:** `{ error: string, details: string, solution: string }`

## Rate limits / Best practices

- Use connection pooling for Postgres (max 20 connections)
- Implement exponential backoff on database errors
- Cache agent registry lookups for 5 minutes
- Don't store raw embeddings in logs
- Use prepared statements to prevent SQL injection

## Agent prompt

```text
You have vooodoo-os-setup capability. When initializing or deploying VOODOO OS:

1. Check if services are already running (docker compose ps)
2. Deploy core kernel services (Postgres, NATS, Keycloak)
3. Initialize Agent Registry and Knowledge Graph schemas
4. Verify all health endpoints are responding
5. Provide next steps for agent registration and skill loading

Always use environment variables for secrets, never hardcode passwords.
```

## Troubleshooting

**Error scenario 1:**
- Symptom: "Port 5432 already in use"
- Solution: Stop existing PostgreSQL instance or change port in docker-compose.yml

**Error scenario 2:**
- Symptom: "Connection refused to NATS"
- Solution: Check if NATS container is running with `docker compose ps`

**Error scenario 3:**
- Symptom: "Keycloak admin login failed"
- Solution: Verify KEYCLOAK_ADMIN_PASSWORD environment variable is set correctly

## See also

- [../obsidian-vault-integration/SKILL.md](../obsidian-vault-integration/SKILL.md) — Add Obsidian as knowledge source
- [../opencode-prompt-optimizer/SKILL.md](../opencode-prompt-optimizer/SKILL.md) — Integrate prompt optimization

---

## Notes

- Skill file path should be `skills/vooodoo-os-setup/SKILL.md`
- Quote `description` when it includes `:` to avoid YAML parsing issues
- Keep examples copy-paste friendly and verify they run before submitting
- See [CONTRIBUTING.md](CONTRIBUTING.md) for full contribution standards
