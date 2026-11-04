# Reprodução controlada

Este roteiro pode criar novas fixtures de teste; requer autorização explícita para live writes e acesso TRUNK existente. Não contém nem fornece credenciais. Não reexecutar automaticamente ao abrir o pacote.

1. Confira a branch/HEAD remoto e use cópia isolada do commit `a93616df7d369ee1d6546186bfb423edf49bd697`. Use a configuração TRUNK privada já autorizada, sem compartilhá-la. Verifique identidade do SQL e CALCOFF=N, exposição=S por SELECT. Não altere flags.
2. Organize `<raiz>/source` com o checkout exato e `<raiz>/runner` com Program.cs e Runner.csproj desta pasta. O runner lê a configuração privada em `source/src/Grpc/appsettings.Development.json`; isso é caminho de leitura, não credencial embutida. O arquivo de configuração não faz parte do pacote.
3. Compile Release: `dotnet build <raiz>/source/src/Grpc/Grpc.csproj -c Release`. Registre os hashes e metadados do binário realmente executado.
4. Inicie o serviço local na pasta `source/src/Grpc`, aplicando somente configuração de processo: ASPNETCORE_ENVIRONMENT=Development, EFCoreConfiguration__RunMigrations=false, EFCoreConfiguration__EnableSensitiveDataLogging=false, EFCoreConfiguration__EnableLog=false, UseInMemoryDatabase=false, Kestrel__Endpoints__Grpc__Url=http://127.0.0.1:24600, Kestrel__Endpoints__Grpc__Protocols=Http2, Kestrel__Endpoints__Http__Url=http://127.0.0.1:24601, Kestrel__Endpoints__Http__Protocols=Http1. Execute `dotnet bin/Release/net10.0/Grpc.dll`. Não iniciar se as portas estiverem ocupadas; não parar processos alheios.
5. Leitura apenas: `dotnet run --project <raiz>/runner/Runner.csproj -- --calcoff --preflight`.
6. Somente com autorização live: `dotnet run --project <raiz>/runner/Runner.csproj -- --calcoff`. Cria quatro contratos TEST MW92734 pelo AddContract e aplica os onze testes via UpdateContract, sem writes SQL diretos. Gera novas evidências em evidence-calcoff. Preserve o pacote original.
7. Confira resultados, flags e estado posterior, registre IDs e encerre somente o serviço iniciado por você depois de terminarem requests/runner. Não deletar fixtures nem consumir fila.

O runner compartilhável recebeu apenas uma guarda de empacotamento exigindo --calcoff, para impedir uso fora desse modo; não foi reexecutado após essa guarda. A lógica dos onze testes é a mesma da execução registrada. PackageReference aponta SQLClient6.1.1 e o SDK do source1.46.0. Não há tolerâncias novas nem alterações de aplicação.
