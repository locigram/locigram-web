# locigram-web: Full Redesign + K3s Deploy

## What you're building
A redesigned marketing website for Locigram — an open-source, self-hosted memory layer for AI assistants. Think "long-term memory for Claude/GPT/any LLM." It ingests from email, tickets, chat; extracts facts into memory units (locigrams); promotes high-confidence facts into truths via a decay-based scoring engine. MCP + REST APIs. Stack: Hono, TypeScript, Postgres, Qdrant, Docker/K3s.

Repos are public: github.com/locigram/locigram and github.com/locigram/locigram-web

## Design Brief

**Vibe:** Dark, premium, developer-focused. Vercel/Linear/Resend energy. Clean, confident.

**Design tokens:**
- Background: #0a0a0f
- Surface: #111118
- Border: #1e1e2a
- Accent: #7c6dfa (purple)
- Green: #22c55e
- Text: #d4d4e0
- Font: Inter (Google Fonts), monospace for code
- Rounded cards, subtle gradients, SVG icons only, no stock photos

**Page structure (story first, tech second):**
1. **Nav** — "Locigram" logo left, links right: GitHub, Quick Start, Contact
2. **Hero** — Bold tagline (e.g. "Your AI finally remembers."), subhead explaining what it is in one sentence, two CTAs: "Get Started →" (https://github.com/locigram/locigram) and "View on GitHub" (same URL)
3. **Problem** — 3-panel section: "LLMs forget everything" / "RAG is clunky to maintain" / "Your data, someone else's cloud"
4. **How it works** — 3 steps with icons: Ingest → Extract → Recall
5. **Features** — 4–6 cards: MCP-native, Own your data, Truth Engine, Connectors, Postgres+Qdrant, Docker/K3s
6. **Quick Start** — Docker Compose code block (dark terminal style), copy-paste ready
7. **Architecture** — Package breakdown table (@locigram/server, @locigram/pipeline, @locigram/truth, @locigram/vector, @locigram/connector-*)
8. **Footer** — GitHub, hello@locigram.ai, © 2026 Locigram

**Deliverable:** Single `index.html`, all CSS in `<style>` tag, no external JS, responsive/mobile-friendly. Replace the existing index.html completely.

## Other files to create/modify

1. **Dockerfile:**
```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

2. **`.gitignore`:**
```
node_modules/
.DS_Store
.env
```

3. **Remove** `proxy.js` and `serve.py`: `git rm proxy.js serve.py`

4. **`.github/workflows/deploy.yml`** — on push to main:
   - Build Docker image
   - Push to `ghcr.io/locigram/locigram-web:latest` using GITHUB_TOKEN
   - SSH to K3s and rollout restart:
     ```
     sshpass -p "${{ secrets.K3S_SSH_PASS }}" ssh -o StrictHostKeyChecking=no sudobot@10.10.100.80 \
       "sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml kubectl rollout restart deployment/locigram-web -n locigram-web"
     ```

5. **`deploy/k8s.yaml`:**
   - Namespace: `locigram-web`
   - Deployment: name `locigram-web`, 1 replica, image `ghcr.io/locigram/locigram-web:latest`, imagePullPolicy Always, imagePullSecrets: `ghcr-secret`
   - Service: NodePort, port 30320 → containerPort 80

## Steps to execute after writing files

```bash
# 1. Commit and push
git add -A && git commit -m "feat: full redesign + K3s deployment setup" && git push

# 2. Apply K3s manifests
sshpass -p "Tesla1tesla" ssh -o StrictHostKeyChecking=no sudobot@10.10.100.80 \
  "sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml kubectl apply -f -" < deploy/k8s.yaml

# 3. Create GHCR pull secret on K3s
GH_TOKEN=$(gh auth token)
sshpass -p "Tesla1tesla" ssh -o StrictHostKeyChecking=no sudobot@10.10.100.80 \
  "sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io --docker-username=sudobot99 --docker-password=${GH_TOKEN} \
  --namespace=locigram-web --dry-run=client -o yaml | \
  sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml kubectl apply -f -"

# 4. Build and push Docker image
docker build -t ghcr.io/locigram/locigram-web:latest .
docker push ghcr.io/locigram/locigram-web:latest

# 5. Rollout restart
sshpass -p "Tesla1tesla" ssh -o StrictHostKeyChecking=no sudobot@10.10.100.80 \
  "sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml kubectl rollout restart deployment/locigram-web -n locigram-web"

# 6. Verify
sshpass -p "Tesla1tesla" ssh -o StrictHostKeyChecking=no sudobot@10.10.100.80 \
  "sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml kubectl get pods -n locigram-web"
curl -s http://10.10.100.80:30320 | head -5

# 7. Add GitHub secret for CI
gh secret set K3S_SSH_PASS --repo locigram/locigram-web --body "Tesla1tesla"
```

## When done
Run: `openclaw system event --text "Done: locigram-web redesigned and deployed, locigram.ai is live" --mode now`
