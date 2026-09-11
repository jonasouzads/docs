# Docs (Mintlify)

- `docs.json` é a navegação. Página fora dele não é publicada — é assim que os
  relatórios soltos deste diretório convivem com o site.
- `x-mint.href` é obrigatório em operação de API; sem ele a URL sai acentuada.
- O schema oficial do `docs.json` é a fonte confiável para a estrutura de
  navegação — não deduza pelo que já está escrito.
- `openapi.json` é gerado do comportamento real dos controllers do backend,
  não do que a doc promete.
