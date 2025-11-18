Docker Compose já preparado para produção do n8n, cobrindo melhores práticas, segurança e resiliência. O desenho contempla:

PostgreSQL (banco recomendado para produção). 
Redis para Queue Mode (escala horizontal com workers). 
n8n (main + worker) com variáveis de ambiente essenciais (URLs, segurança, pruning, métricas etc.). 
Traefik com TLS automático (Let’s Encrypt), sem expor a porta 5678 diretamente (boa prática).
Healthchecks e logging, além de opções de pruning de execuções e endpoints de /healthz e /metrics. 
⚠️ Observação sobre binários (arquivos): Em Queue Mode, o n8n não dá suporte a armazenamento de binários em filesystem; para fluxos que precisam persistir binários, use S3 (recurso Enterprise). Se você não usa binários persistentes (ou usa apenas JSON/HTTP/DB), pode seguir este stack sem S3. 


Notas importantes do stack

Postgres 13+ é recomendado para Queue Mode. 
Queue Mode separa o main (UI/API e orquestração) dos workers (execução), usando Redis. 
Em reverse proxy, defina WEBHOOK_URL e N8N_PROXY_HOPS=1 para links/registro de webhooks corretos. 
Métricas & health: o n8n expõe /healthz, /healthz/readiness e /metrics (se N8N_METRICS=true). Proteja /metrics (ex.: IP allow no Traefik). 
Workers + métricas: versões recentes tiveram um bug em que workers não expunham health/metrics em certas versões; se notar, deixe métricas só no n8n (main) até atualizar. 
Pruning: EXECUTIONS_DATA_PRUNE=true e EXECUTIONS_DATA_MAX_AGE/_MAX_COUNT evitam crescimento do banco. 
Chave de criptografia: é obrigatório definir N8N_ENCRYPTION_KEY e usar a mesma em todos os processos (main/worker), especialmente em Queue Mode, para leitura das credenciais. 


Segurança e hardening (essenciais)

Desabilite a API pública se não precisa (N8N_PUBLIC_API_DISABLED=true e N8N_PUBLIC_API_SWAGGERUI_DISABLED=true). 
Cookies seguros (N8N_SECURE_COOKIE=true + TLS) e N8N_SAMESITE_COOKIE=lax. 
Bloqueie acesso a env no Code node (N8N_BLOCK_ENV_ACCESS_IN_NODE=true). 
Bloqueie nós sensíveis (ex.: Execute Command) via NODES_EXCLUDE. 
Reverse proxy corretamente configurado com WEBHOOK_URL e N8N_PROXY_HOPS=1 para links/IPs confiáveis. 
SSO/LDAP/2FA (opcional) — n8n oferece SSO (OIDC/SAML) e MFA para fortalecer o acesso (pode-se ativar depois). 


Resiliência, escala e dados binários

Escala horizontal: aumente n8n-worker (mais réplicas) para paralelismo.
Binários (arquivos):
Em Queue Mode, filesystem não é suportado para persistência; use S3 (Enterprise) quando fluxos exigirem arquivos entre passos/instâncias. 
Se seus fluxos de IA e integrações não dependem de grandes binários, tudo certo com este stack (JSON/HTTP/DB).
Execuções: ative pruning para evitar crescimento do banco. Ajuste EXECUTIONS_DATA_MAX_AGE e EXECUTIONS_DATA_PRUNE_MAX_COUNT ao seu volume. 

Observabilidade e saúde

Health: GET /healthz e /healthz/readiness — use em healthchecks externos e alertas. 
Métricas Prometheus: N8N_METRICS=true expõe /metrics. Proteja via Traefik (whitelist/IP) ou rede interna. 


Backups e recuperação

Banco: snapshot/dump do Postgres (ex.: pg_dump periódico).
n8n (workflows/credenciais): CLI de export (n8n export:workflow --all e n8n export:credentials). Combine com versionamento externo se quiser. 


Boas práticas de backup (múltiplas cópias e testes de restauração). 

IA, banco e integrações (práticas para produção)
Credenciais de IA/Apps/DB: guarde no gerenciador de credenciais do n8n (não em variáveis de ambiente do container). A chave de criptografia garante sigilo das credenciais no banco. 
Webhooks de apps externos: verifique que a URL exibida no editor seja a do domínio HTTPS (graças ao WEBHOOK_URL). 
Limites de payload: se trabalhar com prompts/dados maiores (IA), ajuste N8N_PAYLOAD_SIZE_MAX e N8N_FORMDATA_FILE_SIZE_MAX. 


Teste rápido pós‑deploy

Acesse https://SEU_DOMINIO → crie o usuário proprietário.
/healthz: curl -I https://SEU_DOMINIO/healthz deve retornar 200. 
Crie um workflow simples (ex.: Webhook → Respond to Webhook) e valide o endpoint público mostrado (deve bater com seu domínio). 


Por que essas escolhas

Postgres é o banco recomendado e dá robustez. 
Queue Mode + Redis traz resiliência e escala (workers paralelos). 
Chave de criptografia e variáveis de segurança endurecem o ambiente (cookies seguros, bloqueio de env no Code node, desativação da API pública quando não usada). 
Webhook/Proxy: WEBHOOK_URL e N8N_PROXY_HOPS evitam problemas clássicos de URL incorreta/headers atrás de proxy. 
Pruning & métricas ajudam a manter o desempenho e o banco sob controle. 
