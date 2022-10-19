**[Archived]**

> This guide is explaining how increasing RAM usage to make importing data more efficient.

| Indicator         | Before     | After      |
| ----------------- | ---------- | ---------- |
| Memory Peak Usage | 116.79 MB  | 336.21 MB  |
| Execution Time    | 1h 19m 51s | 18m 34s    |

Before we continue, let's rethink about importing speed. Does it necessary to be fast for the user? Does user often run the import feature? If those two questions answered with "Yes", then let's deep dive into technical configurations. 

Note: This guide is using **maatwebsite/excel** with **Laravel Framework 8.51.0**.

## Modifying PHP & MySQL Configuration File
For this guide, I will use XAMPP v3.2.4, if you use LAMP or WAMP you can locate configuration file according to their guides.

Find **php.ini** file and modify these configuration:
- **max_execution_time** is modified to 0 because importing process will take unknown time to finish.
- **max_input_time** is modified to -1 because we will use HTTP POST to send import file. This is optional if you are using [Laravel Queue](https://laravel.com/docs/8.x/queues) and pass the import process to background process.
- **memory_limit** is modified to 512M to allow higher memory usage for the program, because reading excel file will exhaust lot of memory usage. You can make this number higher or lower depends on your system specification.

After that, find **my.ini** file and modify these configuration:
- **max_allowed_packet** is modified to 16M to increase temporary buffer to be processed. I haven't find the most suitable number yet but 16M still do the job.

**Warning** : This **php.ini** and **my.ini** modification is for testing purpose only, ask your server admin to find suitable number for your program.

## Preparing Import Process

To begin the import process, let's define excel dummy sheet to test. I will use User dummy with **100.000 rows** of records consists of:

Full Name, Gender, Birthday, National ID, Address, City, State, Phone Number, Mobile Number, Height, Weight, Blood Type, Job, Company Name, Company Industry, Employment Status, Vehicle, Vehicle Plate, Company Website, Educational Background

First, let's prepare the user migration:

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreateUsersTable extends Migration
{
    public function up()
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('full_name');
            $table->string('gender');
            $table->string('birthday');
            $table->string('national_id');
            $table->string('address');
            $table->string('city');
            $table->string('state');
            $table->string('phone_number');
            $table->string('mobile_number');
            $table->string('height');
            $table->string('weight');
            $table->string('blood_type');
            $table->string('job');
            $table->string('company_name');
            $table->string('company_industry');
            $table->string('employment_status');
            $table->string('vehicle');
            $table->string('vehicle_plate');
            $table->string('company_website');
            $table->string('educational_background');
            $table->timestamps();
        });
    }

    public function down()
    {
        Schema::dropIfExists('users');
    }
}
```

This migration will generate total of **23 columns** (id + 20 user column + created_at + updated_at).

Then, let's prepare the model:

```php
class User
{
    protected $fillable = [
        'full_name',
        'gender',
        'birthday',
        'national_id',
        'address',
        'city',
        'state',
        'phone_number',
        'mobile_number',
        'height',
        'weight',
        'blood_type',
        'job',
        'company_name',
        'company_industry',
        'employment_status',
        'vehicle',
        'vehicle_plate',
        'company_website',
        'educational_background',
    ];
}
```

After that, let's follow [5 minute quickstart](https://docs.laravel-excel.com/3.1/imports/) and create a **UsersImport.php** file with few modification:

```php
class UsersImport implements ToModel, WithChunkReading, WithBatchInserts, ShouldQueue
{
    public function model(array $row)
    {
        return null;
    }

    public function chunkSize(): int
    {
        return 2848;
    }

    public function batchSize(): int
    {
        return 2848;
    }
}
```

let me explain some things here:
- **batchSize** function will handle how many rows will be inserted per insert batch. It's returning 2848 because MySQL Prepared Statement has maximum parameter value of 65535. Because we will insert 23 value each user, so the maximum insert batch will be 65535/23 = 2849,34. For safety purpose, I will use 2848.
-  **chunkSize** function will handle how many rows will be read each time. Smaller value will use less memory but more time, bigger value will use more memory but less time. Let's make the value same as batch size for now.
- **model** function will retrieve user data and process the data. For testing purpose, let's make it do nothing.

Finally, create simple **UserController** like this:

```php
namespace App\Http\Controllers;

use App\Imports\UsersImport;
use Illuminate\Http\Request;
use Maatwebsite\Excel\Facades\Excel;

class UserController extends Controller
{
    public function import(Request $request){
        $validated = $request->validate([
            'file' => 'required|file|mimetypes:application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
        ]);
    
        $time_start = microtime(true); 
        Excel::import(new UsersImport, $validated['file']);
        $memory_peak = memory_get_peak_usage();
        
        return response()->json([
            'memory_peak' => $memory_peak,
            'execution_time' => (microtime(true) - $time_start)
        ]);
    }
}
```

### Benchmark Test - 2848 Chunk & 2848 Batch (Do Nothing)

Result:

| Indicator         | 1st        | 2nd        | 3rd        | Average     |
| ----------------- | ---------- | ---------- | ---------- | ----------- |
| Memory Peak Usage | 115.58 MB  | 115.58 MB  | 115.58 MB  | 115.58 MB   |
| Execution Time    | 48m 52s    | 51m 15s    | 1h 3m 47s  | 53m 47s     |

The result of memory peak usage is consistently at 115.58 MB, but execution time vary.

### Benchmark Test - 2848 Chunk & 2848 Batch (Full Insert)

Now, let's change the **model** function to insert to database:

```php
class UsersImport implements ToModel, WithHeadingRow, WithChunkReading, WithBatchInserts, ShouldQueue
{
    public function model(array $row)
    {
        return new User([
            'full_name' => $row['full_name'],
            'gender' => $row['gender'],
            'birthday' => $row['birthday'],
            'national_id' => $row['national_id'],
            'address' => $row['address'],
            'city' => $row['city'],
            'state' => $row['state'],
            'phone_number' => $row['phone_number'],
            'mobile_number' => $row['mobile_number'],
            'height' => $row['height'],
            'weight' => $row['weight'],
            'blood_type' => $row['blood_type'],
            'job' => $row['job'],
            'company_name' => $row['company_name'],
            'company_industry' => $row['company_industry'],
            'employment_status' => $row['employment_status'],
            'vehicle' => $row['vehicle'],
            'vehicle_plate' => $row['vehicle_plate'],
            'company_website' => $row['company_website'],
            'educational_background' => $row['educational_background'],
        ]);
    }
}
```

Note that I added WithHeadingRow interface to make row readable.

Result:

| Indicator         | 1st        | 2nd        | 3rd        | Average     |
| ----------------- | ---------- | ---------- | ---------- | ----------- |
| Memory Peak Usage | 116.73 MB  | 116.67 MB  | 116.73 MB  | 116.71 MB   |
| Execution Time    | 1h 7m 34s  | 42m 40s    | 48m 37s    | 52m 58s     |

The result of memory peak usage is stable at 116 MB, but execution time vary.

### Benchmark Test - 2848 Chunk & 2848 Batch (Insert + Conditional Update)

Now, let's change the processing to enable update inside **model** function. We will check `national_id` and compare it to database. If it's already exist, then we will update the record inside database. Modify the code:

```php
public function model(array $row)
{
    $user = User::where('national_id', $row['national_id'])->first();

    if ($user == null) {
        return new User([
            'full_name' => $row['full_name'],
            'gender' => $row['gender'],
            'birthday' => $row['birthday'],
            'national_id' => $row['national_id'],
            'address' => $row['address'],
            'city' => $row['city'],
            'state' => $row['state'],
            'phone_number' => $row['phone_number'],
            'mobile_number' => $row['mobile_number'],
            'height' => $row['height'],
            'weight' => $row['weight'],
            'blood_type' => $row['blood_type'],
            'job' => $row['job'],
            'company_name' => $row['company_name'],
            'company_industry' => $row['company_industry'],
            'employment_status' => $row['employment_status'],
            'vehicle' => $row['vehicle'],
            'vehicle_plate' => $row['vehicle_plate'],
            'company_website' => $row['company_website'],
            'educational_background' => $row['educational_background'],
        ]);
    } else {
        $user->full_name = $row['full_name'];
        $user->gender = $row['gender'];
        $user->birthday = $row['birthday'];
        $user->national_id = $row['national_id'];
        $user->city = $row['city'];
        $user->state = $row['state'];
        $user->phone_number = $row['phone_number'];
        $user->mobile_number = $row['mobile_number'];
        $user->height = $row['height'];
        $user->weight = $row['weight'];
        $user->blood_type = $row['blood_type'];
        $user->job = $row['job'];
        $user->company_name = $row['company_name'];
        $user->company_industry = $row['company_industry'];
        $user->employment_status = $row['employment_status'];
        $user->vehicle = $row['vehicle'];
        $user->vehicle_plate = $row['vehicle_plate'];
        $user->company_website = $row['company_website'];
        $user->educational_background = $row['educational_background'];
        $user->save();
    }

    return null;
}
```

Result:

| Indicator         | 1st        | 2nd        | 3rd        | Average     |
| ----------------- | ---------- | ---------- | ---------- | ----------- |
| Memory Peak Usage | 116.79 MB  | 116.79 MB  | 116.79 MB  | 116.79 MB   |
| Execution Time    | 1h 5m 11s  | 1h 28m 34s | 1h 19m 51s | 1h 17m 53s  |

The result of memory peak usage is stable at 116 MB, but execution time vary.

## Increase Chunk Size

Now let's increase chunk size x10 times greater like this:

```php
public function chunkSize(): int
{
    return 2848 * 10;
}
```

And let's reiterate the benchmark test from "Do Nothing", "Full Insert" to "Insert + Conditional Update".

### Benchmark Test - 28480 Chunk & 2848 Batch (Do Nothing)

Result:

| Indicator         | 1st        | 2nd        | 3rd        | Average     |
| ----------------- | ---------- | ---------- | ---------- | ----------- |
| Memory Peak Usage | 334.87 MB  | 334,87 MB  | 334,87 MB  | 334,87 MB   |
| Execution Time    | 7m 21s     | 5m 16s     | 4m         | 5m 32s      |

The result of memory peak usage is increased yet stable at 334 MB, but execution greatly decrease.

### Benchmark Test - 28480 Chunk & 2848 Batch (Full Insert)

Result:

| Indicator         | 1st        | 2nd        | 3rd        | Average     |
| ----------------- | ---------- | ---------- | ---------- | ----------- |
| Memory Peak Usage | 391.72 MB  | 391.72 MB  | 391.72 MB  | 391.72 MB   |
| Execution Time    | 17mm 38s   | 7m 55s     | 9m 3s      | 11m 33s     |

The result of memory peak usage is stable at 391 MB, but execution vary.

### Benchmark Test - 28480 Chunk & 2848 Batch (Full Insert)

Result:

| Indicator         | 1st        | 2nd        | 3rd        | Average     |
| ----------------- | ---------- | ---------- | ---------- | ----------- |
| Memory Peak Usage | 336.21 MB  | 336.21 MB  | 336.21 MB  | 336.21 MB   |
| Execution Time    | 21mm 57s   | 16m 37s    | 18m 34s    | 19m 3s      |

The result of memory peak usage is stable at 336 MB, but execution vary.

After increasing chunk limit will sacrifice memory for execution time. This optimization is simple because it just increasing chunk limit. But don't forget to try most suitable chunk size because memory limit is 512 MB (php.ini).


## How about CSV instead of XLSX?

CSV is mostly used to move data between programs that aren't ordinarily able to exchange data. Will it be faster than XLSX? Let's try to modify **UsersImport** like this:

```php
class UsersImport implements ToModel, WithHeadingRow, WithChunkReading, WithBatchInserts, ShouldQueue, WithCustomCsvSettings
{
    // Another Code ...
}
```

Add **WithCustomCsvSettings** interface to modify how CSV can be read according to your system or company.

Then add function like this:

```php
public function getCsvSettings(): array
    {
        return [
            'input_encoding' => 'UTF-8',
            'delimiter' => ';',
        ];
    }
```

Adjust your input_encoding and delimiter based on your needs.

Don't forget to modify **Excel::import** in **UserController** like this:

```php
Excel::import(new UsersImport, $validated['file'], null, \Maatwebsite\Excel\Excel::CSV);
```

### Benchmark Test - CSV 28480 Chunk & 2848 Batch (Do Nothing)

Result:

| Indicator         | 1st        | 2nd        | 3rd        | Average     |
| ----------------- | ---------- | ---------- | ---------- | ----------- |
| Memory Peak Usage | 317 MB     | 317.01 MB  | 317.01 MB  | 317,0067 MB |
| Execution Time    | 6m 30s     | 7m 15s     | 6m 45s     | 6m 49s      |

The result of memory peak usage is smaller than XLSX at 317 MB, but execution time vary.

### Benchmark Test - CSV 28480 Chunk & 2848 Batch (Full Insert)

Result:

| Indicator         | 1st        | 2nd        | 3rd        | Average     |
| ----------------- | ---------- | ---------- | ---------- | ----------- |
| Memory Peak Usage | 373.92 MB  | 373.92 MB  | 373.92 MB  | 373.92 MB   |
| Execution Time    | 8m 35s     | 9m 13s     | 8m 33s     | 8m 48s      |

The result of memory peak usage is smaller than XLSX at 373.92 MB, but execution time overall faster than XLSX.

### Benchmark Test - CSV 28480 Chunk & 2848 Batch (Insert + Conditional Update)

Result:

| Indicator         | 1st        | 2nd        | 3rd        | Average     |
| ----------------- | ---------- | ---------- | ---------- | ----------- |
| Memory Peak Usage | 318.35 MB  | 318.35 MB  | 318.35 MB  | 318.35 MB   |
| Execution Time    | 16m 41s    | 28m 48s    | 31m 7s     | 25m 33s     |

The result of memory peak usage is smaller than XLSX at 318.35 MB, but execution time unknowingly slower each tries.

I haven't really researched about CSV due to time and my knowledge about CSV.

If anyone know how to make this faster, you can reach me at linkedin -> [Daniel Nathanael](https://linkedin.com/in/danielnathanael)







