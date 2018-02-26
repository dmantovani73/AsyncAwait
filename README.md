# Problema
Il metodo VeryLongIO esegue un'operazione di I/O molto lunga. L'I/O (accesso al disco, accesso alla rete, chiamate HTTP, accesso al db, ...) solitamente è fatto tramite chiamate ad API del sistema operativo che a sua volta utilizza thread di più basso livello (non applicativi). Il problema è quindi che il thread corrente rimane inutilmente bloccato in attesa che il metodo VeryLongIO termini la propria esecuzione. Se non rimanesse bloccato sarebbe libero di eseguire altro codice applicativo. 
Ci sono scenari particolarmente critici come ad es. quando un'operazione di I/O tiene bloccato il thread UI (il solo thread che per ogni processo è deputato ad aggiornare la GUI) bloccando quindi la gestione di eventi (click, resize di finestre, ...) e l'aggiornamento della GUI.

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
L'utilizzo della classe Task permette di eseguire parallelamente (in un thread separato rispetto al thread corrente) un blocco di codice.

```csharp
Task<int> VeryLongIOAsync(int count = 2)
{
	return Task.Run(() => VeryLongIO(count));
}
```

E' quindi possibile eseguire un metodo in un thread (senza bloccare quello corrente) e invocare un callback al termine dell'operazione.

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

```csharp
async Task Main()
{
	Dump("Prima");
	var length = await VeryLongIOAsync();
	Dump("Dopo");
}
```

# Awaitable / Awaiter Pattern

```csharp
async Task Main()
{
	var result = await new FuncAwaitable<int>(() => 30);
	result.Dump();
	
// Utilizzo di un extension method.
// var result = await new Func<int>(() => 20);
// result.Dump();

// In realtà tutto quello che c'è sotto non serve (è solo a scopo didattico) e basta così:
// var result = await Task.Run(() => 30);
// result.Dump();
}

static class AwaitableExtensions
{
	public static FuncAwaiter<T> GetAwaiter<T>(this Func<T> func) => new FuncAwaiter<T>(func);
}


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

interface IAwaiter<T> : INotifyCompletion
{
	bool IsCompleted { get; }

	T GetResult();
}

interface IAwaitable<T>
{
	IAwaiter<T> GetAwaiter();
}
```
