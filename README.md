### Fiber concurrency

- Cho phép tạo ra các hàm fullstack có khả năng phân tách ra
- Đây còn được gọi là coroutine hoặc green thread
- Fiber có thể dừng toàn bộ Execution Stack, vì vậy người gọi hàm không cần thay đổi lệnh gọi.

## Fiber

- Có thể sử dụng Fiber :: pause () để tạm dừng theo ý muốn. Lời gọi đến Fiber :: Suspements () có thể được lồng sâu trong hàm hoặc nó có thể không ở bất kỳ đâu.

- Không giống như các Generator không có Stack, Fiber giữ call stack của riêng nó, vì vậy nó có thể tạm dừng ngay cả khi trong phần call có chứa những hàm đan xen phức tạp. Không giống như các Generator, phải trả về một Generator Instance, các hàm sử dụng Fiber không cần phải thay đổi kiểu trả về.

- Fiber có thể bị tạm dừng với bất kỳ lệnh gọi hàm nào, chẳng hạn như array_map, Iterator foreach hoặc một cuộc gọi từ bên trong một máy ảo PHP. Để tiếp tục một Fiber bị tạm dừng, hãy tiếp tục với Fiber-> resume () hoặc ném một ngoại lệ với Fiber-> throw (). , Giá trị được trả về bằng Fiber :: susu () hoặc một ngoại lệ được ném ra.

- Tạo một object Fiber bằng cách truyền một hàm gọi lại tùy ý làm Fiber mới ($ callback có thể gọi lại). Hàm gọi lại không nhất thiết phải gọi Fiber :: Susan (). Nó có thể nằm sâu bên trong stack lồng nhau hoặc có thể không bao giờ được gọi.

- Object Fiber được tạo được bắt đầu bằng cách truyền Fiber-> start (mixed ... $ args) và các đối số.

- Fiber :: pause () tạm dừng việc thực thi Fiber hiện tại và trả lại quá trình cho trình gọi của Fiber-> start (), Fiber-> resume (), Fiber-> throw (). Bạn có thể coi nó như một cái gì đó tương tự như năng suất trong Generator.

- Fiber đã bị treo có thể được nối lại bằng hai cách sau. --Khởi động lại bằng cách chuyển một giá trị cho Fiber-> resume (). --Exception bằng cách truyền một giá trị cho Fiber-> throw ().

- Từ Fiber đã xử lý, bạn có thể nhận giá trị trả về của hàm gọi lại với Fiber-> getReturn (). Lỗi FiberError được đưa ra khi quá trình thực thi không được hoàn thành hoặc một ngoại lệ xảy ra.

- Fiber :: this () trả về cá thể Fiber hiện đang chạy. Điều này cho phép bạn đưa một tham chiếu Fiber đến một vị trí khác, chẳng hạn như một cuộc gọi lại event loop hoặc một mảng các cá thể Fiber bị treo.

## Reflection Fiber

- Reflection Fiber là phương thức kiểm tra Fiber đang chạy. Có thể tạo Reflection Fiber từ bất kỳ object Fiber nào, cho dù nó được thực thi trước hay hoàn tất. ReflectionFiber rất giống với ReflectionGenerator.

## Fiber stack

- Mỗi thread được gán một stack C riêng biệt và stack VM trên heap. C stack được phân bổ bằng mmap khi có sẵn. Đó là, bộ nhớ vật lý được sử dụng theo yêu cầu trên hầu hết các nền tảng. Mỗi stack Fiber được cấp phát tối đa 8M theo mặc định và có thể được thay đổi với thiết lập ini fiber.stack_size. Lưu ý rằng bộ nhớ này được cấp cho stack C và không liên quan gì đến bộ nhớ được PHP sử dụng.

- Stack VM phân bổ bộ nhớ và CPU theo cách giống như Trình tạo. Vì stack VM có thể được thay đổi động nên chỉ 4K được sử dụng ở trạng thái ban đầu.

- Các thay đổi ngược lại không tương thích Các lớp Fiber / FiberError / FiberExit / ReflectionFiber được thêm vào không gian tên chung. Không có thay đổi không tương thích ngược nào khác.

## Eventloop

- Sau đây là một ví dụ về một event loop rất đơn giản. Thăm dò soclet để nhận dữ liệu, rồi gọi callback khi có thể. event loop này chỉ cho phép khởi động lại Fiber khi có dữ liệu trên Socket, tránh bị chặn đọc.

```
class EventLoop
{
    private string $nextId = 'a';
    private array $deferCallbacks = [];
    private array $read = [];
    private array $streamCallbacks = [];

    public function run(): void
    {
        while (!empty($this->deferCallbacks) || !empty($this->read)) {
            $defers = $this->deferCallbacks;
            $this->deferCallbacks = [];
            foreach ($defers as $id => $defer) {
                $defer();
            }

            $this->select($this->read);
        }
    }

    private function select(array $read): void
    {
        $timeout = empty($this->deferCallbacks) ? null : 0;
        if (!stream_select($read, $write, $except, $timeout, $timeout)) {
            return;
        }

        foreach ($read as $id => $resource) {
            $callback = $this->streamCallbacks[$id];
            unset($this->read[$id], $this->streamCallbacks[$id]);
            $callback($resource);
        }
    }

    public function defer(callable $callback): void
    {
        $id = $this->nextId++;
        $this->deferCallbacks[$id] = $callback;
    }

    public function read($resource, callable $callback): void
    {
        $id = $this->nextId++;
        $this->read[$id] = $resource;
        $this->streamCallbacks[$id] = $callback;
    }
}

[$read, $write] = stream_socket_pair(
    stripos(PHP_OS, 'win') === 0 ? STREAM_PF_INET : STREAM_PF_UNIX,
    STREAM_SOCK_STREAM,
    STREAM_IPPROTO_IP
);

// Đưa stream vào chế độ non proxy
stream_set_blocking($read, false);
stream_set_blocking($write, false);

$loop = new EventLoop;

// Khi có thể đọc stream, lại khởi động việc đọc data bằng 1 cái Fiber khác nữa
$fiber = new Fiber(function () use ($loop, $read): void {
    echo "Waiting for data...\n";

    $fiber = Fiber::this();
    $loop->read($read, fn() => $fiber->resume());
    Fiber::suspend();

    $data = fread($read, 8192);

    echo "Received data: ", $data, "\n";
});

// Chạy Fiber。
$fiber->start();

// Đọc data bằng call back
$loop->defer(fn() => fwrite($write, "Hello, world!"));

// Chạy event loop
$loop->run();
```

## amphp

- amphp v3 xây dựng một Coroutine để thực hiện nhiều chức năng khác nhau và Promise, mã để xử lý không đồng bộ trên ,, Fiber API bằng cách sử dụng event loop. Người dùng amphp v3 không cần trực tiếp sử dụng API thread . Framework sẽ thực hiện việc xử lý đăng ký đó và tạm dừng đối với Fiber, nếu cần. Do đó, trong khuôn khổ tương tự khác, bạn có thể phải tạo và sử dụng như thế nào khác nhau một chút.

- Hàm defer (có thể gọi lại $ callback, mixed ... $ args) đăng ký Fiber sau sẽ được thực thi khi Fiber hiện tại đã bị dừng hoặc kết thúc tạm thời. delay (int $ millisecond), milli giây khi FIber hiện tại được ngắt quãng.

```
use function Amp\defer;
use function Amp\delay;

// defersẽ tạo fiber mới, Fiber đang chạy mà xong thì tự động tạo cái tiếp theo
defer(function (): void {
    delay(1500);
    var_dump(1);
});

defer(function (): void {
    delay(1000);
    var_dump(2);
});

defer(function (): void {
    delay(2000);
    var_dump(3);
});

// Tạm đình chỉ main thread
delay(500);
var_dump(4);
```

- Để hiển thị cách event loop chạy trong khi main thread bị tạm dừng, tôi sẽ đưa ra một ví dụ khác bằng cách sử dụng amphp v3. await (Promise $ promise) sẽ ngừng thực thi cho đến khi đối số Promse được giải quyết. Và async (có thể gọi lại $ callback, mixed ... $ args) trả về một đối tượng Promise.

```
    use function Amp\async;
    use function Amp\await;
    use function Amp\defer;
    use function Amp\delay;

    // Lưu ý rằng giá trị trả về là một int và được thực thi dưới dạng một coroutine không giống như Promises và Generator.
    function asyncTask(int $id): int {
        // CHỗ này ko làm gì cả. Chỉ thay IO không đồng bộ
        delay(1000); // Chỉ dừng  giây
        return $id;
    }

    $running = true;
    defer(function () use (&$running): void {
        // Tôi chỉ muốn cho thấy là không bị block bởi các Fiber khác
        while ($running) {
            delay(100);
            echo ".\n";
        }
    });

    // asyncTask() sẽ trả về int sau 1 giây
    $result = asyncTask(1);
    var_dump($result);

    // Chạy cùng lúc 2 Fiber。Chờ cho đến khi toàn bộ await() mới ngưng Fiber.
    $result = await([  // 1 giây là xong
        async(fn() => asyncTask(2)), // async() tạo FIber và trả về Promise
        async(fn() => asyncTask(3)),
    ]);
    var_dump($result); // Chạy sau 2 giây
    $result = asyncTask(4); // Tốn 1 giây
    var_dump($result);

    // array_map() tốn hai giây. Cái này hiển thị việc gọi ra không đồng bộ
    $result = array_map(fn(int $value) => asyncTask($value), [5, 6]);
    var_dump($result);

    $running = false; // Ngừng cái defer ở trên
```

## Generator

- Fiber cũng có thể bị tạm dừng trong khi gọi PHP VM, vì vậy cũng có thể tạo các trình tạo và trình lặp không đồng bộ. Ví dụ dưới đây sử dụng amphp v3 để tạm ngưng Fiber trong Generator. Khi lặp qua Generator, vòng lặp foreach tạm dừng trong khi chờ giá trị trả về của Generator.

  ```
    use Amp\Delayed;
    use function Amp\await;

    function generator(): Generator {
    yield await(new Delayed(500, 1));
    yield await(new Delayed(1500, 2));
    yield await(new Delayed(1000, 3));
    yield await(new Delayed(2000, 4));
    yield 5;
    yield 6;
    yield 7;
    yield await(new Delayed(2000, 8));
    yield 9;
    yield await(new Delayed(1000, 10));
    }

    // Generator đã chạy lại như bình thường, tùy vào mức độ cần thiết mà có thể phải ngừng loop
    foreach (generator() as $value) {
    printf("Generator yielded %d\n", $value);
    }

    // unpack function cũng vậy
    var_dump(...generator());
  ```

## Flow của Fiber

- Hình dưới đây cho thấy luồng thực thi giữa Main thread và Fiber Luồng thực thi đi qua lại mỗi khi gọi Fiber :: pause () và Fiber-> resume ().

![Flow Fiber!](/d5c1f322-b23e-41ce-906b-10ad3da6ccbb.png "Flow Fiber")

## [Demo Fiber](https://github.com/huydevct/demo-fiber)
