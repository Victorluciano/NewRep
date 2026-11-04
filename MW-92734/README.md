# MW-92734 — testes no serviço implantado

## Objetivo e execução

- Endpoint real `https://trunk.grpc.web.mercadoe.com`, prefixo `/contract`; cliente .NET gRPC com TLS validado e Basic Auth obtido de configuração local autorizada, sem credenciais no pacote.
- SDK publicado `ME.Core.Contract 1.46.340-beta`; assembly SDK `1.46.0.0`. A revisão exata do processo remoto não foi introspectada; versão SDK não é prova da imagem implantada.
- Banco SQL Server TRUNK confirmado pelo preflight. Workflow 606660; origem dedicada 638299. CALC `N` confirmado antes/depois, sem alterar flags.
- Novas fixtures exclusivas desta rodada: 638768 (regular), 638769 (IBS UF), 638770 (IBS municipal), 638771 (CBS). Criadas via AddContract; alterações via UpdateContract. SQL usado apenas para SELECT.

## Resultado

- **23/23 verificações UpdateContract PASS**: 19 casos funcionais e 4 restabelecimentos de taxa para controles I/Z.
- CBS parcial preservou taxa/redução armazenadas; estado efetivo inválido rejeitado; omissão preservou valores; zero explícito persistiu.
- Lote inválido rejeitado sem alterações nos snapshots SQL capturados, inclusive primeiro item; não equivale a teste de falha injetada durante commit.
- Mono-only IBS UF/municipal/CBS: aplicação parcial com adrem armazenado e adrem zero explícito aceitos; siblings/subitens preservados.
- CBS e IBS I/Z: taxa positiva herdada rejeitada sem writes observados; taxa/redução zero explícitas aceitas.
- **5/5 verificações readback PASS**, cada uma com GetContractDataById e GetContractItemsDataById: quatro fixtures novas e fixture histórica 638755. Duas linhas de item e um subitem por fixture; valores finais das novas fixtures verificados.
- A fixture 638755 retornou HTTP 200 e dados corretos via .NET. O NOT_FOUND observado no Postman não foi reproduzido; a causa específica no cliente Postman não foi determinada.

## Evidência e limites

- Requests/responses protobuf, SQL antes/depois e observação posterior de 1,5 s, IDs, flags e resultados em JSON. Nenhuma diferença tardia detectada nessa janela; isso não prova ausência de processamento futuro.
- Consolidação com zero linhas nas fixtures: não comprova cálculo agregado nem rollback global; não houve injeção de falha SQL ou de eventos.
- Não foi testado CALCON nesta rodada nem feita consulta de fila. Nenhuma flag WF foi alterada; nenhuma fixture de negócio foi escrita.
- Nenhum servidor local iniciado, deploy, commit ou push; processos clientes encerrados. Fixtures dedicadas preservadas, sem exclusão.
- Reprodução contém referências a arquivos locais de configuração, sem o conteúdo desses arquivos. Reexecutar cria novas fixtures e exige autorização/conectividade/credenciais locais.

## Casos

- `cbs-partial-valid`: PASS (OK); fixture 638768.
- `effective-invalid`: PASS (InvalidArgument); fixture 638768.
- `omitted-preserves`: PASS (OK); fixture 638768.
- `explicit-zero`: PASS (OK); fixture 638768.
- `batch-invalid-no-partial-writes`: PASS (InvalidArgument); fixture 638768.
- `mono-UF-stored-adrem`: PASS (OK); fixture 638769.
- `mono-UF-zero-adrem`: PASS (OK); fixture 638769.
- `mono-Municipal-stored-adrem`: PASS (OK); fixture 638770.
- `mono-Municipal-zero-adrem`: PASS (OK); fixture 638770.
- `mono-CBS-stored-adrem`: PASS (OK); fixture 638771.
- `mono-CBS-zero-adrem`: PASS (OK); fixture 638771.
- `CBS-I-restore`: PASS (OK); fixture 638768.
- `CBS-I-inherited-invalid`: PASS (InvalidArgument); fixture 638768.
- `CBS-I-explicit-zero-valid`: PASS (OK); fixture 638768.
- `CBS-Z-restore`: PASS (OK); fixture 638768.
- `CBS-Z-inherited-invalid`: PASS (InvalidArgument); fixture 638768.
- `CBS-Z-explicit-zero-valid`: PASS (OK); fixture 638768.
- `IBS-I-restore`: PASS (OK); fixture 638768.
- `IBS-I-inherited-invalid`: PASS (InvalidArgument); fixture 638768.
- `IBS-I-explicit-zero-valid`: PASS (OK); fixture 638768.
- `IBS-Z-restore`: PASS (OK); fixture 638768.
- `IBS-Z-inherited-invalid`: PASS (InvalidArgument); fixture 638768.
- `IBS-Z-explicit-zero-valid`: PASS (OK); fixture 638768.
