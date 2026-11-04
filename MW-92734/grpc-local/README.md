# MW-92734 — testes live do gRPC local com SQL Server TRUNK

## Objetivo e escopo

Validar, antes do merge do MR !488, a persistência da extensão fiscal do UpdateContract Legacy através do **serviço gRPC local real**, conectado ao SQL Server TRUNK. Não foram chamadas diretas ao handler nem teste do serviço publicado.

- HEAD: `a93616df7d369ee1d6546186bfb423edf49bd697`, branch `feature/MW-92734/RT2027_Patch`.
- 896 arquivos versionados da cópia isolada conferidos contra os blobs do HEAD, sem diferenças; hashes dos binários Release no manifesto técnico.
- Versão declarada em project.json: **1.46.0**; SDK observado: **1.46.0.0**. O Grpc.dll do build local sem `/p:Version` conserva metadados `1.0.0.0`/`1.0.0`; não houve retag/rebuild posterior para alterar evidência. HEAD e hashes identificam o código executado.
- SQL: `DB2K16MESQLDEV / TRUNK_MEProd`; workflow **606660**, produto **159270393**, fornecedor **1519353**, reutilizados da fixture documentada `638299` (`TEST MW89963 regular`), consultada somente para proveniência.
- Serviço local HTTP/2: `http://127.0.0.1:24600`, PID **31540**. Compilação Release, migrations e logging sensível desativados por configuração de processo. Nenhum deploy.
- CALCOFF confirmado: `CalculoFiscalReformaTributaria=N`; exposição: `EnviaCamposReformaTributaria2027=S`, antes/depois. Flags não foram alteradas pelo executor.
- Rodada: 28/10/2026, criação da primeira fixture às **18:18:34 UTC / 15:18:34 -03:00**; auditoria final às **18:20:38 UTC / 15:20:38 -03:00**. Intervalos por caso usam timestamps dos arquivos originais de captura, preservados em JSON próprio.

## Resultados

**11/11 casos aprovados**, sem falhas. Todas as chamadas AddContract/UpdateContract passaram pelo cliente SDK e transporte gRPC até o serviço local, com gravação e releitura SQL em conexão independente. Perfil Legacy (enum protobuf default, omitido no JSON quando zero).

| Caso | Fixture | Resultado e comprovação |
|---|---:|---|
| CBS parcial válido | 638755 | `cbsIncluded=N` aceito; CBS armazenado 0,9 e redução 0 preservados |
| Combinação efetiva inválida | 638755 | redução 17 rejeitada com InvalidArgument; snapshots monitorados idênticos |
| Omissão | 638755 | nota atualizada; CBS, redução, amount e duas bases monofásicas omitidos preservados |
| Zero explícito | 638755 | ValorCBS e duas bases monofásicas passaram de valores positivos para 0 |
| Lote com segundo item inválido | 638755 | InvalidArgument; nota do primeiro item não gravada, snapshots monitorados idênticos |
| IBS UF mono-only, aplicação parcial | 638756 | ad rem armazenado 2 reconhecido sem contraparte regular; aplicação N gravada |
| IBS UF ad rem zero | 638756 | zero aceito e gravado, sem preenchimento artificial de taxa/redução regular |
| IBS municipal mono-only, aplicação parcial | 638757 | indicador IBS compartilhado e ad rem municipal armazenado reconhecidos |
| IBS municipal ad rem zero | 638757 | zero aceito e gravado |
| CBS mono-only, aplicação parcial | 638758 | ad rem CBS armazenado reconhecido sem campos regulares |
| CBS ad rem zero | 638758 | zero aceito e gravado |

Em **todos os 11 casos**, o item sibling omitido e os subitens permaneceram iguais byte a byte na representação SQL capturada. A amostra posterior (após pausa de 1,5 s e releitura) ficou igual ao snapshot após a resposta. Isso é uma janela observada, não garantia indefinida contra outros escritores.

As fixtures foram criadas pelo AddContract local, com títulos `TEST MW92734 ...` e marcadores exclusivos registrados em `evidencias/fixtures.json`. **Não houve seed SQL direto, DDL, alteração de flags, remoção de filas ou modificação de contratos de negócio.** As fixtures ficam disponíveis para auditoria; não houve limpeza destrutiva.

## SQL, consolidação e limites

- Antes/depois/depois-observação incluem header, todos os campos dos dois itens, subitem, logs, histórico de tipo Contrato e consolidação. Auditoria final adicional inclui contagens de históricos Contrato/ItemContrato. Não se afirma monitoramento global de todos os objetos da base.
- Rejeições demonstram ausência de alterações nas linhas monitoradas antes/depois; **não houve injeção de falha durante save nem prova de rollback global**.
- O código executado chama `ConsolidaProdutoContrato` no fluxo real; respostas OK cobrem o término desse caminho. Foram observadas **zero linhas em ConsolidacaoContrato** para as fixtures (Status do contrato nulo). Isso **não comprova o resultado agregado em contratos elegíveis**, regras internas da SP ou rollback de agregação pós-commit.
- A conta tem `VIEW DEFINITION=0` para a SP; metadados observados estão em `procedure-metadata.json`. Fonte de repositório não foi tomada como prova da definição implantada.
- A consulta final de StreamQueue, limitada aos quatro IDs e à última hora do servidor, encontrou **zero eventos correspondentes**. A consulta está incluída; zero nessa seleção não prova inexistência de qualquer atividade assíncrona fora dela.
- Null/omissão de wrappers protobuf não são intenções distintas de clear; a rodada valida omissão preservando e zero explícito persistindo. Não valida clear nulo, concorrência geral, outras combinações de flags, REST/Internal ou o serviço publicado.
- Somente o processo de serviço iniciado nesta rodada foi encerrado, após término do runner, em **15:20:38 -03:00**. PID31540 encerrado e portas24600/24601 livres conferidos. Nenhum outro serviço/contêiner foi parado.

## Arquivos e reprodução

`evidencias/` contém requests/responses protobuf JSON, SQL snapshots, flags, identidade, schema, fixtures, resultados e intervalos. `logs/` contém build/execução/serviço Release. `proveniencia/` contém HEAD, hashes de fontes/binários e ciclo do processo. `reproducao/` contém runner e consultas sem credenciais; veja REPRODUZIR.md. Não foram incluídos appsettings/configs, connection strings, bin/obj, bancos, tokens ou senhas.

`manifesto-sha256.json` lista SHA256/tamanho dos arquivos (exceto o próprio manifesto). O ZIP tem CRC verificado e SHA256 externo. Originais preservados fora deste pacote.
