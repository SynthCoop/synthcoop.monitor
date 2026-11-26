# Coletor de apreensões — drogas sintéticas e precursores (PF, PRF, Receita Federal)

Alimenta o painel “Apreensões de Drogas Sintéticas no Brasil — 2026”. Gera `dados/apreensoes.json` no mesmo formato do painel, mais um CSV e um relatório por execução.

## Modo sem custo (padrão no GitHub)
O agendamento roda `python coleta.py --so-links`, **sem usar nenhuma API**:
1. descobre as notícias e filtra por substância e verbo de apreensão;
2. converte os links do Google Notícias no endereço real da matéria (conversor embutido no script; teste com `python coleta.py --testar-gnews`);
3. **descarta notícias de casos que já estão no painel** (mesma substância, cidade citada e data a até 5 dias de um registro de `dados/apreensoes.json`); o que foi descartado fica em `links/descartados_ja_no_painel.json`;
4. lê um **trecho** de cada matéria;
5. **agrupa** notícias sobre o mesmo caso (um item com várias fontes, a oficial primeiro);
6. para casos novos com uma só fonte, faz uma **busca complementar** por outras matérias sobre o mesmo caso;
7. notícias de casos que já estão no painel viram **fontes extras** desses casos (seção no fim de `links/pendentes.md`);
8. acumula tudo em `links/pendentes.md`.

A ação **“completar fontes do painel”** (`python coleta.py --completar-fontes`) procura 2ª e 3ª fonte para os casos do painel que têm só um link (até 80 por execução) e as acrescenta à mesma lista.

Na segunda-feira, cole `links/pendentes.md` no chat do Claude e peça “extraia as apreensões destes casos e atualize o painel”. Depois, rode a ação “limpar pendentes”. Links já entregues nunca voltam.

### Manter a base sincronizada (importante)
O descarte automático compara com `dados/apreensoes.json`. A cada atualização do painel, o Claude entrega o arquivo novo: substitua `dados/apreensoes.json` no repositório por ele (no GitHub: entre na pasta `dados`, use **Add file → Upload files** e envie o arquivo com o mesmo nome; ele substitui o anterior).

## Modo completo (opcional, usa a API paga)
1. **Descobre** notícias novas: listagem e busca interna da PF, busca da Receita Federal, API de busca da PRF (a página da PRF é montada por JavaScript), Sindireceita (aduana) e imprensa nacional/local via Google Notícias.
2. **Filtra barato**: só segue adiante o que cita uma substância monitorada e um verbo de apreensão (na imprensa, também PF/PRF/Receita). Incinerações e balanços são descartados.
   - **Títulos genéricos** ("PRF apreende drogas em ônibus"): o texto completo é lido e a notícia só fica se citar substância monitorada (até 150 por execução).
   - **Avisos de conferência** na lista de pendentes: "ice" (pode ser haxixe), data do fato incoerente com a publicação (dia da semana errado, data futura ou mais de 7 dias antes).
   - **Fontes**: gov.br/pf, gov.br/prf, gov.br/receitafederal, gov.br/mj (FICCOs), Sindireceita, Agência Gov/EBC, secretarias estaduais de segurança e sindicatos (via Google Notícias com `site:`), imprensa pelo Google Notícias e pelo GDELT (banco aberto de notícias). Fora dos sites de PF/PRF/Receita, a notícia precisa citar um órgão federal.
   - **Confiança** de cada linha: Alta (nota oficial ou 2+ fontes), Média (1 fonte de imprensa), Baixa (a conferir).
   - **Municípios**: nome oficial e código do IBGE (`python coleta.py --padronizar-municipios` gera `dados/municipios_mapa.json`).
   - **Carga retroativa**: aba Actions → “Manutenção da base” → carga-periodo (datas inicial e final).
   - **Substâncias monitoradas**: Ecstasy/MDMA (inclui MD cristal), metanfetamina, anfetamina (rebite), LSD, selos, cetamina, GHB, 2C-B/tusi, catinonas sintéticas, canabinoides sintéticos (droga K/K2/K4/K9 — "equipe K9" de cães não conta), opioides sintéticos (fentanil, nitazenos, tramadol), DMT, benzodiazepínicos, outras sintéticas; precursores: acetato de etila, helional, safrol, efedrina/pseudoefedrina, P2P/BMK/PMK, piperonal, anidrido acético, permanganato de potássio, solventes (acetona/tolueno em volume de carga) e outros.
3. **Lê** apenas as candidatas e **extrai** os campos com a API do Claude (data, cidade/UF, modal, instituição, substâncias, quantidades, presos, armas, dinheiro).
4. **Deduplica**: mesma substância + mesma UF + cidade semelhante + data ±2 dias + quantidade compatível (±20%). A versão oficial prevalece; as demais viram fontes secundárias e completam campos vazios.
5. **Grava** a base, o CSV (`;`, abre direto no Excel) e `relatorios/coleta_AAAA-MM-DD_HHMM.md` (novas, atualizadas, para revisar).

URLs já lidas ficam em `dados/estado.json` e não são lidas de novo. Cada execução cobre desde a anterior, com 3 dias de folga.

## Instalação
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export ANTHROPIC_API_KEY=...      # sem chave: extração heurística, tudo marcado "revisar"
```

## Uso
```bash
python coleta.py --so-links           # sem custo: só a lista de candidatas (links/pendentes.md)
python coleta.py --limpar-pendentes   # arquiva a lista já processada
python coleta.py --desde 2026-01-01   # modo completo: carga retrospectiva do ano
python coleta.py                      # rotina: incremental
python coleta.py --seco -v            # testa sem gravar, com log detalhado
python coleta.py --fontes pf,prf      # só algumas fontes (pf, prf, receita, sindireceita, imprensa)
```
Modelo: `claude-haiku-5-5` (troque com `MODELO_CLAUDE=...`). Só as notícias que passam no filtro geram chamadas.

## Agendamento (seg, qua e sex às 7h)
- **GitHub Actions** (`.github/workflows/coleta.yml`): roda o modo sem custo. Na aba Actions → “Run workflow” há três ações: coletar links, limpar pendentes e carga do ano.
- **cron** (Linux/Mac, horário local de Brasília):
  `0 7 * * 1,3,5 cd /caminho/coletor_apreensoes && .venv/bin/python coleta.py >> coleta.log 2>&1`

## Atualizar o painel
O script não publica no painel. Na segunda-feira, envie `dados/apreensoes.json` ao Claude na conversa do projeto e peça “atualize o painel com este arquivo”. Antes, confira a seção “Para revisar” do relatório.

## Pontos de atenção
- **PRF**: usa a API de busca que alimenta a página. Se ela mudar, o script avisa e tenta o Playwright (opcional). Sem os dois, a PRF fica coberta só pela imprensa.
- **PF**: a listagem do site só volta cerca de três meses; meses anteriores vêm da busca interna por termo.
- **Imprensa**: quando o texto completo não abre, usa título e resumo e marca para revisão.
- Os totais refletem o que os órgãos e a imprensa divulgam: são indicadores, não estatística oficial.


## Laboratórios clandestinos (espaço à parte)

- **Base:** `dados/laboratorios.json` (uma linha por laboratório, local de descarte ou equipamento apreendido) e aba “Laboratórios clandestinos” no painel e na página pública.
- **Tipos** (UNODC, *Guidelines for the Dismantling of Clandestine Laboratories*, 2024, cap. 3–4, + categorias do projeto): Síntese, Precursores, Processamento/corte/embalagem, Comprimidos, Misto, Refino de cocaína, Inalantes, Descarte de resíduos, Equipamento apreendido. Escala: pequeno, médio, industrial.
- **Regra de inclusão:** qualquer força policial ou órgão de justiça (federal, estadual, municipal); não exige quantidade divulgada.
- **Coleta:** fonte `labs` (Google Notícias + GDELT com termos de laboratório). Toda notícia de qualquer fonte que cite laboratório/prensa/descarte com ação policial vai para `links/laboratorios_pendentes.json` e `.md`, com tipo, escala, local, equipamentos, riscos e esfera sugeridos e aviso de caso provavelmente já registrado.
- **Uso:** anexe `links/laboratorios_pendentes.json` no chat e peça “atualize os laboratórios do painel”. A limpeza de pendentes também arquiva a lista de laboratórios.
