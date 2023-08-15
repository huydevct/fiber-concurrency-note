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

## amphp

- amphp v3 xây dựng một Coroutine để thực hiện nhiều chức năng khác nhau và Promise, mã để xử lý không đồng bộ trên ,, Fiber API bằng cách sử dụng event loop. Người dùng amphp v3 không cần trực tiếp sử dụng API thread . Framework sẽ thực hiện việc xử lý đăng ký đó và tạm dừng đối với Fiber, nếu cần. Do đó, trong khuôn khổ tương tự khác, bạn có thể phải tạo và sử dụng như thế nào khác nhau một chút.

- Hàm defer (có thể gọi lại $ callback, mixed ... $ args) đăng ký Fiber sau sẽ được thực thi khi Fiber hiện tại đã bị dừng hoặc kết thúc tạm thời. delay (int $ millisecond), milli giây khi FIber hiện tại được ngắt quãng.

## Flow của Fiber

- Hình dưới đây cho thấy luồng thực thi giữa Main thread và Fiber Luồng thực thi đi qua lại mỗi khi gọi Fiber :: pause () và Fiber-> resume ().

![Flow Fiber!](/d5c1f322-b23e-41ce-906b-10ad3da6ccbb.png "Flow Fiber")

## [Demo Fiber](https://github.com/huydevct/demo-fiber)
