# Índice de risco climático

Metodologia proposta em `CLAUDE.md` (seções 5 e 5-A); a explicação de como interpretar cada nível de risco no dia a dia está em [Regras de negócio](rules.md#como-interpretar-o-nivel-de-risco). A implementação exata da fórmula (pesos, faixas, constantes reais do código) está em [Serviços internos](../system/services.md#formula-do-indice-de-risco-riskpy).

Resumo:

- Rota dividida em **segmentos** (~10-15 min de trajeto).
- Cada segmento recebe um score 0-100 a partir de probabilidade e intensidade de chuva, proximidade a rio com risco de alagamento, horário (dia/noite) e sensibilidade da carga transportada.
- Faixas: Baixo (0-24) · Médio (25-49) · Alto (50-74) · Crítico (75-100).
- Índice da viagem = pior segmento (máximo), com média e lista de segmentos de risco exibidas como contexto adicional.
- Evolução planejada: neblina, deslizamento, interdição real de via, vento forte, onda de calor.
