## Python + Docker + Qdrant local

### 1) Estrutura do projeto
- repo/
  - docker/
    - docker-compose.yml
  - src/
    - app/
      - repository/
        - qdrant_client.py
        - models.py
        - repo.py
      - embeddings/
        - encoder.py
      - services/
        - ingest.py
        - query.py
      - tests/
        - integration/
          - test_qdrant_integration.py
    - requirements.txt
    - Dockerfile
  - data/
  - README.md

### 2) docker-compose.yml (essencial)
- Serviço qdrant com volume e porta 6333.
- Opcional: qdrant-ui (se desejar).
Exemplo (resumo):
- image: qdrant/qdrant:latest
- ports: "6333:6333"
- volumes: ./qdrant_storage:/qdrant/storage

### 3) Dependências Python principais
- qdrant-client
- numpy
- pytest (e pytest-asyncio se usar async)
- requests or httpx (se necessário)
- sentence-transformers ou transformers (para embeddings locais)
- pydantic (models)

### 4) Camada de repositório (responsabilidades)
- Inicializar cliente Qdrant (host, port, api key opcional).
- Criar/garantir coleção (collection) com dimensão correta e configurações (distance, hnsw config).
- Inserir/upsert vetores + metadados (IDs).
- Buscar por similaridade (top_k, score threshold).
- Buscar por filtros (metadados).
- Deletar/limpar coleção para testes.

Arquitetura mínima de arquivo:
- models.py: Pydantic model para Document {id: str, vector: List[float], metadata: dict, payload fields}.
- qdrant_client.py: função get_client() que lê env vars.
- repo.py: classe QdrantRepository com métodos: create_collection(dim), upsert(docs), search(query_vector, top_k, filter), delete_collection().

### 5) Embeddings (local)
- encoder.py: função embed(texts: List[str]) -> np.ndarray usando sentence-transformers (ex.: "all-MiniLM-L6-v2") ou modelo HuggingFace offline.
- Embeddings normalizados se desejar (L2).

### 6) Serviços de ingest e query
- ingest.py: pipeline que recebe documentos (textos), gera embeddings, cria IDs e chama repo.upsert.
- query.py: recebe texto de consulta, embede, chama repo.search e retorna top resultados com scores e metadados.

### 7) Testes de integração com Qdrant
- Objetivo: testar integração real com Qdrant em container.
- Estratégia: usar pytest, fixtures que:
  - sobem o docker-compose (ou assumem que Qdrant já está rodando localmente).
  - antes do teste: criar coleção de teste com dimensão 384 (ou do modelo de embeddings), garantir empty.
  - inserir 3-10 documentos conhecidos (vetores simples, ex.: vetores unitários ou vetores pequenos predefinidos).
  - executar search e validar retornos (IDs, ordem por similaridade, scores).
  - cleanup: deletar coleção.

Exemplo de teste (resumo):
- fixture qdrant_client -> instancia QdrantRepository apontando para localhost:6333
- test_upsert_and_search:
  - repo.create_collection(dim=4)
  - upsert docs com vetores: [1,0,0,0], [0,1,0,0], [0,0,1,0]
  - busca por [1,0,0,0] top_k=1 -> espera id do primeiro doc, score perto de 1.0
  - repo.delete_collection()

Dicas práticas:
- Para testes determinísticos, use vetores fixos em vez de embeddings reais.
- Ajuste distance metric (Cosine/Euclid) consistente com embeddings; Qdrant usa "Cosine" via cosine similarity.
- Use timeout/retries em testes para esperar Qdrant ficar saudável.
- Use environment variables em CI para alterar host/port.
- Se quiser evitar docker-compose em unit tests, use mocking do qdrant-client; mantenha integração separada.

### 8) Snippets essenciais (resumidos)

- docker-compose.yml (resumo):
  - qdrant: image qdrant/qdrant:latest, ports 6333:6333, volumes ./qdrant_storage:/qdrant/storage

- Criação de coleção (pseudocódigo):
  - client.recreate_collection(name, vectors_config=VectorParams(size=dim, distance=Distance.COSINE))

- Upsert (pseudocódigo):
  - points = [PointStruct(id=..., vector=..., payload=...)]
  - client.upsert(collection_name, points)

- Search (pseudocódigo):
  - client.search(collection_name, query_vector, limit=top_k, with_payload=True)

### 9) Checklist para entrega
- Docker compose funcionando e documentado.
- Repositório com classe QdrantRepository (testeable).
- Embedding wrapper que pode ser trocado por mocks.
- Tests de integração que sobem/verificam coleção, inserem, buscam, limpam.
- README com comandos: docker-compose up, instalar deps, rodar testes.

Se quiser, gero os arquivos concretos (docker-compose.yml, exemplo de QdrantRepository, encoder usando sentence-transformers, e um teste pytest completo). Qual formato prefere: código pronto para colar ou apenas snippets?