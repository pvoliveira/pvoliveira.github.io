---
layout:   post
title:    "Cliente para Kafka (do zero) em .NET Core - parte 1"
date:     2020-08-06 23:40
comments: true
tags:     dotnet-core .net-core kafka protocol bedrock-framework
---

# Cliente para Kafka (do zero) em .NET Core - parte 1

Segundo a Wikpedia protocolo é descrito como:

> _Na ciência da computação, um protocolo é uma convenção que controla e possibilita uma conexão, comunicação, transferência de dados entre dois sistemas computacionais.
De maneira simples, um protocolo pode ser definido como "as regras que governam" a sintaxe, semântica e sincronização da comunicação. Os protocolos podem ser implementados pelo hardware, software ou por uma combinação dos dois._

Faz alguns meses que tenho investido tempo em compreender melhor gerenciamento de memória e otimização com .NET Core, e depois de tanto tempo trabalhando com isso percebo que realmente deveria ter começado antes, antes cedo do nunca 🤷‍♂️.

Nesse sentido estava a procura de um projeto em que pudesse desenvolver e testar essas capacidades do .NET, e a alguns meses conheci esse projeto criado pelo [David Fowler](https://twitter.com/davidfowl) chamado [Bedrock Framework](https://github.com/davidfowl/BedrockFramework), basicamente é um conjunto de APIs em .NET Core que pode ser usado para construção de protocolos de comunicação entre cliente e servidor, o projeto se basea em novas abstrações introduzidas no .NET Core 3 ([Microsoft.AspNetCore.Connections.Abstractions](https://www.nuget.org/packages/Microsoft.AspNetCore.Connections.Abstractions)). Então você pode criar seu próprio servidor que se comunica usando um protocolo customizado, utilizando a infraestrutura do [Kestrel](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel?view=aspnetcore-3.1) e [System.IO.Pipelines](https://devblogs.microsoft.com/dotnet/system-io-pipelines-high-performance-io-in-net/).

Neste post, falarei pouco sobre o funcionamento do _Kafka_, não explicarei termos básicos, o foco é o que se relaciona a comunicação e a implementação em .NET, por isso posso citar termos que não terei explicado, mas podem ser facilmente encontrados.

## Kafka

Como o título deve deixar claro, o meu objetivo é criar do zero uma implementação do protocolo de comunicação com o [Apache Kafka](https://kafka.apache.org/), não tenho a ambição de cobrir todos os recursos, mas o mínimo para publicar mensagens e consumir mensagens.

> _Apache Kafka é uma plataforma distribuída de fluxo de eventos, é de código-aberto e usado por milhares de companhias para fluxos de alta performace de dados, fluxo de análises, integração de dados, e aplicações críticas. (https://kafka.apache.org) [minha tradução livre]_

A [documentação](https://kafka.apache.org/protocol.html) é muito boa e traz nos detalhes como deve ser feita a comunicação e também é uma oportunidade de aprender mais profundamente comunicação via TCP em .NET e o desenho uma API (biblioteca/framework). Na Farfetch trabalhamos com Kafka como principal plataforma para comunicação assíncrona entre as aplicações, recentemente foi lançado o [primeiro projeto](https://github.com/Farfetch/kafka-flow) open-source da empresa que é um framework para comunicação com Kafka 🙄.

### Comunicação

Kafka usa um protocolo binário sobre TCP, onde as mensagens de requisição e resposta são aos pares, para cada requisição há uma resposta. Todas mensagens são de tamanho delimitado e são construídas com tipos primitivos (char, integer, long e etc). Há também indicações para como as conexões com os _brokers_ (nós de um cluster) deve ser geridas, deve se manter uma conexão para cada _broker_, os servidores vão garantir a sequência de execução das requisições em uma conexão. A conexão com os _brokers_ deve ser mantida por conta de como as mensagens são particionadas pelos _brokers_ e também para obter uma melhor performace.

### Particionamento e Inicialização

Kafka, não mantém todos os dados em todos os nós do _cluster_, o tópico é criado contando com um número de partições inicial, e então cada partição é replicada usando um dado fator de replicação. As mensagens devem ser publicadas para o _broker_ que é o lider para dada partição do tópico, e isso é possível descobrir enviando uma requisição para buscar os metadados do _cluster_, você terá a informação de quais _brokers_ possuem quais tópicos e partições, assim pode se ter um cache do lado do cliente com esses metadados e atualizar o cache quando as requisições para publicar ou recuperar mensagens falharem com o código de erro _**NotLeaderForPartition**_.

### Protocolo

Como disse anteriormente as mensagens tem um tamanho delimitado, ou seja, devemos saber exatamente o tamanho da mensagem antes de começar enviar os bytes, isso porque é com base no tamanho da mensagem que o servidor pode delimitar os campos que a mensagem contém. Os campos padrão para o envio de uma mensagem ao Kafka são os seguintes:

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

Ok, com essas informações acho que já podemos começar, e a primeira coisa a fazer é criar um projeto e definir onde-vai-ficar-o-que (😅). Tenho algumas dúvidas quanto ao design de uma biblioteca desse tipo e vou tentar esclarece-las nesse processo. Inicialmente penso em [_**KafkaRaw**_]() como um bom nome e não encontrei nada igual no _Nuget_, então será esse o nome do projeto. Também não desejo ter/manter compatilidade com versões mais antigas do .NET Core, tudo será baseado no 3.1.

![screenshot da solução inicial](/assets/images/kafkaraw-solution.jpg)

Baseado na estrutura básica de uma requisição procurei uma chamada da API que fosse simples e pudesse comprovar o funcionamento do projeto., por fim decidi utilizar a chamada para retornar as versões compatíveis da API pelo servidor [ApiVersions (ApiKey = 18)](https://kafka.apache.org/protocol#The_Messages_ApiVersions), basicamente só precisamos enviar no cabeçalho o _ApiKey_ da chamada e qual versão do método queremos usar, neste caso _*request_api_key*_ = 18 e _*request_api_version*_ = 0, o cabeçalho _*correlation_id*_ neste caso não tem muita importância pois apenas uma chamada será feita, e a vamos analisar imediatamente após a requisição ser feita.

Criei um método simples para se conectar com os _brokers_ indicados no construtor da classe e outro para realizar a chamada em si:

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

Olhando primeiro para o método `ConnectAsync` onde fazemos a conexão com os hosts, podemos ver aqui o uso do `ClientBuilder` fornecido pelo _Bedrock Framework_, o _framework_ faz uso de uma "API fluente" para conseguirmos encadear métodos que configuram parâmetros ou comportamentos do _framework_. Nesse caso queremos nos conectar via TCP ao host, por isso o uso do método `UseSockets`, adicionalmente `UseConnectionLogging` gerá logs da comunicação, ao final o método `Build` vai retornar um cliente que contém em si uma instância para um `IConnectionFactory` (definido pelo método `UseSockets`) o qual abrirá uma conexão com o host e nos devolverá uma conexão pronta para comunicação com o host.

O `GetApiVersions` trata das chamada para realizar a escrita dos dados para a conexão e análise da resposta - sem grandes complexidades.


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