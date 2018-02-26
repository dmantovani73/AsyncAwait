# Problema
Il metodo VeryLongIO esegue un'operazione di I/O molto lunga. L'I/O (accesso al disco, accesso alla rete, chiamate HTTP, accesso al db, ...) solitamente è fatto tramite chiamate ad API del sistema operativo che a sua volta utilizza thread di più basso livello (non applicativi). Il problema è quindi che il thread corrente rimane inutilmente bloccato in attesa che il metodo VeryLongIO termini la propria esecuzione. Se non rimanesse bloccato sarebbe libero di eseguire altro codice applicativo. 
Ci sono scenari particolarmente critici come ad es. quando un'operazione di I/O tiene bloccato il thread UI (il solo thread che per ogni processo è deputato ad aggiornare la GUI) bloccando quindi la gestione di eventi (click, resize di finestre, ...) e l'aggiornamento della GUI stessa.

```csharp
void Main()
{
	Dump("Prima");
	VeryLongIO();
	
	// Il thread rimane inultimente bloccato fino a quanto non termina l'operazione di IO (VeryLongIO).
	Dump("Dopo");
}

int VeryLongIO(int count = 2)
{
	var url = "https://it.lipsum.com/";

	using (var client = new WebClient())
	{
		var length = 0;
		for (var i = 0; i < count; i++)
		{
			var html = client.DownloadString(url);
			length += html.Length;
		}

		return length;
	}
}

void Dump(string text) => text.Dump($"Thread ID: {Thread.CurrentThread.ManagedThreadId}");
```

# Task
.NET mette a disposizione una classe (e relative API) che permette di eseguire più operazioni in parallelo: https://docs.microsoft.com/it-it/dotnet/csharp/async.

La classe Task si trova in altri linguaggi (es. Java, JavaScript) con il nome di Promise.
Task è un'astrazione del concetto di thread, più task potrebbero però essere eseguiti dallo stesso thread. In questo senso il Task esprime l'intenzione in senso astratto mentre il thread è il mezzo fisico per portarla a compimento.

```csharp
Task<int> VeryLongIOAsync(int count = 2)
{
	return Task.Run(() => VeryLongIO(count));
}
```

I task consentono in modo semplice di aggiungere una continuation (AKA callback) da eseguire quando il task stesso è terminato:

```csharp
void Main()
{
	Dump("Prima");
  
	// Logicamente equivalente a:
	VeryLongIOAsync().ContinueWith(task =>
	{
		Dump($"Dopo, Length = {task.Result}");
	});

	Dump("Dopo");
}
```

# Async / Await

Il "pattern" task / continuation in C# diventa un idioma grazie alle keyword async / await. Il precedente codice può essere così semplificato scrivendo:

```csharp
async Task Main()
{
	Dump("Prima");
	var length = await VeryLongIOAsync();
	Dump("Dopo");
}
```

# Awaitable / Awaiter Pattern

Alla base di async / await c'è una coppia di tipi che svolgono il ruolo di awaitable (ad es. il Task) / awaiter (il codice che avvia il task e si mette in attesa asincrona che termini). 
Supponiamo ad esempio di voler implementare il tipo FuncAwaitable che permette di eseguire in modo asincrono (await) una Func<T>:

```csharp
async Task Main()
{
	var result = await new FuncAwaitable<int>(() => 10 + 20);
	result.Dump();
}
```

E' possibile fare await di un tipo se questo espone un metodo GetAwaiter che ritorna un oggetto di tipo "awaiter". Incapsuliamo il contratto in un'interfaccia:

```csharp
interface IAwaitable<T>
{
	IAwaiter<T> GetAwaiter();
}
```

L'oggetto ritornato da GetAwaiter deve implementare l'interfaccia INotifyCompletion:

```csharp
interface IAwaiter<T> : INotifyCompletion
{
	bool IsCompleted { get; }

	T GetResult();
}
```

A questo punto implementiamo i due tipi:

```csharp
class FuncAwaitable<T> : IAwaitable<T>
{
	Func<T> func;
	
	public FuncAwaitable(Func<T> func)
	{
		this.func = func;
	}
	
	public IAwaiter<T> GetAwaiter() => new FuncAwaiter<T>(func);
}


class FuncAwaiter<T> : IAwaiter<T>
{
	Task<T> task;
	
	public FuncAwaiter(Func<T> func)
	{
		task = new Task<T>(func);
		task.Start();
	}
	
	public void OnCompleted(Action continuation) => continuation?.Invoke();
	
	public bool IsCompleted => task.IsCompleted;
	
	public T GetResult() => task.Result;
}
```

In realtà il pattern funziona anche eliminando la class awaitable e sostituendola con un extension method. FuncAwaitable può quindi essere eliminato e sostituito con un extension method:

```csharp
static class AwaitableExtensions
{
	public static FuncAwaiter<T> GetAwaiter<T>(this Func<T> func) => new FuncAwaiter<T>(func);
}
```

Il precedente è un esempio solamente didattico perchè nella realtà per eseguire un modo asincrono una funzione è sufficiente scrivere:

```csharp
async Task Main()
{
	var result = await Task.Run(() => 10 + 20);
	result.Dump();
}
```

# Task.Yield

# Riferimenti
https://docs.microsoft.com/it-it/dotnet/csharp/async
https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/task-based-asynchronous-programming
https://weblogs.asp.net/dixin/understanding-c-sharp-async-await-1-compilation
https://weblogs.asp.net/dixin/understanding-c-sharp-async-await-2-awaitable-awaiter-pattern
