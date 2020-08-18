---
layout:   post
title:    "Cliente para Kafka (do zero) em .NET Core - parte 1"
date:     2020-08-18 12:40
comments: true
tags:     dotnet-core kafka protocol bedrock-framework
---

# Cliente para Kafka (do zero) em .NET Core - parte 1

Segundo a Wikpedia protocolo é descrito como:

> _Na ciência da computação, um protocolo é uma convenção que controla e possibilita uma conexão, comunicação, transferência de dados entre dois sistemas computacionais.
De maneira simples, um protocolo pode ser definido como "as regras que governam" a sintaxe, semântica e sincronização da comunicação. Os protocolos podem ser implementados pelo hardware, software ou por uma combinação dos dois._

Faz alguns meses que tenho investido tempo em compreender melhor gerenciamento de memória e otimização com .NET Core, e depois de tanto tempo trabalhando com isso percebo que realmente deveria ter começado antes - antes tarde do nunca não é?! 🤷‍♂️.

Nesse sentido estava a procura de um projeto em que pudesse desenvolver e testar essas capacidades do .NET, e a alguns meses conheci um projeto criado pelo [David Fowler](https://twitter.com/davidfowl) chamado [Bedrock Framework](https://github.com/davidfowl/BedrockFramework), basicamente é um conjunto de APIs em .NET Core que pode ser usado para construção de protocolos de comunicação entre cliente e servidor, o projeto se basea em novas abstrações introduzidas no .NET Core 3 ([Microsoft.AspNetCore.Connections.Abstractions](https://www.nuget.org/packages/Microsoft.AspNetCore.Connections.Abstractions)). Então você pode criar seu próprio servidor que se comunica usando um protocolo customizado, utilizando a infraestrutura do [Kestrel](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel?view=aspnetcore-3.1) e [System.IO.Pipelines](https://devblogs.microsoft.com/dotnet/system-io-pipelines-high-performance-io-in-net/).

_Neste post, falarei pouco sobre o funcionamento do [Kafka](https://kafka.apache.org/), não explicarei termos básicos, o foco é o que se relaciona a comunicação e a implementação em .NET, por isso posso citar termos que não terei explicado, mas podem ser facilmente encontrados._

## Kafka

Como o título deve deixar claro, o meu objetivo é criar do zero uma implementação do protocolo de comunicação com o [Apache Kafka](https://kafka.apache.org/), não tenho a ambição de cobrir todos os recursos, mas o mínimo para publicar e consumir mensagens.

> _Apache Kafka é uma plataforma distribuída de fluxo de eventos, é de código-aberto e usado por milhares de companhias para fluxos de alta performace de dados, fluxo de análises, integração de dados, e aplicações críticas. (https://kafka.apache.org) [minha tradução livre]_

A [documentação](https://kafka.apache.org/protocol.html) é muito boa e traz nos detalhes como deve ser feita a comunicação. Na Farfetch trabalhamos com Kafka como principal plataforma para comunicação assíncrona entre as aplicações, recentemente foi lançado o [primeiro projeto](https://github.com/Farfetch/kafka-flow) open-source da empresa que é um framework para comunicação com Kafka 🙄.

### Comunicação

Kafka usa um protocolo binário sobre TCP, onde as mensagens de requisição e resposta são aos pares, para cada requisição há uma resposta. Todas mensagens são de tamanho delimitado e são construídas com tipos primitivos (char, integer, long e etc). Há também indicações para como as conexões com os _brokers_ (nós de um cluster) deve ser geridas, deve se manter uma conexão para cada _broker_, os servidores vão garantir a sequência de execução das requisições em uma conexão. A conexão com os _brokers_ deve ser mantida por conta de como as mensagens são particionadas pelos _brokers_ e também para obter uma melhor performace.

### Particionamento e Inicialização

Kafka, não mantém todos os dados em todos os nós do _cluster_, o tópico é criado contando com um número de partições inicial, e então cada partição é replicada usando um dado fator de replicação. As mensagens devem ser publicadas para o _broker_ que é o lider para dada partição do tópico, e isso é possível descobrir enviando uma requisição para buscar os metadados do _cluster_, você terá a informação de quais _brokers_ possuem quais tópicos e partições, assim pode se ter um cache do lado do cliente com esses metadados e atualizar o cache quando as requisições para publicar ou recuperar mensagens falharem com o código de erro _**NotLeaderForPartition**_.

### Protocolo

Como disse anteriormente as mensagens tem um tamanho delimitado, ou seja, devemos saber exatamente o tamanho da mensagem antes de começar enviar os bytes, isso porque é com base no tamanho da mensagem que o servidor pode delimitar as mensagens recebidas pelo cliente, o mesmo vale quando o cliente está a analisar os dados recebidos do servidor. Os campos obrigatórios para o envio ou leitura de uma mensagem Kafka são os seguintes:

#### Tamanho (requisições/respostas)

| Campo | Tipo | Tamanho (bytes) | Descrição |
|--------------------------------------------|
| message_size | INT32 | 4 (big-endian) | O campo message_size fornece o tamanho da mensagem de solicitação ou resposta subsequente em bytes. |
|--------------------------------------------|

#### Cabeçalho da requisição

| Campo | Tipo | Tamanho (bytes) | Descrição |
|--------------------------------------------|
| request_api_key | INT16 | 2 (big-endian) | Chave da API desta requisição. |
| request_api_version | INT16 | 2 (big-endian) | Versão da API desta requisição. |
| correlation_id | INT32 | 4 (big-endian) | Identificador de co-relação desta requisição (usado posteriormente para identificar as respostas geradas). |
|--------------------------------------------|

#### Cabeçalho da resposta

| Campo | Tipo | Tamanho (bytes) | Descrição |
|--------------------------------------------|
| correlation_id | INT32 | 4 (big-endian) | Identificador de co-relação da requisição. |
|--------------------------------------------|

## Projeto

Ok, com essas informações acho que já podemos começar, e a primeira coisa a fazer é criar um projeto e definir "onde-vai-ficar-o-que" 😅. Tenho algumas dúvidas quanto ao design de uma biblioteca desse tipo e vou tentar esclarece-las nesse processo. Inicialmente penso em [KafkaRaw](https://github.com/pvoliveira/kafkaraw) como um bom nome e não encontrei nada igual no _Nuget_, então será esse o nome do projeto. Também não desejo ter/manter compatilidade com versões mais antigas do .NET Core, tudo será baseado no 3.1.

![screenshot da solução inicial](/assets/images/kafkaraw-solution.jpg)

Baseado na estrutura básica de uma requisição procurei uma chamada da API que fosse simples e pudesse comprovar o funcionamento do projeto, por fim decidi utilizar a chamada para retornar as versões compatíveis da API pelo servidor [ApiVersions (ApiKey = 18)](https://kafka.apache.org/protocol#The_Messages_ApiVersions), basicamente só precisamos enviar no cabeçalho o _ApiKey_ da chamada e qual versão queremos usar, neste caso _*request_api_key*_ = 18 e _*request_api_version*_ = 0, o cabeçalho _*correlation_id*_ neste caso não tem muita importância pois apenas uma chamada será feita, e a vamos analisar a resposta imediatamente após a requisição.

Criei um método simples para se conectar com os _brokers_ indicados no construtor da classe e outro para realizar a chamada em si, vamos dar uma olhada neles:

```csharp
/// ...

public async Task ConnectAsync()
{
    foreach (var host in _initialBrokers)
    {
        _connections.TryAdd(host, await makeConnection(host));
    }

    _logger.LogInformation("Starting Kafka client.");

    async Task<ConnectionContext> makeConnection(DnsEndPoint h)
    {
        var client = new ClientBuilder(_serviceProvider)
                        .UseSockets()
                        .UseConnectionLogging()
                        .Build();

        var connection = await client.ConnectAsync(h);
        _logger.LogInformation($"Connected to {connection.LocalEndPoint}");

        return connection;
    };
}

public async Task<ApiVersionsResponse> GetApiVersions()
{
    var request = new ApiVersionsRequest(18, 0 , "kafkaraw/0.1");

    var conn = _connections.First();

    var protocol = new Protocols.ApiVersions();
    var reader = conn.Value.CreateReader();
    var writer = conn.Value.CreateWriter();

    await writer.WriteAsync(protocol, request);

    var result = await reader.ReadAsync(protocol);

    reader.Advance();

    return result.Message;
}

/// ...
```

No método `ConnectAsync` fazemos a conexão com os hosts, podemos ver aqui o uso do `ClientBuilder` fornecido pelo _Bedrock Framework_, o _framework_ faz uso de uma "API fluente" para encadear métodos que configuram parâmetros ou comportamentos do _framework_. Nesse caso queremos nos conectar via TCP ao host, por isso o uso do método `UseSockets`, adicionalmente `UseConnectionLogging` gerará logs da comunicação, ao final o método `Build` vai retornar um cliente que contém em si uma instância de `IConnectionFactory`, mais especificamente do tipo `SocketConnectionFactory` definido pelo método `UseSockets`, o qual abrirá uma conexão com o host e nos devolverá uma conexão pronta para comunicação. A conexão retornada pelo método `ConnectAsync` do _client_ é um `ConnectionContext` definido em `Microsoft.AspNetCore.Connections.Abstractions`, o _framework_ implementa "uma ligação" para a escrita/leitura utilizando `System.IO.Pipelines.Pipe` para otimizar a alocação de memória e performance.

O método `GetApiVersions` trata das chamadas para realizar a escrita dos dados para a conexão e análise da resposta - sem complexidades aqui.

A classe `ApiVersions` é quem contém a lógica para escrita e leitura para da requisição e resposta, ou seja, a ordem em que devem ser escritos/lidos os bytes.

```csharp
public class ApiVersions :
        IMessageReader<ApiVersionsResponse>,
        IMessageWriter<ApiVersionsRequest>
{
    public bool TryParseMessage(in ReadOnlySequence<byte> input, ref SequencePosition consumed, ref SequencePosition examined, out ApiVersionsResponse message)
    {
        var reader = new SequenceReader<byte>(input);
        if (!reader.TryReadBigEndian(out int length)
            || input.Length < length)
        {
            message = default;
            return false;
        }

        reader.TryReadBigEndian(out int correlationId);

        if (!reader.TryReadBigEndian(out short errorCode)
            || errorCode != 0)
        {
            throw new Exception($"Error Core: {errorCode}");
        }

        ApiKeys[] apiKeys = null;
        if (reader.TryReadBigEndian(out int apiKeysLength))
        {

            apiKeys = new ApiKeys[apiKeysLength];
            for (int i = 0; i < apiKeysLength; i++)
            {
                reader.TryReadBigEndian(out short apiKey);
                reader.TryReadBigEndian(out short minVersion);
                reader.TryReadBigEndian(out short maxVersion);
                apiKeys[i] = new ApiKeys(apiKey, minVersion, maxVersion);
            }
        }

        message = new ApiVersionsResponse(errorCode, apiKeys);

        consumed = reader.Position;
        examined = consumed;
        return true;
    }

    public void WriteMessage(ApiVersionsRequest message, IBufferWriter<byte> output)
    {
        var cltIdBytes = Encoding.UTF8.GetBytes(message.ClientId);

        var size = 8 + 2 + cltIdBytes.Length;

        // Size
        var sizeBuffer = output.GetSpan(4);
        BinaryPrimitives.WriteInt32BigEndian(sizeBuffer, size);
        output.Advance(4);

        // Header - RequestApiKey
        var rakBuffer = output.GetSpan(2);
        BinaryPrimitives.WriteInt16BigEndian(rakBuffer, message.RequestApiKey);
        output.Advance(2);

        // Header - RequestApiVersion
        var ravBuffer = output.GetSpan(2);
        BinaryPrimitives.WriteInt16BigEndian(ravBuffer, message.RequestApiVersion);
        output.Advance(2);

        // Header - CorrelationId
        var crIdBuffer = output.GetSpan(4);
        BinaryPrimitives.WriteInt32BigEndian(crIdBuffer, message.CorrelationId);
        output.Advance(4);

        // Headers - ClientId
        var cltIdNBuffer = output.GetSpan(2);
        BinaryPrimitives.WriteInt16BigEndian(cltIdNBuffer, (short)cltIdBytes.Length);
        output.Advance(2);
        output.Write(cltIdBytes);
    }
}
```

Com isso feito, é possível criar uma pequena aplicação console para por tudo a funcionar 🤞. Para testar, iniciei uma instância do Kafka usando _Docker_ a ouvir a porta padrão 9092.

```csharp
class Program
{
    static async Task Main(string[] args)
    {
        var provider = new ServiceCollection()
            .AddLogging(builder =>
                builder
                    .SetMinimumLevel(LogLevel.Debug)
                    .AddConsole(opts => opts.DisableColors = false))
            .AddKafkaRawClient("localhost:9092")
            .BuildServiceProvider();

        await provider.GetService<IKafkaRawClient>().ConnectAsync();

        var r = await provider.GetService<IKafkaRawClient>().GetApiVersions();

        string apis = JsonSerializer.Serialize(r, new JsonSerializerOptions { WriteIndented = true });

        provider.GetService<ILogger<Program>>().LogDebug($"ApiVersions: {apis}");

        Console.ReadKey();
    }
}
```

Abaixo temos a saída da execução da aplicação console:

```shell
info: KafkaRaw.KafkaRawClient[0]
      Connected to [::1]:8662
info: KafkaRaw.KafkaRawClient[0]
      Starting Kafka client.
dbug: Bedrock.Framework.LoggingConnectionMiddleware[0]
      WriteAsync[26] 00 00 00 16 00 12 00 00 00 00 00 01 00 0C 6B 61 66 6B 61 72 61 77 2F 30 2E 31
      \x00\x00\x00\x16\x00\x12\x00\x00\x00\x00\x00\x01\x00\x0Ckafkaraw/0.1
dbug: Bedrock.Framework.LoggingConnectionMiddleware[0]
      ReadAsync[302] 00 00 01 2A 00 00 00 01 00 00 00 00 00 30 00 00 00 00 00 08 00 01 00 00 00 0B 00 02 00 00 00 05 00 03 00 00 00 09 00 04 00 00 00 04 00 05 00 00 00 02 00 06 00 00 00 06 00 07 00 00 00 03 00 08 00 00 00 08 00 09 00 00 00 06 00 0A 00 00 00 03 00 0B 00 00 00 06 00 0C 00 00 00 04 00 0D 00 00 00 04 00 0E 00 00 00 04 00 0F 00 00 00 05 00 10 00 00 00 03 00 11 00 00 00 01 00 12 00 00 00 03 00 13 00 00 00 05 00 14 00 00 00 04 00 15 00 00 00 01 00 16 00 00 00 02 00 17 00 00 00 03 00 18 00 00 00 01 00 19 00 00 00 01 00 1A 00 00 00 01 00 1B 00 00 00 00 00 1C 00 00 00 02 00 1D 00 00 00 01 00 1E 00 00 00 01 00 1F 00 00 00 01 00 20 00 00 00 02 00 21 00 00 00 01 00 22 00 00 00 01 00 23 00 00 00 01 00 24 00 00 00 01 00 25 00 00 00 01 00 26 00 00 00 02 00 27 00 00 00 01 00 28 00 00 00 01 00 29 00 00 00 01 00 2A 00 00 00 02 00 2B 00 00 00 02 00 2C 00 00 00 01 00 2D 00 00 00 00 00 2E 00 00 00 00 00 2F 00 00 00 00
      \x00\x00\x01*\x00\x00\x00\x01\x00\x00\x00\x00\x000\x00\x00\x00\x00\x00\x08\x00\x01\x00\x00\x00\x0B\x00\x02\x00\x00\x00\x05\x00\x03\x00\x00\x00\x09\x00\x04\x00\x00\x00\x04\x00\x05\x00\x00\x00\x02\x00\x06\x00\x00\x00\x06\x00\x07\x00\x00\x00\x03\x00\x08\x00\x00\x00\x08\x00\x09\x00\x00\x00\x06\x00\x0A\x00\x00\x00\x03\x00\x0B\x00\x00\x00\x06\x00\x0C\x00\x00\x00\x04\x00\x0D\x00\x00\x00\x04\x00\x0E\x00\x00\x00\x04\x00\x0F\x00\x00\x00\x05\x00\x10\x00\x00\x00\x03\x00\x11\x00\x00\x00\x01\x00\x12\x00\x00\x00\x03\x00\x13\x00\x00\x00\x05\x00\x14\x00\x00\x00\x04\x00\x15\x00\x00\x00\x01\x00\x16\x00\x00\x00\x02\x00\x17\x00\x00\x00\x03\x00\x18\x00\x00\x00\x01\x00\x19\x00\x00\x00\x01\x00\x1A\x00\x00\x00\x01\x00\x1B\x00\x00\x00\x00\x00\x1C\x00\x00\x00\x02\x00\x1D\x00\x00\x00\x01\x00\x1E\x00\x00\x00\x01\x00\x1F\x00\x00\x00\x01\x00 \x00\x00\x00\x02\x00!\x00\x00\x00\x01\x00"\x00\x00\x00\x01\x00#\x00\x00\x00\x01\x00$\x00\x00\x00\x01\x00%\x00\x00\x00\x01\x00&\x00\x00\x00\x02\x00'\x00\x00\x00\x01\x00(\x00\x00\x00\x01\x00)\x00\x00\x00\x01\x00*\x00\x00\x00\x02\x00+\x00\x00\x00\x02\x00,\x00\x00\x00\x01\x00-\x00\x00\x00\x00\x00.\x00\x00\x00\x00\x00/\x00\x00\x00\x00
dbug: KafkaClient.Program[0]
      ApiVersions: {
        "ErrorCode": 0,
        "ApiKeys": [
          {
            "ApiKey": 0,
            "MinVersion": 0,
            "MaxVersion": 8
          },
          {
            "ApiKey": 1,
            "MinVersion": 0,
            "MaxVersion": 11
          },
          {
            "ApiKey": 2,
            "MinVersion": 0,
            "MaxVersion": 5
          },
          {
            "ApiKey": 3,
            "MinVersion": 0,
            "MaxVersion": 9
          },
          ...
        ]
      }
```

E funciona!!! 😁😁😁 o método da API do Kafka chamado retorna qual a versão mínima e máxima aceita pelo _broker_.

## Conclusão

A protocolo de comunicação com o Kafka é bem documentado e será fácil seguir implementando novos métodos, o próximo desafio será organizar o fluxo de comunicação: _healthchecks_, cache dos metadados do _cluster_, e definir quais recursos o [KafkaRaw](https://github.com/pvoliveira/kafkaraw) irá expor.

Tentei ser mais direto possível para mostrar como você pode implementar um protocolo customizado de comunicação com .NET Core, por isso não tentei deixar as coisas no "melhor estado da arte", e muito menos tocar em performance, gerenciamento de memória e etc, mas espero após alguns posts chegar a isso. Para qualquer dúvida ou discussão, o projeto já está no _GitHub_, pode se abrir uma _issue_, deixar um comentário aqui no post ou falar diretamente comigo.

Muito obrigado e até o próximo 😎
