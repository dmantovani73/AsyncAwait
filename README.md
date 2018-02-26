# Problema
Un metodo eseguito sequenzialmente blocca il thread corrente fino a quando non è terminata l'operazione di IO e quindi non può eseguire altre operazioni nel mentre.

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

