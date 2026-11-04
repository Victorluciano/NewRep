# MW-92734 — Roteiro Postman (gRPC Contract, TRUNK, CALCOFF)

Reexecução manual dos 23 casos de `UpdateContract` e das 5 releituras do teste no serviço implantado.

> **Pendente:** as mensagens JSON de cada caso estão em `evidence-calcoff/*-request.json`, dentro de `MW-92734-deployed-CALCOFF.zip`. Esse ZIP ainda não foi recebido.
> Até lá, cada passo traz só a **intenção** da alteração, tirada do resumo do ticket. Monte a mensagem com **Use Example Message** no Postman ou cole o JSON do ZIP.

## 1. Preparação (uma vez)

1. **Environment:** selecione `TRUNK — MW-92734 (gRPC Contract)` (já criado no Personal Workspace).
2. **Vault:** crie as chaves `grpc_contract_user` e `grpc_contract_password` com o Basic Auth do gRPC TRUNK.
3. **Coleção gRPC:** use uma coleção gRPC (ex.: "New Collection") e crie uma requisição gRPC:
   - URL: `{{grpc_url}}`, com o cadeado TLS **ligado**.
   - **Service definition:** Server reflection; se não houver, importe o `.proto` do `contract.Contract`.
   - **Authorization:** Basic Auth, com `{{grpc_user}}` / `{{grpc_password}}`.
   - Salve como modelo e **duplique** para cada passo abaixo, trocando o método e a mensagem.
4. **Pré-condição:** CALCOFF ligado. Confirme, por SELECT, `CalculoFiscalReformaTributaria = N` e `EnviaCamposReformaTributaria2027 = S` no workflow 606660. Não altere flags.

## 2. Scripts (aba *Scripts → After response*)

**Caso que espera OK (status gRPC 0):**

```js
pm.test("gRPC OK", () => pm.response.to.have.statusCode(0));
```

**Caso que espera InvalidArgument (status gRPC 3):**

```js
pm.test("gRPC INVALID_ARGUMENT", () => pm.response.to.have.statusCode(3));
```

**AddContract (só se `criar_fixtures = true`):** guarda o ID retornado. Troque `fixture_regular` pela variável da fixture daquele passo.

```js
pm.test("gRPC OK", () => pm.response.to.have.statusCode(0));
const msg = pm.response.messages.idx(0).data || {};
const id = msg.id ?? msg.contractId ?? msg.Id ?? msg.ContractId;
pm.test("ID retornado", () => pm.expect(id, JSON.stringify(msg)).to.exist);
if (id) pm.environment.set("fixture_regular", String(id));
```

> O nome do campo de ID na resposta não foi confirmado. Ajuste o script depois da primeira execução.

## 3. Ordem de execução

**Execute na ordem.** Os casos 12–23 dependem do estado deixado pelos anteriores. Os passos de *restore* religam o CBS/IBS regular antes de cada controle I/Z.

### 3.0 Fixtures

| Passo | Método | Opção A: reaproveitar (padrão) | Opção B: `criar_fixtures = true` |
|---|---|---|---|
| F1 | `AddContract` | usar `{{fixture_regular}}` = 638768 | seed `regular-seed-request.json` |
| F2 | `AddContract` | usar `{{fixture_ibs_uf}}` = 638769 | seed `mono-UF-seed-request.json` |
| F3 | `AddContract` | usar `{{fixture_ibs_municipal}}` = 638770 | seed `mono-Municipal-seed-request.json` |
| F4 | `AddContract` | usar `{{fixture_cbs}}` = 638771 | seed `mono-CBS-seed-request.json` |

**Opção A:** as fixtures já estão no estado **final** da última rodada, com zeros e I/Z aplicados. Os casos 1–11 partem de outro estado. Por isso, rodar a Opção A **não reproduz** os casos 1–11 com o mesmo ponto de partida. Para uma reexecução fiel, use a Opção B.

### 3.1 UpdateContract — fixture regular (`{{fixture_regular}}`)

| # | Caso (arquivo do ZIP) | Intenção | Esperado |
|---:|---|---|:---:|
| 1 | `cbs-partial-valid` | Só a aplicação CBS muda para `N`. Alíquota 0,9 e redução 0 armazenadas devem ser preservadas | OK (0) |
| 2 | `effective-invalid` | Redução CBS que torna o estado efetivo inválido (redução 17 na rodada local) | INVALID_ARGUMENT (3) |
| 3 | `omitted-preserves` | Altera campo não fiscal e omite os campos fiscais. Taxas, reduções, valores e quantidades mono preservados | OK (0) |
| 4 | `explicit-zero` | Valor CBS e as duas quantidades monofásicas = 0 explícito | OK (0) |
| 5 | `batch-invalid-no-partial-writes` | Lote com 2 itens: o 1º é válido e o 2º inválido. Nada do 1º pode ser gravado | INVALID_ARGUMENT (3) |

### 3.2 Mono-only

| # | Caso | Fixture | Intenção | Esperado |
|---:|---|---|---|:---:|
| 6 | `mono-UF-stored-adrem` | `{{fixture_ibs_uf}}` | Aplicação IBS UF `N`, com adrem armazenado 2 preservado | OK (0) |
| 7 | `mono-UF-zero-adrem` | `{{fixture_ibs_uf}}` | Adrem IBS UF = 0 explícito | OK (0) |
| 8 | `mono-Municipal-stored-adrem` | `{{fixture_ibs_municipal}}` | Aplicação IBS municipal `N`, com adrem 2 preservado | OK (0) |
| 9 | `mono-Municipal-zero-adrem` | `{{fixture_ibs_municipal}}` | Adrem municipal = 0 explícito | OK (0) |
| 10 | `mono-CBS-stored-adrem` | `{{fixture_cbs}}` | Aplicação CBS `N`, com adrem 2 preservado | OK (0) |
| 11 | `mono-CBS-zero-adrem` | `{{fixture_cbs}}` | Adrem CBS = 0 explícito | OK (0) |

### 3.3 Controles I/Z — fixture regular (`{{fixture_regular}}`)

| # | Caso | Intenção | Esperado |
|---:|---|---|:---:|
| 12 | `CBS-I-restore` | Religa CBS regular com taxa positiva (setup) | OK (0) |
| 13 | `CBS-I-inherited-invalid` | Aplicação CBS `I`, sem enviar taxa: a taxa positiva herdada deve ser rejeitada | INVALID_ARGUMENT (3) |
| 14 | `CBS-I-explicit-zero-valid` | Aplicação CBS `I` + taxa 0 + redução 0 explícitas | OK (0) |
| 15 | `CBS-Z-restore` | Religa CBS regular (setup) | OK (0) |
| 16 | `CBS-Z-inherited-invalid` | Aplicação CBS `Z`, com taxa positiva herdada | INVALID_ARGUMENT (3) |
| 17 | `CBS-Z-explicit-zero-valid` | Aplicação CBS `Z` + taxa/redução 0 explícitas | OK (0) |
| 18 | `IBS-I-restore` | Religa IBS regular (setup) | OK (0) |
| 19 | `IBS-I-inherited-invalid` | Aplicação IBS `I`, com taxa positiva herdada | INVALID_ARGUMENT (3) |
| 20 | `IBS-I-explicit-zero-valid` | Aplicação IBS `I` + taxa/redução 0 explícitas | OK (0) |
| 21 | `IBS-Z-restore` | Religa IBS regular (setup) | OK (0) |
| 22 | `IBS-Z-inherited-invalid` | Aplicação IBS `Z`, com taxa positiva herdada | INVALID_ARGUMENT (3) |
| 23 | `IBS-Z-explicit-zero-valid` | Aplicação IBS `Z` + taxa/redução 0 explícitas | OK (0) |

### 3.4 Releitura

Para cada fixture abaixo, chame `GetContractDataById` e `GetContractItemsDataById`. Use o script "gRPC OK".

| Fixture | O que conferir na resposta |
|---|---|
| `{{fixture_historica}}` (638755) | Header e itens legíveis: 2 itens e 1 subitem. No Postman já deu NOT_FOUND; via .NET retornou 200 |
| `{{fixture_regular}}` | Zeros finais (caso 4) e aplicações I/Z (casos 14, 17, 20, 23) |
| `{{fixture_ibs_uf}}` | Adrem 0 e item irmão inalterado |
| `{{fixture_ibs_municipal}}` | Adrem 0 e item irmão inalterado |
| `{{fixture_cbs}}` | Adrem 0 e item irmão inalterado |

## 4. O que o Postman não cobre

A rodada original também conferiu **SQL antes/depois e 1,5 s depois**. Isso inclui itens irmãos e subitens byte a byte, e a ausência de escrita nas rejeições. No Postman só se vê o status e a releitura via gRPC. Para a mesma garantia, rode os SELECTs de `reproducao/consultas-readonly.sql` (pacote local) antes e depois dos casos 2, 5, 13, 16, 19 e 22.
