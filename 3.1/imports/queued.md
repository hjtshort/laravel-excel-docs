# Queued reading

[[toc]]

## Queuing chunks

When using the `WithChunkReading` concern, you can also choose to execute each chunk into a queue job. You can do so by simply adding the `ShouldQueue` contract.

```php
namespace App\Imports;

use App\User;
use Maatwebsite\Excel\Concerns\ToModel;
use Illuminate\Contracts\Queue\ShouldQueue;
use Maatwebsite\Excel\Concerns\WithChunkReading;

class UsersImport implements ToModel, WithChunkReading, ShouldQueue
{
    public function model(array $row)
    {
        return new User([
            'name' => $row[0],
        ]);
    }
    
    public function chunkSize(): int
    {
        return 1000;
    }
}
```

Each chunk of 1000 rows will now be executed into a queue job.

:::warning
`ShouldQueue` is only supported in combination with `WithChunkReading`.
:::

### Explicit queued imports

You can explicitly queue the import by using `::queueImport`. 

```
Excel::queueImport(new UsersImport, 'users.xlsx');
```

When using the `Importable` trait you can use the `queue` method:

```
(new UsersImport)->queue('users.xlsx');
```

:::warning
The `ShouldQueue` is always required.
:::

### Implicit queued imports

When `ShouldQueue` is used, the import will automatically be queued.

```
Excel::import(new UsersImport, 'users.xlsx');
```

## Appending jobs

When queuing an import an instance of Laravel's `PendingDispatch` is returned. This means you can chain extra jobs that will be added to the end of the queue and only executed if all import jobs are correctly executed.

```php
(new UsersImport)->queue('users.xlsx')->chain([
    new NotifyUserOfCompletedImport(request()->user()),
]);
```

```php
namespace App\Jobs;

use App\User;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\SerializesModels;

class NotifyUserOfCompletedImport implements ShouldQueue
{
    use Queueable, SerializesModels;
    
    public $user;
    
    public function __construct(User $user)
    {
        $this->user = $user;
    }

    public function handle()
    {
        $this->user->notify(new ImportReady());
    }
}
```

## Custom queues

Because `PendingDispatch` is returned, we can also change the queue that should be used.

```php
(new UsersImport)->queue('users.xlsx')->allOnQueue('imports');
```

## Notes
:::warning
You currently cannot queue `xls` imports. PhpSpreadsheet's Xls reader contains some non-utf8 characters, which makes it impossible to queue.
:::
