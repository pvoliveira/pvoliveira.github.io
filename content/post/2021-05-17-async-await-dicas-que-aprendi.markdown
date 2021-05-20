---
comments: true
title: "Async/Await - Pequenas dicas ;)"
date: 2021-05-20T00:59:51+01:00
categories: [.net,async-await,dotnet,dotnet-core,performance,optimization]
---

Olá novamente, sei que passou muito tempo até chegarmos nesse novo post, tive contra-tempos, falta de vontade de continuar, e comecei a fazer algo que não fazia que era guardar em algum lugar o que aprendi para partilhar aqui. Enfim, vamos a isso 🤓

Estive a fazer um curso sobre código assíncrono em .NET/C# [Async Experts](https://asyncexpert.com), e acho que tem conteúdo suficiente para render alguns posts, então comecei a fazer uma lista para partilhar aqui, agora vou deixar uma dica que você pode já saber ou talvez nunca tenha notado.

## Exceptions

Observe o exemplo abaixo por um minuto e tente identificar se existe algo "mal":

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

O código não tem nenhum problema, o método `A_MethodA_Async` irá ser executado sem gerar exceções, enquanto o método `B_MethodA_Async` lançará uma exceção do tipo `ArgumentException`. Você pode já ter escrito métodos dessa forma e vão funcionar, mas se pensarmos em como métodos assíncronos são executados estamos a utilizar um pouco mal os recursos quando o método `B_MethodA_Async` é executado.

Quando fazemos `await` para computar o valor de uma `Task` estamos a escalar a execução da `Task` para o `ThreadPool`, pois bem, no exemplo acima existem dois caminhos para nosso código executar: primeiro quando passar um valor que é vazio ou nulo, o segundo é quando passamos um valor válido e temos uma nova chama assíncrona. Isso significa que podemos (ou não) estar a utilizar mais de uma thread para processar código que poderia ser síncrono, vamos reescrever o código do método `Method_A_Async`.

```csharp
public Task<bool> Method_B_Async(string value)
{
    if (string.IsNullOrEmpty(value))
    {
        throw new ArgumentException();
    }

    return Method_B_InternalAsync(value);
}

private async Task<bool> Method_B_InternalAsync(string value)
{
    var response = await httpClient.GetAsync(value);
    
    return response != null;
}
```

Primeiro notem que removi o `async` da assinatura do método, não estamos mais a utilizar o `await` para nenhuma chamada, então isso torna `Method_B_Async` um método síncrono. Também criei um novo método `Method_B_InternalAsync` (usando `async` na assinatura) que não tem mais as validações necessárias para os parâmetros (feitas no método `Method_B_Async`), isso é chamado de "async eliding" e a tradução pode ficar um pouco estranha mas é algo como *"suprimir/omitir o assíncrono"*.

Abaixo podemos ver melhor o que significa adicionar o `async` na assinatura do método:

```csharp
[AsyncStateMachine(typeof(<Method_A_Async>d__1))]
[DebuggerStepThrough]
public Task<bool> Method_A_Async(string value)
{
    <Method_A_Async>d__1 stateMachine = new <Method_A_Async>d__1();
    stateMachine.<>t__builder = AsyncTaskMethodBuilder<bool>.Create();
    stateMachine.<>4__this = this;
    stateMachine.value = value;
    stateMachine.<>1__state = -1;
    stateMachine.<>t__builder.Start(ref stateMachine);
    return stateMachine.<>t__builder.Task;
}
```

E sem `async`:

```csharp
public Task<bool> Method_B_Async(string value)
{
    if (string.IsNullOrEmpty(value))
    {
        throw new ArgumentException();
    }
    return Method_B_InternalAsync(value);
}

[AsyncStateMachine(typeof(<Method_B_InternalAsync>d__2))]
[DebuggerStepThrough]
private Task<bool> Method_B_InternalAsync(string value)
{
    <Method_B_InternalAsync>d__2 stateMachine = new <Method_B_InternalAsync>d__2();
    stateMachine.<>t__builder = AsyncTaskMethodBuilder<bool>.Create();
    stateMachine.<>4__this = this;
    stateMachine.value = value;
    stateMachine.<>1__state = -1;
    stateMachine.<>t__builder.Start(ref stateMachine);
    return stateMachine.<>t__builder.Task;
}
```

Um máquina de estados é criada para a execução do método assíncrono e a validação dos parâmetros será feito dentro dessa máquina de estados. Enquanto se utilizarmos a outra abordagem, conseguimos manter as validações que não dependem de métodos assíncronos fora do fluxo assíncrono de execução.

Pode parecer um detalhe mas evitar a máquina de estados (no nosso exemplo quando vamos parar a execução do método por causa de uma validação/exceção) além de poder melhorar o uso de mais threads na execução também evita a alocação de objetos para gerir o código assíncrono. São otimizações que em *hot-paths* (trechos muito executados na aplicação) podem gerar ganhos positivos e performance 😉.
