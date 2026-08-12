---
name: obsidian-vault-integration
description: "Ingest and sync Obsidian vault notes into agent knowledge graphs for semantic search and context hydration."
---

# Obsidian Vault Integration

Enables agents to automatically ingest, index, and query Obsidian vaults as a persistent knowledge source. Supports real-time sync, semantic search via embeddings, and bidirectional linking preservation.

## Quick quality checklist

- `name` matches folder name exactly (kebab-case)
- All examples are tested and runnable
- Includes both Bash and Node.js examples
- Uses free/public tools first (obsidian local files)
- No secrets, API keys, or personal data in examples

## When to use

- Use case 1: When the user wants to query their personal knowledge base stored in Obsidian
- Use case 2: When building a self-hydrating knowledge graph from markdown notes
- Use case 3: When agents need context from meeting notes, research, or documentation

## Required tools / APIs

- No external API required (reads local markdown files)
- Optional: `sqlite` for local indexing, or connect to Postgres for distributed setups
- Optional: `pgvector` extension for semantic search (PostgreSQL)

Install options:

```bash
# macOS
brew install sqlite

# Ubuntu/Debian
sudo apt-get install -y sqlite3

# For Postgres + vector search
brew install postgresql
psql -c "CREATE EXTENSION vector;"
```

## Skills

### basic_usage

Scan an Obsidian vault and extract all markdown files with frontmatter metadata.

```bash
# List all markdown files in vault
find /path/to/obsidian/vault -name "*.md" -type f | head -20

# Extract frontmatter from a note
grep -A 10 "^---$" /path/to/note.md | head -12

# Get all tags across vault
grep -rh "^tags:" /path/to/vault --include="*.md" | sort -u
```

**Node.js:**

```javascript
import { readdir, readFile } from 'fs/promises';
import { join, dirname } from 'path';

async function scanObsidianVault(vaultPath) {
  const notes = [];
  
  async function scanDir(dir) {
    const entries = await readdir(dir, { withFileTypes: true });
    
    for (const entry of entries) {
      const fullPath = join(dir, entry.name);
      
      if (entry.isDirectory()) {
        // Skip hidden folders
        if (!entry.name.startsWith('.')) {
          await scanDir(fullPath);
        }
      } else if (entry.name.endsWith('.md')) {
        const content = await readFile(fullPath, 'utf-8');
        const relativePath = fullPath.replace(vaultPath + '/', '');
        
        // Extract frontmatter
        const frontmatterMatch = content.match(/^---\n([\s\S]*?)\n---/);
        let frontmatter = {};
        
        if (frontmatterMatch) {
          const fmLines = frontmatterMatch[1].split('\n');
          for (const line of fmLines) {
            const [key, ...valueParts] = line.split(':');
            if (key && valueParts.length) {
              frontmatter[key.trim()] = valueParts.join(':').trim();
            }
          }
        }
        
        // Extract first 500 chars as preview
        const preview = content.replace(/^---\n[\s\S]*?\n---\n/, '').substring(0, 500);
        
        notes.push({
          path: relativePath,
          title: frontmatter.title || entry.name.replace('.md', ''),
          tags: frontmatter.tags ? frontmatter.tags.split(',').map(t => t.trim()) : [],
          created: frontmatter.created || null,
          modified: frontmatter.modified || null,
          preview,
          fullContent: content
        });
      }
    }
  }
  
  await scanDir(vaultPath);
  return notes;
}

// Usage
// scanObsidianVault('/path/to/obsidian/vault').then(console.log);
```

### robust_usage

Production-grade vault sync with change detection and incremental updates.

```bash
# Monitor vault for changes (macOS)
mdfind "kMDItemFSName == '*.md'" -onlyin /path/to/vault -count 0

# Get modification times for sync detection
find /path/to/vault -name "*.md" -newer /tmp/last_sync.txt -exec ls -la {} \;

# Create vault index for fast search
sqlite3 /tmp/vault_index.db <<EOF
CREATE TABLE notes (
  id INTEGER PRIMARY KEY,
  path TEXT UNIQUE,
  title TEXT,
  content TEXT,
  updated_at TIMESTAMP
);
EOF
```

**Node.js:**

```javascript
import { readdir, readFile, stat, writeFile } from 'fs/promises';
import { join, dirname } from 'path';
import { createHash } from 'crypto';

class ObsidianVaultSync {
  constructor(vaultPath, indexFile = '/tmp/vooodoo_vault_index.json') {
    this.vaultPath = vaultPath;
    this.indexFile = indexFile;
    this.index = this.loadIndex();
  }
  
  loadIndex() {
    try {
      const data = JSON.parse(readFile(this.indexFile, 'utf-8'));
      return data.notes || [];
    } catch {
      return [];
    }
  }
  
  saveIndex() {
    writeFile(this.indexFile, JSON.stringify({ 
      notes: this.index,
      lastSync: new Date().toISOString()
    }, null, 2));
  }
  
  async computeHash(content) {
    return createHash('md5').update(content).digest('hex');
  }
  
  async scanForChanges() {
    const changes = { added: [], modified: [], deleted: [] };
    const currentFiles = new Set();
    
    async function scanDir(dir) {
      const entries = await readdir(dir, { withFileTypes: true });
      
      for (const entry of entries) {
        const fullPath = join(dir, entry.name);
        currentFiles.add(fullPath);
        
        if (entry.isDirectory() && !entry.name.startsWith('.')) {
          await scanDir(fullPath);
        } else if (entry.name.endsWith('.md')) {
          const content = await readFile(fullPath, 'utf-8');
          const hash = await this.computeHash(content);
          const relativePath = fullPath.replace(this.vaultPath + '/', '');
          
          const existing = this.index.find(n => n.path === relativePath);
          
          if (!existing) {
            changes.added.push({ path: relativePath, hash, content });
          } else if (existing.hash !== hash) {
            changes.modified.push({ path: relativePath, hash, content });
          }
        }
      }
    }
    
    await scanDir.call(this, this.vaultPath);
    
    // Find deleted files
    for (const note of this.index) {
      if (!currentFiles.has(join(this.vaultPath, note.path))) {
        changes.deleted.push(note.path);
      }
    }
    
    return changes;
  }
  
  async sync() {
    const changes = await this.scanForChanges();
    
    // Process added and modified
    for (const change of [...changes.added, ...changes.modified]) {
      const note = this.parseNote(change.path, change.content);
      const idx = this.index.findIndex(n => n.path === note.path);
      
      if (idx >= 0) {
        this.index[idx] = { ...this.index[idx], ...note, hash: change.hash };
      } else {
        this.index.push({ ...note, hash: change.hash });
      }
    }
    
    // Remove deleted
    for (const deletedPath of changes.deleted) {
      this.index = this.index.filter(n => n.path !== deletedPath);
    }
    
    this.saveIndex();
    return changes;
  }
  
  parseNote(path, content) {
    const frontmatterMatch = content.match(/^---\n([\s\S]*?)\n---/);
    let frontmatter = {};
    
    if (frontmatterMatch) {
      const fmLines = frontmatterMatch[1].split('\n');
      for (const line of fmLines) {
        const [key, ...valueParts] = line.split(':');
        if (key && valueParts.length) {
          frontmatter[key.trim()] = valueParts.join(':').trim();
        }
      }
    }
    
    const body = content.replace(/^---\n[\s\S]*?\n---\n/, '');
    
    return {
      path,
      title: frontmatter.title || path.split('/').pop().replace('.md', ''),
      tags: frontmatter.tags ? frontmatter.tags.split(',').map(t => t.trim()) : [],
      body,
      wordCount: body.split(/\s+/).length
    };
  }
}

// Usage
// const sync = new ObsidianVaultSync('/path/to/vault');
// const changes = await sync.sync();
// console.log('Added:', changes.added.length, 'Modified:', changes.modified.length);
```

### advanced_usage

Integrate with Postgres + pgvector for semantic search across vault notes.

```sql
-- Connect to Postgres and create vector extension
CREATE EXTENSION vector;

-- Create notes table
CREATE TABLE obsidian_notes (
  id SERIAL PRIMARY KEY,
  path TEXT UNIQUE NOT NULL,
  title TEXT NOT NULL,
  content TEXT,
  tags JSONB,
  embedding vector(1536),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Create index for similarity search
CREATE INDEX ON obsidian_notes USING ivfflat (embedding vector_l2_ops);

-- Search for similar notes
SELECT path, title, similarity(embedding, '[0.1, 0.2, ...]'::vector) AS score
FROM obsidian_notes
ORDER BY embedding <-> '[0.1, 0.2, ...]'::vector
LIMIT 10;
```

**Node.js:**

```javascript
import { Client } from 'pg';
import { ObsidianVaultSync } from './obsidian-vault-sync.js';

class ObsidianKnowledgeGraph {
  constructor(dbConfig, vaultPath) {
    this.client = new Client(dbConfig);
    this.sync = new ObsidianVaultSync(vaultPath);
  }
  
  async connect() {
    await this.client.connect();
    await this.client.query(`
      CREATE TABLE IF NOT EXISTS obsidian_notes (
        id SERIAL PRIMARY KEY,
        path TEXT UNIQUE NOT NULL,
        title TEXT NOT NULL,
        content TEXT,
        tags JSONB,
        embedding vector(1536),
        created_at TIMESTAMP DEFAULT NOW(),
        updated_at TIMESTAMP DEFAULT NOW()
      )
    `);
    await this.client.query(`
      CREATE INDEX IF NOT EXISTS idx_notes_embedding 
      ON obsidian_notes USING ivfflat (embedding vector_l2_ops)
    `);
  }
  
  async syncAndIndex() {
    const changes = await this.sync.sync();
    
    for (const change of [...changes.added, ...changes.modified]) {
      const note = this.sync.parseNote(change.path, change.content);
      
      // In production, call embedding API here
      // const embedding = await getEmbedding(note.body);
      const embedding = this.mockEmbedding(); 
      
      await this.client.query(`
        INSERT INTO obsidian_notes (path, title, content, tags, embedding)
        VALUES ($1, $2, $3, $4, $5)
        ON CONFLICT (path) DO UPDATE SET
          title = EXCLUDED.title,
          content = EXCLUDED.content,
          tags = EXCLUDED.tags,
          embedding = EXCLUDED.embedding,
          updated_at = NOW()
      `, [
        note.path,
        note.title,
        note.body,
        JSON.stringify(note.tags),
        embedding
      ]);
    }
    
    for (const deletedPath of changes.deleted) {
      await this.client.query('DELETE FROM obsidian_notes WHERE path = $1', [deletedPath]);
    }
  }
  
  async search(query, limit = 10) {
    // In production, embed the query first
    const queryEmbedding = this.mockEmbedding();
    
    const result = await this.client.query(`
      SELECT path, title, tags,
             1 - (embedding <=> $1::vector) AS similarity
      FROM obsidian_notes
      ORDER BY embedding <-> $1::vector
      LIMIT $2
    `, [queryEmbedding, limit]);
    
    return result.rows;
  }
  
  mockEmbedding() {
    // Replace with actual embedding API call
    return Array(1536).fill(0).map(() => Math.random() * 0.1 - 0.05);
  }
  
  async close() {
    await this.client.end();
  }
}

// Usage
// const kg = new ObsidianKnowledgeGraph(
//   { connectionString: 'postgresql://user:pass@localhost/voodoo' },
//   '/path/to/obsidian/vault'
// );
// await kg.connect();
// await kg.syncAndIndex();
// const results = await kg.search('project decisions');
```

## Output format

Define exactly what the agent should return.

- **Sync result:** Object with `added`, `modified`, `deleted` counts
- **Search result:** Array of notes with `path`, `title`, `similarity` score
- **Note object:** `{ path, title, tags[], body, wordCount }`
- **Error shape:** `{ error: string, solution: string }`

## Rate limits / Best practices

- Scan vault incrementally (only changed files) to avoid full re-index
- Cache embeddings for at least 24 hours to avoid redundant API calls
- Use exponential backoff on database connection errors
- Respect Obsidian lock files (.obsidian/workspace.json etc.)
- Don't index files in `.trash` or `__MACOSX` folders

## Agent prompt

```text
You have obsidian-vault-integration capability. When a user asks to query their notes, search their knowledge base, or sync Obsidian:

1. Check if the vault path is configured (default: ~/.obsidian/vault or user-specified)
2. Use ObsidianVaultSync to detect changes since last sync
3. For new/changed notes, compute embeddings and store in knowledge graph
4. For queries, search the knowledge graph using vector similarity
5. Return relevant notes with paths and excerpts
6. Always prefer incremental sync over full re-index to save time

If no vault path is specified, ask the user for the location.
```

## Troubleshooting

**Error scenario 1:**
- Symptom: "Vault path not found"
- Solution: Verify the path exists and contains .obsidian folder

**Error scenario 2:**
- Symptom: "Vector extension not available"
- Solution: Run `CREATE EXTENSION vector;` in PostgreSQL first

**Error scenario 3:**
- Symptom: "Embedding API rate limit"
- Solution: Implement caching with 24-hour TTL for embeddings

## See also

- [../opencode-prompt-optimizer/SKILL.md](../opencode-prompt-optimizer/SKILL.md) — Optimize prompts before sending to LLMs
- [../vooodoo-os-setup/SKILL.md](../vooodoo-os-setup/SKILL.md) — Set up VOODOO OS architecture

---

## Notes

- Skill file path should be `skills/obsidian-vault-integration/SKILL.md`
- Quote `description` when it includes `:` to avoid YAML parsing issues
- Keep examples copy-paste friendly and verify they run before submitting
- See [CONTRIBUTING.md](CONTRIBUTING.md) for full contribution standards
