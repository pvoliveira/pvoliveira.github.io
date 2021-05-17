---
title: "Async/Await - Pequenas dicas ;)"
date: 2021-05-16T19:28:51+01:00
draft: true
---

Olá novamente, sei que passou muito tempo até chegarmos nesse novo post, tive contra-tempos, falta de vontade de continuar, e comecei a fazer algo que não fazia que era guardar em algum lugar o que aprendi para partilhar aqui. Enfim, vamos a isso 🤓


Estive a fazer um curso sobre código assíncrono em .NET/C# [Async Experts](https://asyncexpert.com), e estava cheio de coisas para render vários posts, então comecei a fazer uma lista para partilhar aqui, agora vou deixar uma dica que você pode já saber sobre isso ou talvez nunca tenha notado.

## Exceptions

Primeira dica é sobre como lidar com exceções no código assíncrono, para ajudar observe o exemplo abaixo por um minuto e tente identificar se existe algo "mal":

```csharp
public async Task<bool> Method_A_Async(string value)
{
    if (string.IsNullOrEmpty(value))
    {
        throw new ArgumentException();
    }

    var response = await httpClient.GetAsync(value);
    
    return response != null;
}

public async Task A_MethodA_Async()
{
    var x = await Method_A_Async("x");
}

public async Task B_MethodA_Async()
{
    var x = await Method_A_Async("");
}
```

O código não tem nenhum problema, o método `A_MethodA_Async` irá ser executado sem gerar exceções, enquanto o método `B_MethodA_Async` lançará uma exceção do tipo `ArgumentException`. Você pode já ter escrito métodos dessa forma e vão funcionar, mas se pensarmos em como métodos assíncronos são executados estamos a utilizar um pouco mal os recursos no método `B_MethodA_Async`.

Quando fazemos `await` para computar o valor de uma `Task` estamos a escalar a execução da `Task` para o `ThreadPool`, pois bem, no exemplo acima existem dois caminhos para nosso código executar: primeiro quando passar um valor que é vazio ou nulo, o segundo é quando passamos um valor válido e temos uma nova chama assíncrona. Isso significa que podemos (ou não) estar a utizar mais de uma thread para processar código que poderia ser síncrono, vamos reescrever o código do método `Method_A_Async`.

```csharp
public Task<bool> Method_A_Async(string value)
{
    if (string.IsNullOrEmpty(value))
    {
        throw new ArgumentException();
    }

    return Method_A_InternalAsync(value);
}

private async Task<bool> Method_A_InternalAsync(string value)
{
    var response = await httpClient.GetAsync(value);
    
    return response != null;
}
```

Primeiro notem que removi o `async` da assinatura do método, não estamos mais a utilizar o `await` para nenhuma chamada, então isso torna `Method_A_Async` um método síncrono. Também criei um novo método `Method_A_InternalAsync` (usando `async` na assinatura) que não tem mais as validações necessárias para os parâmetros (feitas no método `Method_A_Async`)