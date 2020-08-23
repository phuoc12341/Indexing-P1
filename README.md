# Đánh index cho table 1 triệu bản ghi (P1 - tăng 300 lần tốc độ truy vấn Mysql)


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Trong bài này mình sẽ trình bày cách đánh index Unique cho 1 table có 1 triệu bản ghi

- Bài toán: Bây giờ mình có 1 table tests (1 triệu bản ghi): có 5 trường - id, email, name, description, status chẳng hạn. Bây giờ mình cần lấy ra 1 record cụ thể thông qua 1 email cho trước.


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Như mọi người đã biết thì với khi query với bảng có lượng dữ liệu hàng triệu thì thời gian ứng dụng phải query trong DB là rất lâu. Làm cách nào chúng ta lấy ra bản ghi này 1 cách tối ưu thời gian đây.



&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Ok giờ mình sẽ setup, đầu tiên ta cần tạo migration cho table tests: 
```
php artisan make:migration create_tests_table
```


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Mình sẽ setup như sau: Ở đây các bạn chú ý chúng ta sẽ đánh index cho trường email - kiểu index là unique()


```
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreateTestsTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('tests', function (Blueprint $table) {
            $table->id();
            $table->string('email')->unique();
            $table->string('name');
            $table->string('description');
            $table->unsignedTinyInteger('status');
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('tests');
    }
}
```


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Sau đó ta đã tạo xong bảng này, chúng ta chạy lệnh: 

```
php artisan migrate 

```


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Giờ dến việc khó nhằn đây, giờ chúng ta phải seed ra 1 triệu bản ghi trong table này. Không sao mình đã có cách :D cái máy laptop cùi bắp của mình seed tầm 4'30s là được máy các bạn ngon thì sẽ nhanh hơn. Các bạn tạo 1 file Seeder làm như sau: 


```
php artisan make:seeder TestSeeder
```


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Trong này các bạn sẽ tạo 1 triệu bản ghi như sau:


```
<?php

use App\Models\Test;
use Illuminate\Database\Seeder;
use Faker\Generator as Faker;
use Illuminate\Support\Arr;
use Illuminate\Support\Facades\DB;

class TestSeeder extends Seeder
{
    /**
     * Run the database seeds.
     *
     * @return void
     */
    public function run(Faker $faker)
    {
        Test::truncate();

        Test::unguard();

        ini_set('memory_limit', '-1');

        $status = [
            0, 1 , 2, 3
        ];
        
        for ($i=0; $i < 100; $i++ ) {
            $data = collect([]);

            try {
                DB::beginTransaction();

                for ($j=0; $j < 10000; $j++ ) {
                    $row = [
                        'email' => $faker->unique()->email,
                        'name' => $faker->name,
                        'description' => $faker->text,
                        'status' => Arr::random($status),
                    ];
                    
                    $data->push($row);
                }

                Test::insert($data->toArray());

                DB::commit();
            } catch (Exception $exception) {
                dd($exception->errorInfo);
                DB::rollBack();
            }
        }
    }
}
```



&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Giải thích đoạn code trên 1 chút: Mình sẽ chia thành 100 batch mỗi batch insert 10000 bản ghi như vậy là được 1 triệu bản ghi. Các bạn chú ý lini_set('memory_limit', '-1'); nếu không Mysql sẽ báo lỗi do cạn kiệt bộ nhớ vì mình seed quá nhiều dữ liệu nha. 


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Mình viết trực tiếp thế này cho nhanh, nếu không thích các bạn vào sửa file mysql.cnf của mysql cũng được. Nhớ set nó memory_limit=-1 để ko có giới hạn bộ nhớ khi chúng ta insert nha



&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Ok bây giờ chúng ta seed 1 triệu bản ghi thôi. Các bạn chạy lệnh 

```
php artisan db:seed --class=TestSeeder
```



&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Sau đó các bạn ra youtube đá vài bộ phim ngắn hoặc nghe nhạc tí cũng được :v


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Sau khi seed xong. Giờ chúng ta sẽ tạo Controller và Model để test nhanh nhé. Hóng lắm rồi :D


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Các bạn tạo controller + model như sau: 

```
php artisan make:controller TestController --resource --model=Test
```


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Trong hàm TestController chúng ta viết hàm index() đơn giản ngắn gọn như sau:

```
<?php

namespace App\Http\Controllers;

use App\Test;
use Debugbar;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;


class TestController extends Controller
{
    /**
     * Display a listing of the resource.
     *
     * @return \Illuminate\Http\Response
     */
    public function index()
    {
         Debugbar::startMeasure('render','Time for rendering');
         $test = Test::whereEmail('vickie22@hintz.org')->get();
         Debugbar::stopMeasure('render');

         return true;
    }

}
```


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Giải thích: email "vickie22@hintz.org" các bạn lấy 1 email bất kì sau khi seed xong trong bảng test nhé


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Ở đây mình có cài thêm cái LaravelDebugBar để check thời gian chạy câu query


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Tí quên các bạn vào trong file route/web.php: đình nghĩa 1 cái route mình dùng tạm để test như sau

```
Route::get('/tests', 'TestController@index');
```


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Sau đó mình lên trình duyệt gõ: 
```
http://127.0.0.1/tests
```


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Rồi ngắm kết quả trong DebugBar nhé, ở đầy mình ko upload ảnh được các bạn thông cảm =)))


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Và đây là kết quả mình đạt được:



&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Với câu lệnh trên nếu ta không đánh index unique() cho trường email thì để chạy câu query trên máy của mình phải mất trung bình là: 620ms



&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Còn nếu ta có đánh index unique() cho email thì chỉ mất có 2.3ms thôi



&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Ok vậy là xong, chỉ với 1 câu đoạn code đơn giản chúng ta đã query nhanh hơn 300 lần rồi đúng không nào ^_^



&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Hẹn gặp các bạn ở P2 -> mình sẽ trình bày sâu hơn các case khi truy vấn với dữ liệu hàng triệu bản ghi nhé :)))















